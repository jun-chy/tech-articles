# 嵌入式串口通信：UART协议与DMA传输优化

> 作者：蔡浩宇 | 嵌入式开发笔记

## 引言

UART（Universal Asynchronous Receiver/Transmitter）是嵌入式系统中最常用的通信接口之一。从调试打印到模块通信，UART无处不在。然而，许多开发者只停留在"发个字符串、收几个字节"的层面，对底层协议细节、DMA高效传输、中断接收机制等缺乏深入理解。本文将系统性地梳理UART通信的核心知识点。

---

## 1. UART协议基础

### 1.1 帧格式

```
空闲状态(高电平)
    │
    ▼
┌───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ S │D0 │D1 │D2 │D3 │D4 │D5 │D6 │D7 │ P │ 1 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┘
  │       8位数据位(先LSB)         │奇偶│停止│
  起始位                          校验│ 位 │

空闲(高电平)
```

### 1.2 关键参数

| 参数 | 说明 | 常用值 |
|------|------|--------|
| 波特率 | 每秒传输的位数 | 9600, 115200 |
| 数据位 | 每帧有效数据长度 | 8位 |
| 停止位 | 帧结束标志 | 1位 |
| 校验位 | 错误检测 | None/Even/Odd |
| 流控 | 数据流控制 | None/RTS-CTS |

> **波特率与比特率的区别**：在UART中，每个数据位都携带信息（无编码开销），因此波特率等于比特率。但在RS232等带调制的技术中，两者不同。

---

## 2. STM32 UART初始化与收发

### 2.1 基础收发

```c
UART_HandleTypeDef huart1;

void UART1_Init(void)
{
    huart1.Instance          = USART1;
    huart1.Init.BaudRate     = 115200;
    huart1.Init.WordLength   = UART_WORDLENGTH_8B;
    huart1.Init.StopBits    = UART_STOPBITS_1;
    huart1.Init.Parity      = UART_PARITY_NONE;
    huart1.Init.Mode        = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl   = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    HAL_UART_Init(&huart1);
}

/* 轮询方式发送 */
HAL_UART_Transmit(&huart1, (uint8_t *)"Hello\r\n", 7, HAL_MAX_DELAY);

/* 轮询方式接收 */
uint8_t rx_buf;
HAL_UART_Receive(&huart1, &rx_buf, 1, HAL_MAX_DELAY);
```

### 2.2 重定向printf

```c
#include <stdio.h>

/* 重定向fputc实现printf输出 */
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}

/* 使用示例 */
printf("System initialized, clock: %lu Hz\r\n", HAL_RCC_GetSysClockFreq());
```

---

## 3. 中断方式收发

### 3.1 中断接收的局限性

```c
uint8_t rx_byte;

/* 开启中断接收（每次只接收1字节） */
HAL_UART_Receive_IT(&huart1, &rx_byte, 1);

/* 中断回调函数 - 在stm32f4xx_it.c或main.c中 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        /* 处理接收到的字节 */
        // process_byte(rx_byte);

        /* 重新开启下一次接收 */
        HAL_UART_Receive_IT(&huart1, &rx_byte, 1);
    }
}
```

> **问题**：每接收一个字节就触发一次中断，在115200波特率下，每86微秒就产生一次中断。对于Cortex-M4来说，中断处理的开销不可忽视。

---

## 4. DMA传输——高性能方案

### 4.1 DMA工作原理

```
┌──────────┐    DMA请求     ┌──────────────┐
│  UART DR │ ◄──────────── │   DMA控制器  │
│(数据寄存器)│ ────────────► │              │
└──────────┘   DMA传输       │  ┌────────┐  │
                              │  │外设寄存│  │
┌──────────┐   DMA传输        │  │器地址  │  │
│  内存    │ ◄────────────►  │  └────────┘  │
│(接收缓冲) │                │  ┌────────┐  │
└──────────┘                │  │存储器  │  │
                            │  │地址    │  │
                            │  └────────┘  │
                            └──────────────┘
```

### 4.2 DMA发送配置

```c
#define TX_BUF_SIZE  256
uint8_t tx_buf[TX_BUF_SIZE];

void UART1_DMA_Init(void)
{
    /* 使能DMA时钟 */
    __HAL_RCC_DMA2_CLK_ENABLE();

    /* 配置DMA发送通道 (USART1_TX → DMA2 Stream7 Channel4) */
    hdma_usart1_tx.Instance                 = DMA2_Stream7;
    hdma_usart1_tx.Init.Channel             = DMA_CHANNEL_4;
    hdma_usart1_tx.Init.Direction          = DMA_MEMORY_TO_PERIPH;
    hdma_usart1_tx.Init.PeriphInc           = DMA_PINC_DISABLE;
    hdma_usart1_tx.Init.MemInc              = DMA_MINC_ENABLE;
    hdma_usart1_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    hdma_usart1_tx.Init.MemDataAlignment    = DMA_MDATAALIGN_BYTE;
    hdma_usart1_tx.Init.Mode                = DMA_NORMAL;
    hdma_usart1_tx.Init.Priority            = DMA_PRIORITY_MEDIUM;
    hdma_usart1_tx.Init.FIFOMode            = DMA_FIFOMODE_DISABLE;
    HAL_DMA_Init(&hdma_usart1_tx);

    /* 关联DMA到UART */
    __HAL_LINKDMA(&huart1, hdmatx, hdma_usart1_tx);
    HAL_UART_DMA_TX(&huart1);
}

/* DMA发送（非阻塞） */
void DMA_SendString(const char *str)
{
    uint16_t len = strlen(str);
    memcpy(tx_buf, str, len);
    HAL_UART_Transmit_DMA(&huart1, tx_buf, len);
}

/* DMA发送完成回调 */
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        /* 发送完成，可以发送下一帧或切换状态 */
        tx_complete = 1;
    }
}
```

### 4.3 DMA接收 + 空闲中断（推荐方案）

这是嵌入式开发中最实用的串口接收方案：

```c
#define RX_BUF_SIZE  256
uint8_t rx_buf[RX_BUF_SIZE];

void UART1_DMA_Rx_Init(void)
{
    /* 配置DMA接收通道 (USART1_RX → DMA2 Stream5 Channel4) */
    hdma_usart1_rx.Instance                 = DMA2_Stream5;
    hdma_usart1_rx.Init.Channel             = DMA_CHANNEL_4;
    hdma_usart1_rx.Init.Direction          = DMA_PERIPH_TO_MEMORY;
    hdma_usart1_rx.Init.PeriphInc           = DMA_PINC_DISABLE;
    hdma_usart1_rx.Init.MemInc              = DMA_MINC_ENABLE;
    hdma_usart1_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    hdma_usart1_rx.Init.MemDataAlignment    = DMA_MDATAALIGN_BYTE;
    hdma_usart1_rx.Init.Mode                = DMA_CIRCULAR;  // 循环模式
    hdma_usart1_rx.Init.Priority            = DMA_PRIORITY_HIGH;
    HAL_DMA_Init(&hdma_usart1_rx);
    __HAL_LINKDMA(&huart1, hdmarx, hdma_usart1_rx);
    HAL_UART_DMA_RX(&huart1);

    /* 启动DMA接收 */
    HAL_UART_Receive_DMA(&huart1, rx_buf, RX_BUF_SIZE);

    /* 使能空闲中断 */
    __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);
}

/* 空闲中断处理（在stm32f4xx_it.c的USART1_IRQHandler中） */
void USART1_IRQHandler(void)
{
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE))
    {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);

        /* 计算已接收数据长度 */
        uint16_t rx_len = RX_BUF_SIZE - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);

        /* 处理接收到的数据 */
        Process_RxData(rx_buf, rx_len);

        /* 循环模式下，DMA自动继续，无需重新启动 */
    }
    HAL_UART_IRQHandler(&huart1);
}
```

---

## 5. 数据帧协议设计

在实际项目中，原始字节流通常需要封装为结构化帧：

```
┌──────┬──────┬──────┬───────────┬──────┬──────┐
│ 0xAA │ 0x55 │ LEN  │  DATA...  │ CRC  │ 0x0D │
│帧头1 │帧头2 │长度  │ 有效载荷  │校验  │帧尾  │
└──────┴──────┴──────┴───────────┴──────┴──────┘
```

```c
typedef struct {
    uint8_t  header1;    // 0xAA
    uint8_t  header2;    // 0x55
    uint8_t  length;     // 数据长度
    uint8_t  data[64];   // 有效载荷
    uint8_t  crc;        // CRC8校验
    uint8_t  tail;       // 0x0D
} UART_Frame_t;

/* CRC8计算 */
uint8_t CRC8_Calculate(const uint8_t *data, uint16_t len)
{
    uint8_t crc = 0xFF;
    for (uint16_t i = 0; i < len; i++)
    {
        crc ^= data[i];
        for (uint8_t j = 0; j < 8; j++)
        {
            crc = (crc & 0x80) ? (crc << 1) ^ 0x07 : crc << 1;
        }
    }
    return crc;
}
```

---

## 6. 性能对比

| 方式 | CPU占用 | 延迟 | 适用场景 |
|------|---------|------|----------|
| 轮询 | 最高 | 最低 | 调试、初始化 |
| 中断(逐字节) | 中等 | 低 | 低速通信 |
| DMA+空闲中断 | 最低 | 中 | 高速批量数据 |

---

## 7. 总结

UART通信看似简单，但要做到高效可靠，需要理解DMA传输、空闲中断、帧协议设计等进阶知识。DMA+空闲中断方案是当前嵌入式串口接收的"最佳实践"，值得深入掌握。

---

*本文首发于个人技术博客，欢迎交流讨论。*
