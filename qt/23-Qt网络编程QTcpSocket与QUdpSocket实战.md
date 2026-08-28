# Qt 网络编程：从 QTcpSocket 异步模型到 TCP 粘包拆包实战

> 作者：蔡浩宇 | Qt 开发笔记

## 引言

Qt 的网络编程有一个很有欺骗性的特点：**写一个能连上的 TCP 客户端，只需要二十行代码**。

```cpp
QTcpSocket *sock = new QTcpSocket(this);
sock->connectToHost("192.168.1.10", 8080);
connect(sock, &QTcpSocket::readyRead, this, [=] {
    qDebug() << sock->readAll();
});
```

这段代码在局域网里、数据量小、节奏慢的情况下能正常工作。然后你把它用到真实项目里，问题就接踵而至：

- **`readyRead` 触发时数据总是不完整的**——协议帧被拆成了两半，解析必然失败；
- **服务端收到多个连接后就乱套**——不知道哪个 socket 对应哪个客户端；
- **客户端点了"断开"但服务端毫无察觉**——TCP 没有"通知对端我走了"这回事；
- **网络慢的时候界面卡死**——在槽函数里做了耗时处理，阻塞了事件循环；
- **UDP 收不到广播包**——不知道要绑定 `QHostAddress::AnyIPv4`；
- **子线程里创建的 socket 报错**——`QObject: Cannot create children for a parent that is in a different thread`。

这些问题的根源不是 Qt 的 API 难用，而是 **Qt 网络类是一套基于信号槽的异步 I/O 模型**，而大多数人带着"阻塞式 `recv()`"的思维去用它。

本文从 Qt 网络模块的整体架构讲起，重点解决三件事：**异步 I/O 模型的正确心智模型、TCP 粘包拆包的工程级解析方案、以及多线程场景下的线程归属规则**。最后把这些知识落到一个完整的 **TCP/UDP 网络调试助手**项目上——它会与 #09 的串口调试助手形成一套完整的上位机调试工具链。

---

## 1. Qt 网络模块全景

Qt 的网络功能由 **QtNetwork 模块**提供。先建立整体的类图认知：

```
                        ┌─────────────────┐
                        │   QIODevice     │  (所有 I/O 类的基类)
                        │  read/write/    │
                        │  readyRead/     │
                        └────────┬────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
            ┌───────┴────────┐       ┌────────┴────────┐
            │ QAbstractSocket│       │  QUdpSocket     │
            │ (TCP 公共基类) │       │  (无连接)       │
            └───────┬────────┘       └─────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
┌───────┴────────┐      ┌───────┴────────┐
│   QTcpSocket   │      │  QSslSocket    │  (TLS/SSL 加密)
│   (客户端)     │      │  (HTTPS)       │
└────────────────┘      └────────────────┘

┌──────────────────┐    ┌───────────────────────┐
│   QTcpServer     │    │ QNetworkAccessManager │
│  (监听/接入)     │    │ (HTTP/FTP 高层 API)   │
└──────────────────┘    └───────────────────────┘
```

各类的职责与选型：

| 类 | 职责 | 典型场景 |
|----|------|---------|
| `QTcpServer` | 监听端口，接受 TCP 连接 | 服务端 |
| `QTcpSocket` | TCP 客户端/已建立连接 | 客户端、服务端连接对象 |
| `QUdpSocket` | UDP 收发、广播、组播 | 实时数据、设备发现 |
| `QSslSocket` | TLS 加密的 TCP | HTTPS、安全通信 |
| `QNetworkAccessManager` | HTTP 请求高层封装 | REST API、文件下载 |
| `QHostAddress` | IP 地址封装（IPv4/IPv6） | 地址解析、绑定 |
| `QNetworkInterface` | 网卡信息枚举 | 列出本机 IP、获取 MAC |

**重要**：使用 QtNetwork 必须在 `.pro` 文件中加入模块声明：

```pro
QT += core gui network
```

（Qt 6 + CMake 项目则是 `find_package(Qt6 REQUIRED COMPONENTS Network)` + `target_link_libraries(app PRIVATE Qt6::Network)`）

---

## 2. 异步 I/O 模型：Qt 网络编程的心智模型

### 2.1 核心认知：`readyRead` 不保证"一帧数据完整到达"

这是 Qt 网络编程最反直觉、也最重要的一个事实：

> **`readyRead` 信号只表示"内核接收缓冲区里有数据可读"，它和你的应用层协议帧边界没有任何关系。**

TCP 是**字节流协议**，它不保留消息边界。发送方调用三次 `write()`，接收方可能一次 `readAll()` 全部读完，也可能分五次才读完。这取决于 MTU、Nagle 算法、网络拥塞状况、内核缓冲区大小——**完全不可预测**。

```
发送方:  [AA 55 04 01 02 03 04 CRC]  [AA 55 04 05 06 07 08 CRC]
                    ↓ 经过 TCP 字节流传输
接收方可能收到的情况:
  情况1 (粘包): [AA 55 04 01 02 03 04 CRC AA 55 04 05 06 07 08 CRC]  ← 一帧半/两帧
  情况2 (拆包): [AA 55 04 01 02] + [03 04 CRC AA 55 04 05 06 07 08]   ← 半帧 + 后续
  情况3 (混合): [AA 55 04 01 02 03 04 CRC AA 55] + [04 05 06 07 08 CRC]
```

**因此，`readyRead` 槽函数的唯一正确写法是：把数据追加到累积缓冲区，然后循环尝试从缓冲区中解析出完整帧，解析不成的残留数据留在缓冲区等下次。** 这一点在第 4 节详细展开。

### 2.2 信号对照表

`QTcpSocket` 继承自 `QAbstractSocket`，常用信号如下：

| 信号 | 触发时机 | 用途 |
|------|---------|------|
| `connected()` | 连接建立成功 | 启动心跳、更新 UI |
| `disconnected()` | 连接断开（正常或异常） | 清理资源、触发重连 |
| `readyRead()` | 有数据可读 | 读数据并解析 |
| `bytesWritten(qint64)` | 数据已写入底层缓冲 | 大数据发送进度条 |
| `errorOccurred(SocketError)` | 发生错误 | 错误处理（Qt 5.15+，替代废弃的 `error()`） |
| `stateChanged(SocketState)` | 状态变化 | 状态机监控、调试 |
| `hostFound()` | DNS 解析成功 | 诊断连接慢的问题 |

> **Qt 5 → Qt 6 迁移注意**：`QAbstractSocket::error()` 信号在 Qt 5.15 中已废弃，**Qt 6 中已移除**，必须改用 `errorOccurred()`。同时 `QTcpSocket::error()` 这个"获取错误类型"的函数也更名为 `error()` → 无对应（用 `errorOccurred` 的参数）。

### 2.3 状态机

`QTcpSocket` 内部维护一个状态机，理解它对调试连接问题极有帮助：

```
    UnconnectedState  (初始/已断开)
          │  connectToHost()
          ↓
    HostLookupState   (DNS 解析中)
          │
          ↓
    ConnectingState   (TCP 三次握手)
          │
          ↓
    ConnectedState    (已连接, 可收发数据)
          │  disconnectFromHost() / 对端断开
          ↓
    ClosingState      (等待数据发送完毕)
          │
          ↓
    UnconnectedState  ←── 回到起点
```

```cpp
/* 打印状态名, 调试时很有用 */
QString socketStateName(QAbstractSocket::SocketState s)
{
    static const QMap<QAbstractSocket::SocketState, QString> map = {
        {QAbstractSocket::UnconnectedState, "Unconnected"},
        {QAbstractSocket::HostLookupState,  "HostLookup"},
        {QAbstractSocket::ConnectingState,  "Connecting"},
        {QAbstractSocket::ConnectedState,   "Connected"},
        {QAbstractSocket::ClosingState,     "Closing"},
        {QAbstractSocket::BoundState,       "Bound"},
        {QAbstractSocket::ListeningState,   "Listening"}
    };
    return map.value(s, "Unknown");
}
```

---

## 3. TCP 服务端：QTcpServer

### 3.1 最小可用服务端

```cpp
/* ---------------- tcpserver.h ---------------- */
#ifndef TCPSERVER_H
#define TCPSERVER_H

#include <QTcpServer>
#include <QTcpSocket>
#include <QSet>

class TcpServer : public QTcpServer
{
    Q_OBJECT
public:
    explicit TcpServer(QObject *parent = nullptr);
    bool start(quint16 port);
    void stop();
    void broadcast(const QByteArray &data);

signals:
    void logMessage(const QString &msg);
    void clientCountChanged(int count);
    void dataReceived(const QString &peer, const QByteArray &data);

private slots:
    void onNewConnection();
    void onReadyRead();
    void onDisconnected();
    void onErrorOccurred(QAbstractSocket::SocketError err);

protected:
    /* 标准做法：重写 incomingConnection, 用描述符建立 socket */
    void incomingConnection(qintptr socketDescriptor) override;

private:
    QSet<QTcpSocket*> m_clients;
};

#endif
```

```cpp
/* ---------------- tcpserver.cpp ---------------- */
#include "tcpserver.h"

TcpServer::TcpServer(QObject *parent) : QTcpServer(parent) {}

bool TcpServer::start(quint16 port)
{
    /* 监听所有网卡的指定端口 */
    if (!listen(QHostAddress::Any, port)) {
        emit logMessage(QString("监听失败: %1").arg(errorString()));
        return false;
    }
    emit logMessage(QString("已启动监听, 端口 %1").arg(port));
    return true;
}

void TcpServer::stop()
{
    for (QTcpSocket *c : qAsConst(m_clients)) {
        c->disconnectFromHost();
    }
    close();
    m_clients.clear();
}

/* 有新连接到来：Qt 会先创建 socket 并调用此函数 */
void TcpServer::incomingConnection(qintptr socketDescriptor)
{
    QTcpSocket *client = new QTcpSocket(this);

    /* 用描述符接管连接 */
    if (!client->setSocketDescriptor(socketDescriptor)) {
        emit logMessage(QString("接入失败: %1").arg(client->errorString()));
        client->deleteLater();
        return;
    }

    /* 关键: 为每个客户端单独连接信号槽, 并保存映射关系 */
    connect(client, &QTcpSocket::readyRead,
            this, &TcpServer::onReadyRead);
    connect(client, &QTcpSocket::disconnected,
            this, &TcpServer::onDisconnected);
    connect(client, &QTcpSocket::errorOccurred,
            this, &TcpServer::onErrorOccurred);

    m_clients.insert(client);
    emit clientCountChanged(m_clients.size());
    emit logMessage(QString("客户端接入: %1:%2")
                        .arg(client->peerAddress().toString())
                        .arg(client->peerPort()));
}

void TcpServer::onReadyRead()
{
    /* 关键：用 sender() 获取是哪个客户端发来的数据 */
    QTcpSocket *client = qobject_cast<QTcpSocket*>(sender());
    if (!client) return;

    QByteArray data = client->readAll();
    emit dataReceived(QString("%1:%2").arg(client->peerAddress().toString())
                                      .arg(client->peerPort()), data);
}

void TcpServer::onDisconnected()
{
    QTcpSocket *client = qobject_cast<QTcpSocket*>(sender());
    if (!client) return;

    emit logMessage(QString("客户端断开: %1:%2")
                        .arg(client->peerAddress().toString())
                        .arg(client->peerPort()));

    m_clients.remove(client);
    client->deleteLater();          /* 延迟删除, 避免在槽函数中直接 delete */
    emit clientCountChanged(m_clients.size());
}

void TcpServer::onErrorOccurred(QAbstractSocket::SocketError err)
{
    QTcpSocket *client = qobject_cast<QTcpSocket*>(sender());
    if (!client) return;
    emit logMessage(QString("Socket 错误 [%1]: %2")
                        .arg(err).arg(client->errorString()));
}

/* 广播给所有已连接客户端 */
void TcpServer::broadcast(const QByteArray &data)
{
    for (QTcpSocket *c : qAsConst(m_clients)) {
        c->write(data);
        c->flush();
    }
}
```

### 3.2 三个必须注意的细节

**细节一：`sender()` 的多连接分发。** 服务端通常有多个客户端，如果所有 socket 都连到同一个 `readyRead` 槽，就必须用 `sender()` 区分来源。另一种更清晰的做法是用 **lambda 捕获**：

```cpp
connect(client, &QTcpSocket::readyRead, this, [this, client]() {
    QByteArray data = client->readAll();
    processClientData(client, data);       /* client 身份明确, 无需 sender() */
});
```

**细节二：`deleteLater()` 而非 `delete`。** 在 `disconnected` 槽里直接 `delete client` 会导致 Qt 在后续的事件分发中访问已释放对象（崩溃）。`deleteLater()` 会把删除动作推迟到事件循环下一轮，是 Qt 里的标准做法。

**细节三：`QSet<QTcpSocket*>` 需要哈希函数。** Qt 提供了 `qHash(QAbstractSocket*)` 的重载，指针类型可以直接存入 `QSet`。如果编译报错，改用 `QList` + `contains()` 或 `QHash<QTcpSocket*, ClientInfo>`。

---

## 4. TCP 粘包拆包：工程级解决方案

这一节是本文的核心。

### 4.1 三种解决方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| **定长帧** | 每帧固定 N 字节 | 解析最简单 | 浪费带宽，无法变长 | 固定结构数据 |
| **分隔符** | 用 `\r\n` 等特殊字符分隔 | 人类可读，调试方便 | 载荷中需转义 | 文本协议（AT 指令、HTTP 头） |
| **长度前缀** | 帧头携带载荷长度 | **最通用、最可靠** | 需要设计帧结构 | **二进制协议首选** |

嵌入式与工业领域的二进制协议，**几乎全部采用"帧头 + 长度 + 载荷 + 校验"的长度前缀方案**。下面完整实现它。

### 4.2 协议帧定义

```
┌──────┬──────┬──────┬───────┬──────────────┬───────┬──────┐
│ 0xAA │ 0x55 │ LEN  │  CMD  │    PAYLOAD   │ CRC16 │ 0x0D │
│ 帧头1│ 帧头2│ (2B) │ (1B)  │   (LEN 字节) │ (2B)  │ 帧尾 │
└──────┴──────┴──────┴───────┴──────────────┴───────┴──────┘
   1B     1B     2B     1B        变长          2B      1B
                  ↑ 大端(Big-Endian), 便于跨平台
```

- **双字节帧头** `0xAA 0x55`：单字节帧头误判概率高，双字节大幅提升同步可靠性
- **LEN**：payload 长度（不含帧头/CRC/帧尾），大端传输
- **CRC16**：采用 Modbus CRC16（多项式 `0xA001`），覆盖 CMD + PAYLOAD

### 4.3 CRC16 (Modbus) 实现

```cpp
/* CRC16-Modbus: 多项式 0xA001 (即 0x8005 的位反转), 初值 0xFFFF */
static quint16 crc16_modbus(const quint8 *data, int len)
{
    quint16 crc = 0xFFFF;
    for (int i = 0; i < len; ++i) {
        crc ^= data[i];
        for (int j = 0; j < 8; ++j) {
            if (crc & 0x0001) {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }
    return crc;
}
```

### 4.4 帧解析器：累积缓冲 + 状态机

这是**唯一正确**的处理方式。核心是 `m_rxBuffer` 这个成员变量（不是局部变量），未解析完的数据会一直留在里面。

```cpp
/* ---------------- frameparser.h ---------------- */
#ifndef FRAMEPARSER_H
#define FRAMEPARSER_H

#include <QObject>
#include <QByteArray>

struct Frame {
    quint8      cmd;
    QByteArray  payload;
};

class FrameParser : public QObject
{
    Q_OBJECT
public:
    explicit FrameParser(QObject *parent = nullptr);

    /* 喂入新收到的原始字节, 内部自动处理粘包/拆包 */
    void feed(const QByteArray &data);

    /* 打包一帧, 用于发送 */
    static QByteArray pack(quint8 cmd, const QByteArray &payload);

    /* 统计信息, 便于诊断协议健康度 */
    quint32 frameCount() const { return m_frameCount; }
    quint32 crcErrorCount() const { return m_crcErrors; }
    void    reset();

signals:
    void frameReady(const Frame &frame);
    void parseError(const QString &reason);

private:
    void parseLoop();

private:
    QByteArray m_rxBuffer;      /* ⚠️ 必须是成员: 跨多次 readyRead 累积 */
    quint32    m_frameCount = 0;
    quint32    m_crcErrors  = 0;

    static constexpr quint8  FRAME_HEAD1 = 0xAA;
    static constexpr quint8  FRAME_HEAD2 = 0x55;
    static constexpr quint8  FRAME_TAIL  = 0x0D;
    static constexpr int     MIN_FRAME_LEN = 8;   /* 2+2+1+0+2+1 */
    static constexpr int     MAX_PAYLOAD   = 1024;
};

#endif
```

```cpp
/* ---------------- frameparser.cpp ---------------- */
#include "frameparser.h"
#include <QtEndian>

FrameParser::FrameParser(QObject *parent) : QObject(parent)
{
    m_rxBuffer.reserve(4096);       /* 预分配, 减少频繁扩容 */
}

QByteArray FrameParser::pack(quint8 cmd, const QByteArray &payload)
{
    QByteArray frame;
    frame.reserve(payload.size() + 8);

    frame.append(char(FRAME_HEAD1));
    frame.append(char(FRAME_HEAD2));

    /* 长度: 大端 2 字节 */
    quint16 len = quint16(payload.size());
    frame.append(char((len >> 8) & 0xFF));
    frame.append(char(len & 0xFF));

    frame.append(char(cmd));
    frame.append(payload);

    /* CRC 覆盖 CMD + PAYLOAD */
    QByteArray crcSrc;
    crcSrc.append(char(cmd));
    crcSrc.append(payload);
    quint16 crc = crc16_modbus(reinterpret_cast<const quint8*>(crcSrc.constData()),
                               crcSrc.size());
    /* CRC 低字节在前 (Modbus 惯例) */
    frame.append(char(crc & 0xFF));
    frame.append(char((crc >> 8) & 0xFF));

    frame.append(char(FRAME_TAIL));
    return frame;
}

void FrameParser::feed(const QByteArray &data)
{
    m_rxBuffer.append(data);

    /* 缓冲区过大保护: 协议持续错误时避免内存无限增长 */
    if (m_rxBuffer.size() > 64 * 1024) {
        emit parseError("接收缓冲区溢出, 已清空");
        m_rxBuffer.clear();
        return;
    }

    parseLoop();
}

void FrameParser::parseLoop()
{
    /* 循环解析: 一次 readyRead 可能包含多帧(粘包) */
    while (true) {
        /* ① 不足最小帧长, 等下次数据 */
        if (m_rxBuffer.size() < MIN_FRAME_LEN) {
            return;
        }

        /* ② 同步帧头: 丢弃所有非帧头字节(噪声/失步恢复) */
        int headPos = -1;
        for (int i = 0; i <= m_rxBuffer.size() - 2; ++i) {
            if (quint8(m_rxBuffer[i]) == FRAME_HEAD1 &&
                quint8(m_rxBuffer[i + 1]) == FRAME_HEAD2) {
                headPos = i;
                break;
            }
        }

        if (headPos < 0) {
            /* 没有帧头: 只保留最后一个字节(可能是帧头第一字节) */
            m_rxBuffer = m_rxBuffer.right(1);
            return;
        }
        if (headPos > 0) {
            /* 丢弃帧头之前的垃圾数据 */
            m_rxBuffer.remove(0, headPos);
        }

        /* ③ 读取长度字段 */
        quint16 payloadLen = (quint16(quint8(m_rxBuffer[2])) << 8)
                           |  quint16(quint8(m_rxBuffer[3]));

        if (payloadLen > MAX_PAYLOAD) {
            emit parseError(QString("载荷长度非法: %1").arg(payloadLen));
            m_rxBuffer.remove(0, 2);        /* 跳过这个假帧头, 继续同步 */
            continue;
        }

        /* ④ 检查是否收到完整帧 (拆包时这里会 return, 等下次数据) */
        int frameLen = 2 + 2 + 1 + payloadLen + 2 + 1;
        if (m_rxBuffer.size() < frameLen) {
            return;      /* ⚠️ 半帧, 不清理缓冲区, 等待后续数据 */
        }

        /* ⑤ 取出完整帧 */
        QByteArray frame = m_rxBuffer.left(frameLen);
        m_rxBuffer.remove(0, frameLen);

        /* ⑥ 校验帧尾 */
        if (quint8(frame[frameLen - 1]) != FRAME_TAIL) {
            m_crcErrors++;
            emit parseError("帧尾校验失败");
            continue;
        }

        /* ⑦ 校验 CRC */
        quint16 recvCrc =  quint16(quint8(frame[frameLen - 3]))
                        | (quint16(quint8(frame[frameLen - 2])) << 8);
        quint16 calcCrc = crc16_modbus(
                              reinterpret_cast<const quint8*>(frame.constData() + 4),
                              1 + payloadLen);

        if (recvCrc != calcCrc) {
            m_crcErrors++;
            emit parseError(QString("CRC 错误: 收到 0x%1, 计算 0x%2")
                                .arg(recvCrc, 4, 16, QChar('0'))
                                .arg(calcCrc, 4, 16, QChar('0')));
            continue;
        }

        /* ⑧ 帧合法, 投递 */
        Frame f;
        f.cmd     = quint8(frame[4]);
        f.payload = frame.mid(5, payloadLen);
        m_frameCount++;
        emit frameReady(f);
    }
}

void FrameParser::reset()
{
    m_rxBuffer.clear();
    m_frameCount = 0;
    m_crcErrors  = 0;
}
```

**这个解析器解决了所有粘包拆包场景**，因为它遵循三条铁律：

1. **缓冲区是成员变量**，跨 `readyRead` 累积；
2. **解析循环用 `while(true)`**，一次可能解出多帧（粘包）；
3. **数据不足时直接 `return` 且不清理缓冲区**（拆包），下次继续。

### 4.5 在 socket 里接入解析器

```cpp
/* 每个连接配一个独立的 FrameParser */
void ClientHandler::onReadyRead()
{
    m_parser->feed(m_socket->readAll());
}

/* 构造时连接 */
connect(m_parser, &FrameParser::frameReady,
        this, &ClientHandler::onFrameReady);
connect(m_parser, &FrameParser::parseError,
        this, [this](const QString &e) { qWarning() << "协议错误:" << e; });
```

---

## 5. UDP：QUdpSocket

UDP 与 TCP 的关键差别：**UDP 保留消息边界**（一个 `writeDatagram` 对应一个 `readDatagram`，不会粘包），但**不保证可靠、不保证顺序**。

### 5.1 收发数据报

```cpp
/* ---------------- udpsocket.cpp ---------------- */
UdpWorker::UdpWorker(QObject *parent) : QObject(parent)
{
    m_socket = new QUdpSocket(this);

    connect(m_socket, &QUdpSocket::readyRead,
            this, &UdpWorker::onReadyRead);
    connect(m_socket, &QUdpSocket::errorOccurred,
            this, &UdpWorker::onError);
}

/* 绑定本地端口。AnyIPv4 是常见选择 */
bool UdpWorker::bind(quint16 port)
{
    /* ⚠️ 要接收广播包, 必须绑定 AnyIPv4 或 Any, 不能绑具体 IP */
    if (!m_socket->bind(QHostAddress::AnyIPv4, port,
                        QUdpSocket::ShareAddress | QUdpSocket::ReuseAddressHint)) {
        emit logMessage(QString("绑定失败: %1").arg(m_socket->errorString()));
        return false;
    }
    /* 设置接收缓冲区, 高频数据时减少丢包 */
    m_socket->setSocketOption(QAbstractSocket::ReceiveBufferSizeSocketOption, 1024 * 1024);
    return true;
}

void UdpWorker::sendTo(const QHostAddress &addr, quint16 port, const QByteArray &data)
{
    qint64 n = m_socket->writeDatagram(data, addr, port);
    if (n < 0) {
        emit logMessage(QString("发送失败: %1").arg(m_socket->errorString()));
    }
}

void UdpWorker::onReadyRead()
{
    /* 循环读取, 直到没有待处理数据报 */
    while (m_socket->hasPendingDatagrams()) {
        QByteArray buf;
        buf.resize(int(m_socket->pendingDatagramSize()));

        QHostAddress sender;
        quint16 senderPort;
        qint64 n = m_socket->readDatagram(buf.data(), buf.size(), &sender, &senderPort);
        if (n < 0) break;

        buf.resize(int(n));
        emit datagramReceived(sender.toString(), senderPort, buf);
    }
}
```

### 5.2 广播与组播

```cpp
/* 发送广播（需指定广播地址或 QHostAddress::Broadcast = 255.255.255.255） */
void UdpWorker::broadcast(quint16 port, const QByteArray &data)
{
    m_socket->writeDatagram(data, QHostAddress::Broadcast, port);
}

/* 加入组播组（例如 239.0.0.1） */
bool UdpWorker::joinMulticast(const QString &groupAddr)
{
    QHostAddress group(groupAddr);
    if (!m_socket->bind(QHostAddress::AnyIPv4, 0, QUdpSocket::ShareAddress)) {
        return false;
    }
    if (!m_socket->joinMulticastGroup(group)) {
        emit logMessage(QString("加入组播失败: %1").arg(m_socket->errorString()));
        return false;
    }
    return true;
}
```

### 5.3 UDP 的典型用途与注意事项

| 场景 | 说明 |
|------|------|
| **设备发现** | 广播探测包，设备回复自身 IP/MAC，是 IoT 配网的标准做法 |
| **实时数据流** | 音视频、传感器高频上报，容忍少量丢包换取低延迟 |
| **组播** | 一对多分发，节省带宽 |

UDP 的坑：
- **局域网内建议载荷 ≤ 1472 字节**（1500 MTU - 20 IP 头 - 8 UDP 头），超过会 IP 分片，丢一片整包废掉
- **接收高频数据时必须循环读 `hasPendingDatagrams()`**，只读一次会积压
- **Windows 上绑定 `QHostAddress::Any` 时，IPv4 广播包可能无法收到**，用 `AnyIPv4` 更可靠

---

## 6. 线程模型：socket 的线程归属规则

这是 Qt 网络编程里最容易踩的坑，没有之一。

### 6.1 核心规则

> **`QTcpSocket` / `QUdpSocket` / `QTcpServer` 必须在其所属线程中使用。**
> 在一个线程创建的对象，不能在另一个线程调用它的 `read()`/`write()`/`connectToHost()`。

```
   主线程 (GUI)                     工作线程
   ┌──────────────┐                ┌──────────────┐
   │  MainWindow  │                │ NetworkWorker│
   │              │  signal/slot   │              │
   │  (UI 控件)   │◄──────────────►│  QTcpSocket  │  ← socket 属于这个线程
   │              │  (自动排队)    │              │
   └──────────────┘                └──────────────┘
        ✅ 正确                          ✅ 正确
   只更新界面, 不碰 socket            只处理网络, 不碰 UI 控件

   ❌ 错误：在主线程直接调用 worker 线程里 socket 的 write()
   ❌ 错误：在工作线程里直接更新 QLabel / QTableWidget
```

### 6.2 正确范式：moveToThread + 信号槽

```cpp
/* ---------------- 主线程 ---------------- */
void MainWindow::setupNetwork()
{
    m_worker = new NetworkWorker;          /* 不要指定 parent! */
    m_thread = new QThread(this);
    m_worker->moveToThread(m_thread);

    /* 线程结束时清理 worker */
    connect(m_thread, &QThread::finished, m_worker, &QObject::deleteLater);
    /* 通过信号触发 worker 内部的连接动作(在 worker 线程执行) */
    connect(this, &MainWindow::requestConnect,
            m_worker, &NetworkWorker::doConnect);
    /* worker 通过信号把结果传回主线程(自动排队, 线程安全) */
    connect(m_worker, &NetworkWorker::dataReady,
            this, &MainWindow::onDataReady);

    m_thread->start();
}
```

```cpp
/* ---------------- 工作线程对象 ---------------- */
void NetworkWorker::doConnect(const QString &host, quint16 port)
{
    /* 在构造函数里创建 socket 时, 其所属线程是创建时的线程。
     * 更稳妥的做法是延迟到这里创建, 确保 socket 归属工作线程 */
    if (!m_socket) {
        m_socket = new QTcpSocket(this);
        connect(m_socket, &QTcpSocket::readyRead,
                this, &NetworkWorker::onReadyRead);
        connect(m_socket, &QTcpSocket::disconnected,
                this, &NetworkWorker::onDisconnected);
    }
    m_socket->connectToHost(host, port);
}
```

### 6.3 高并发服务端：单线程事件循环 vs 每连接一线程

很多人第一反应是"每个客户端开一个线程"，但在 Qt 里这通常是**过度设计**：

| 方案 | 连接数 | 复杂度 | 推荐度 |
|------|-------|--------|--------|
| **单线程 + 事件循环** | ≤ 数百 | 低 | ⭐⭐⭐⭐⭐ 推荐 |
| **QThreadPool 处理业务** | 数百~数千 | 中 | ⭐⭐⭐⭐ |
| **每连接一线程** | 数十 | 高（锁、同步） | ⭐⭐ |

**理由**：Qt 的 socket 是完全异步的，一个事件循环就能同时处理成百上千个连接，没有阻塞。真正的耗时操作（数据库、复杂计算、图像处理）才需要扔到线程池：

```cpp
void ClientHandler::onFrameReady(const Frame &f)
{
    if (f.cmd == CMD_SIMPLE_QUERY) {
        /* 轻量业务: 直接在事件循环里处理, 立即返回 */
        handleSimpleQuery(f);
        return;
    }

    /* 重业务: 扔给线程池, 处理完再通过信号回到本线程发送结果 */
    auto task = new HeavyTask(f);
    connect(task, &HeavyTask::finished,
            this, &ClientHandler::onHeavyTaskDone);
    QThreadPool::globalInstance()->start(task);
}
```

**绝对不要在 `readyRead` 槽函数里做这两件事**：① 阻塞式网络请求；② 大量 CPU 计算。两者都会阻塞事件循环，导致界面卡死、其他连接饿死。

---

## 7. 心跳与断线重连

### 7.1 TCP 无法感知"静默断线"

TCP 有个反直觉的特性：**如果网线被拔掉，且没有数据交互，`QTcpSocket` 可能很久都不会触发 `disconnected` 信号。** 因为 TCP 的断线检测依赖数据收发或 KeepAlive 探测。

解决方法是应用层心跳：

```cpp
/* ---------------- 客户端心跳 ---------------- */
void TcpClient::setupHeartbeat()
{
    m_heartbeatTimer = new QTimer(this);
    m_heartbeatTimer->setInterval(3000);      /* 3 秒发一次心跳 */
    connect(m_heartbeatTimer, &QTimer::timeout, this, [this]() {
        if (m_socket->state() != QAbstractSocket::ConnectedState) return;

        m_socket->write(FrameParser::pack(CMD_HEARTBEAT, QByteArray()));

        /* 连续 3 次没收到心跳回应 → 判定掉线 */
        if (++m_missedHeartbeat >= 3) {
            qWarning() << "心跳超时, 主动断开";
            m_socket->abort();          /* abort() 立即关闭, 不等数据发完 */
            m_missedHeartbeat = 0;
        }
    });

    connect(m_socket, &QTcpSocket::connected, this, [this]() {
        m_missedHeartbeat = 0;
        m_heartbeatTimer->start();
    });
    connect(m_socket, &QTcpSocket::disconnected, this, [this]() {
        m_heartbeatTimer->stop();
    });
}

/* 收到心跳回应时清零计数 */
void TcpClient::onFrameReady(const Frame &f)
{
    if (f.cmd == CMD_HEARTBEAT_ACK) {
        m_missedHeartbeat = 0;
        return;
    }
    /* ... 其他命令处理 ... */
}
```

### 7.2 自动重连：指数退避

```cpp
void TcpClient::scheduleReconnect()
{
    /* 指数退避: 1s, 2s, 4s, 8s ... 上限 30s, 避免风暴 */
    int delayMs = qMin(1000 * (1 << qMin(m_retryCount, 5)), 30000);
    m_retryCount++;

    qDebug() << QString("将在 %1 ms 后第 %2 次重连").arg(delayMs).arg(m_retryCount);
    QTimer::singleShot(delayMs, this, &TcpClient::doConnect);
}

void TcpClient::onConnected()
{
    m_retryCount = 0;       /* 连接成功后重置退避计数 */
}
```

> **`abort()` vs `disconnectFromHost()`**：
> - `disconnectFromHost()`：优雅关闭，先发完缓冲区数据，再四次挥手
> - `abort()`：立即关闭，丢弃未发送数据
> 掉线/异常场景用 `abort()`，正常退出用 `disconnectFromHost()`。

---

## 8. 实战：TCP/UDP 网络调试助手

把前面的知识组装成一个真实的工具。

### 8.1 架构设计

```
                        ┌─────────────────────────┐
                        │      MainWindow         │
                        │  (QTabWidget 组织模式)   │
                        └───────────┬─────────────┘
                                    │
        ┌───────────────┬───────────┼───────────┬──────────────┐
        ↓               ↓           ↓           ↓              ↓
 ┌──────────┐   ┌────────────┐ ┌─────────┐ ┌─────────┐  ┌──────────┐
 │TcpServer │   │ TcpClient  │ │  Udp    │ │ Protocol│  │ Statistics│
 │  Panel   │   │   Panel    │ │  Panel  │ │  Panel  │  │   Bar     │
 └────┬─────┘   └──────┬─────┘ └────┬────┘ └────┬────┘  └─────┬────┘
      │                │            │           │              │
      └────────────────┴────────────┴───────────┘              │
                       │                                        │
                 ┌─────┴──────────┐                             │
                 │  FrameParser   │──── frameReady ─────────────┘
                 │ (粘包拆包)      │
                 └────────────────┘

 复用模块: HexEdit / TxRxCounter / AutoReplyEngine  (与 #09 串口助手共用)
```

**关键设计**：`FrameParser` 作为独立的可复用组件，被 TCP 服务端、TCP 客户端、UDP 三种模式共用——这正是模块化带来的收益。

### 8.2 十六进制收发

调试工具必须支持十六进制显示与发送（二进制协议调试的刚需）：

```cpp
/* 字节数组 → 十六进制字符串（每字节间加空格） */
QString toHexString(const QByteArray &data)
{
    QString hex;
    hex.reserve(data.size() * 3);
    for (int i = 0; i < data.size(); ++i) {
        hex += QString("%1 ").arg(quint8(data[i]), 2, 16, QChar('0')).toUpper();
    }
    return hex.trimmed();
}

/* 十六进制字符串 → 字节数组（容错：忽略空格、换行、0x 前缀） */
QByteArray fromHexString(const QString &str)
{
    QString cleaned = str;
    cleaned.remove(QRegularExpression("[\\s\\r\\n]"));
    cleaned.remove(QRegularExpression("0[xX]", QRegularExpression::CaseInsensitiveOption));

    /* 奇数长度, 丢弃最后一个非法字符 */
    if (cleaned.length() % 2 != 0) {
        cleaned.chop(1);
    }

    return QByteArray::fromHex(cleaned.toLatin1());
}
```

### 8.3 自动回复引擎

调试服务端时极其有用——模拟设备按规则自动应答：

```cpp
/* 规则: 收到什么 → 回复什么 */
struct AutoReplyRule {
    QByteArray  match;        /* 匹配内容 */
    QByteArray  reply;        /* 回复内容 */
    bool        useHex;       /* 是否十六进制模式 */
    bool        enabled;
};

QByteArray AutoReplyEngine::matchReply(const QByteArray &recv) const
{
    for (const auto &rule : m_rules) {
        if (!rule.enabled || rule.match.isEmpty()) continue;
        if (recv.contains(rule.match)) {
            return rule.useHex ? fromHexString(QString(rule.reply)) : rule.reply;
        }
    }
    return QByteArray();     /* 无匹配规则 */
}
```

### 8.4 流量统计

```cpp
/* 收发字节计数与速率统计 */
void StatisticsBar::updateRate()
{
    static qint64 lastTx = 0, lastRx = 0;
    qint64 tx = m_totalTx, rx = m_totalRx;

    qint64 txRate = tx - lastTx;        /* 每秒字节数 */
    qint64 rxRate = rx - lastRx;
    lastTx = tx; lastRx = rx;

    m_labelTx->setText(QString("TX: %1 B  ↑%2 B/s").arg(tx).arg(txRate));
    m_labelRx->setText(QString("RX: %1 B  ↓%2 B/s").arg(rx).arg(rxRate));
}
```

### 8.5 与串口调试助手的统一

如果你已经读过 #09 的串口调试助手，会发现两者的上层结构几乎一致：

| 层次 | 串口调试助手 | 网络调试助手 |
|------|-------------|-------------|
| 数据收发 | `QSerialPort` | `QTcpSocket` / `QUdpSocket` |
| 协议解析 | `FrameParser` | **`FrameParser`（复用）** |
| 十六进制编解码 | `toHexString/fromHexString` | **复用** |
| 自动回复 | `AutoReplyEngine` | **复用** |
| 流量统计 | `StatisticsBar` | **复用** |

真正的差异只在最底层的 I/O 类。把 `QSerialPort`、`QTcpSocket`、`QUdpSocket` 统一抽象成 `IIODevice` 接口，就能用一套 UI 代码支持串口、TCP 客户端、TCP 服务端、UDP 四种模式——这是这套工具链最终该有的样子。

---

## 9. 常见坑速查

| 现象 | 原因 | 解决 |
|------|------|------|
| 数据总是解析失败 | 未处理粘包拆包 | 用第 4 节的累积缓冲解析器 |
| 服务端多客户端混乱 | 未用 `sender()` 或 lambda 区分 | 每连接独立 handler 对象 |
| 拔网线后检测不到掉线 | TCP 无静默检测 | 应用层心跳 + 超时计数 |
| 子线程创建 socket 报父子线程错误 | 创建时带了 parent | `new QTcpSocket()` 不带 parent，或在目标线程内创建 |
| UDP 收不到广播 | 绑定到了具体 IP | 绑定 `QHostAddress::AnyIPv4` |
| 界面卡顿 | 在 `readyRead` 里做耗时操作 | 事件循环只做解析，重业务扔线程池 |
| Qt 6 编译报 `error()` 信号不存在 | Qt 6 已移除 | 改用 `errorOccurred()` |
| 大量小包发送效率低 | Nagle 算法攒包 | 设 `setSocketOption(QAbstractSocket::LowDelayOption, 1)` |
| `write()` 后对端没收到 | 数据还在 Qt 缓冲 | 调 `flush()`，或连 `bytesWritten` 确认 |
| 关闭窗口时崩溃 | 直接 `delete` 了正在处理事件的 socket | 用 `deleteLater()`，在析构里先 `disconnectFromHost()` |

---

## 结语

Qt 网络编程的难度不在 API，而在**思维模型的转换**：

1. **接受异步**。`readyRead` 不代表"一帧到了"，它只代表"有字节可读"。所有协议解析都必须建立在**成员级累积缓冲区**上，能处理粘包也能处理拆包——这是唯一正确的写法。
2. **协议必须自带边界和校验**。工业级二进制协议的标准形态是"帧头 + 长度 + 命令 + 载荷 + CRC + 帧尾"。有了长度字段才能应对拆包，有了 CRC 才能剔除误码，有了双字节帧头才能从噪声中重新同步。
3. **线程归属是不可违背的规则**。socket 属于哪个线程，就只能在哪个线程操作；跨线程一律走信号槽。绝大多数 Qt 网络崩溃都源于违反这条规则。
4. **永远不要信任网络**。心跳、超时、重连退避、CRC 校验、缓冲区溢出保护——这些不是"锦上添花"，而是让程序在恶劣网络环境下活下去的必需品。

回看整个 Qt 系列：信号与槽（#08）是所有通信的基础，串口助手（#09）与本文的网络助手构成上位机工具链的两翼，Model/View（#10）解决数据展示，多线程（#14）解决并发，QPainter（#16）解决可视化，事件循环（#19）解释了一切异步行为的底层机制。加上本篇，Qt 的核心能力已经连成一片。

下一篇，我们将回到**数据结构与算法**专题，深入**字符串匹配与字典树**：从朴素匹配、KMP 的 next 数组推导，到 Trie 树在 AT 指令解析与命令分发中的实战——它恰好是本文协议解析的"另一半"：本文解决"如何从字节流里取出一帧"，下一篇解决"如何高效理解帧里的内容"。

---

*本文代码基于 Qt 5.15，兼容 Qt 6.x（注意 `errorOccurred()` 信号与 CMake 构建方式的变化）。协议解析器为纯 Qt/C++，可直接用于生产项目。*
