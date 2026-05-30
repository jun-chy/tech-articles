# I2C与SPI通信协议实战对比：原理、代码与选型指南

> 作者：蔡浩宇 | 嵌入式开发笔记

## 引言

I2C和SPI是嵌入式系统中最常用的两种片间通信协议。无论是连接传感器（如温湿度传感器、加速度计）、EEPROM存储器、OLED显示屏，还是与外部ADC/DAC通信，I2C和SPI都是首选方案。本文将从协议原理、STM32 HAL库实现、典型应用场景三个维度进行横向对比，帮助开发者在项目选型时做出最优决策。

---

## 1. I2C协议详解

### 1.1 总线拓扑

```
    VDD
     │
     ├─ [上拉电阻 R1 ≈ 4.7kΩ]
     │
     ├─── SDA ───────────────────────┐
     │                               ├── Device 1 (地址: 0x50)
     │                               ├── Device 2 (地址: 0x68)
     │                               └── Device 3 (地址: 0x76)
     │
     ├─── SCL ───────────────────────┐
     │                               ├── Device 1
     │                               ├── Device 2
     │                               └── Device 3
     │
     └─ [上拉电阻 R2 ≈ 4.7kΩ]
```

**关键特征**：
- **2线制**：SDA（数据）+ SCL（时钟）
- **多主多从**：总线上可有多个主设备和多个从设备
- **地址寻址**：每个从设备有唯一7位地址（扩展模式下10位）
- **开漏输出 + 外部上拉**：实现线与逻辑

### 1.2 通信时序

```
Master发起读操作：

     ┌──┐   ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐ ┌──┐
SCL  ┘  └───┘  └─┘  └─┘  └─┘  └─┘  └─┘  └─┘  └─┘  └─┘  └─
     S   A6 A5 A4 A3 A2 A1 A0  R  ACK D7 D6 D5 D4 D3 D2 D1 D0 N  P

     ┌────────────────────────────────────┐
SDA  S  │  Device Address (7bit) │R/W│ACK│     Data Byte        │ACK│P
     └────────────────────────────────────┘

     S  = START条件 (SDA下降沿, SCL高)
     P  = STOP条件  (SDA上升沿, SCL高)
     A  = ACK/NACK
     R  = 1(读) / 0(写)
```

### 1.3 STM32 I2C HAL实现

```c
I2C_HandleTypeDef hi2c1;

void I2C1_Init(void)
{
    hi2c1.Instance             = I2C1;
    hi2c1.Init.ClockSpeed      = 100000;  // 100kHz (标准模式)
    hi2c1.Init.DutyCycle       = I2C_DUTYCYCLE_2;
    hi2c1.Init.OwnAddress1     = 0;
    hi2c1.Init.AddressingMode  = I2C_ADDRESSINGMODE_7BIT;
    hi2c1.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
    hi2c1.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
    hi2c1.Init.NoStretchMode   = I2C_NOSTRETCH_DISABLE;
    HAL_I2C_Init(&hi2c1);
}

/* 写操作：向AT24C02写入数据 */
HAL_StatusTypeDef EEPROM_Write(uint16_t mem_addr, uint8_t *data, uint16_t len)
{
    uint8_t dev_addr = 0xA0;  // AT24C02地址 (A0=A1=A2=GND)

    /* 发送内存地址 + 数据 */
    uint8_t buf[len + 1];
    buf[0] = (uint8_t)mem_addr;
    memcpy(&buf[1], data, len);

    return HAL_I2C_Master_Transmit(&hi2c1, dev_addr, buf, len + 1, 100);
}

/* 读操作：从AT24C02读取数据 */
HAL_StatusTypeDef EEPROM_Read(uint16_t mem_addr, uint8_t *data, uint16_t len)
{
    uint8_t dev_addr = 0xA0;

    /* 先发送内存地址 */
    HAL_I2C_Master_Transmit(&hi2c1, dev_addr, (uint8_t *)&mem_addr, 1, 100);

    /* 再读取数据 */
    return HAL_I2C_Master_Receive(&hi2c1, dev_addr, data, len, 100);
}
```

---

## 2. SPI协议详解

### 2.1 总线拓扑

```
Master                    Slave 1              Slave 2
  │                         │                    │
  ├── SCK ─────────────────►│ CS1/SCK ───────────►│ CS2/SCK
  ├── MOSI ────────────────►│ MOSI   ────────────►│ MOSI
  ├── MISO ◄────────────────│ MISO   ◄────────────│ MISO
  ├── CS1  ────────────────►│ CS     (独立片选)   │
  └── CS2  ───────────────────────────────────►│ CS
```

**关键特征**：
- **4线制**：SCK（时钟）+ MOSI（主出从入）+ MISO（主入从出）+ CS（片选）
- **全双工**：收发可同时进行
- **独立片选**：每个从设备有独立CS线，硬件编址
- **无上拉电阻**：推挽输出，速率更高

### 2.2 四种通信模式

| 模式 | CPOL | CPHA | 空闲时钟 | 采样边沿 |
|------|------|------|----------|----------|
| SPI_Mode0 | 0 | 0 | 低电平 | 上升沿 |
| SPI_Mode1 | 0 | 1 | 低电平 | 下降沿 |
| SPI_Mode2 | 1 | 0 | 高电平 | 下降沿 |
| SPI_Mode3 | 1 | 1 | 高电平 | 上升沿 |

### 2.3 STM32 SPI HAL实现

```c
SPI_HandleTypeDef hspi1;

void SPI1_Init(void)
{
    hspi1.Instance               = SPI1;
    hspi1.Init.Mode             = SPI_MODE_MASTER;
    hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16;  // APB2/16
    hspi1.Init.Direction        = SPI_DIRECTION_2LINES;        // 全双工
    hspi1.Init.DataSize         = SPI_DATASIZE_8BIT;
    hspi1.Init.CLKPolarity      = SPI_POLARITY_LOW;           // CPOL=0
    hspi1.Init.CLKPhase         = SPI_PHASE_1EDGE;            // CPHA=0 → Mode0
    hspi1.Init.NSS              = SPI_NSS_SOFT;               // 软件片选
    hspi1.Init.FirstBit         = SPI_FIRSTBIT_MSB;
    HAL_SPI_Init(&hspi1);
}

/* 读写W25Q16 Flash */
void W25Q_Read_ID(uint8_t *id_buf)
{
    uint8_t cmd = 0x9F;  // JEDEC ID命令

    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);  // CS拉低
    HAL_SPI_Transmit(&hspi1, &cmd, 1, 100);                // 发送命令
    HAL_SPI_Receive(&hspi1, id_buf, 3, 100);               // 接收3字节ID
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);    // CS拉高
}

/* 读取Flash数据 */
void W25Q_ReadData(uint32_t addr, uint8_t *buf, uint16_t len)
{
    uint8_t cmd[4];
    cmd[0] = 0x03;  // Read命令
    cmd[1] = (addr >> 16) & 0xFF;
    cmd[2] = (addr >> 8) & 0xFF;
    cmd[3] = addr & 0xFF;

    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
    HAL_SPI_Transmit(&hspi1, cmd, 4, 100);
    HAL_SPI_Receive(&hspi1, buf, len, 100);
    HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
}
```

---

## 3. I2C vs SPI 全面对比

```
              ┌────────────┬────────────┐
              │    I2C     │    SPI     │
─────────────┼────────────┼────────────┤
   线数       │    2+1     │    4+N     │
   速度       │ 100k~3.4M │  1M~80M+   │
   双工       │   半双工   │   全双工   │
   寻址       │  软件地址  │  硬件片选  │
   多设备     │  地址空间  │  CS线数量  │
   距离       │  较短     │  较短      │
   引脚开销   │   低       │   高       │
   连线复杂度 │   简单     │   较复杂   │
─────────────┴────────────┴────────────┘
```

### 选型决策树

```
需要全双工通信？
├─ 是 → SPI（如音频Codec、ADC/DAC）
└─ 否 →
    ├─ 设备数量多且引脚有限？
    │   ├─ 是 → I2C（如传感器网络）
    │   └─ 否 → SPI或I2C均可
    └─ 需要高速传输？
        ├─ 是 → SPI（如Flash、LCD）
        └─ 否 → I2C（如EEPROM、RTC）
```

---

## 4. 常见问题与调试技巧

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| I2C总线挂死 | 从设备拉低SDA不释放 | 发送9个SCL时钟脉冲恢复 |
| SPI数据错位 | CPOL/CPHA不匹配 | 查阅从设备数据手册 |
| I2C速度上不去 | 上拉电阻太大或布线寄生电容 | 减小上拉电阻、缩短走线 |
| SPI噪声大 | 无上拉/信号反射 | 加终端电阻、缩短走线 |

---

## 5. 总结

| 场景 | 推荐协议 | 典型设备 |
|------|----------|----------|
| 低速传感器网络 | I2C | BMP280, MPU6050, AT24C02 |
| 高速显示/存储 | SPI | W25Q Flash, SPI LCD, ADS1256 |
| 全双工数据流 | SPI | 音频Codec, SD卡 |
| 引脚受限场景 | I2C | 多传感器共享总线 |

理解I2C和SPI的协议细节和各自的适用场景，是嵌入式硬件驱动开发的基本功。实际项目中两种协议常常混合使用，合理选型可以优化系统资源。

---

*本文首发于个人技术博客，欢迎交流讨论。*
