# Tech Articles

> 嵌入式开发、数据结构与Qt技术文章合集 | Embedded Systems, Data Structures & Qt Articles

## 嵌入式软件 | Embedded Systems

| # | 文章 | 核心内容 |
|---|------|----------|
| 1 | [STM32 HAL库GPIO配置详解：从寄存器到HAL抽象](embedded/01-STM32-GPIO-HAL详解.md) | GPIO寄存器结构、HAL初始化、按键消抖、BSRR原子操作、AF复用配置 |
| 2 | [嵌入式串口通信：UART协议与DMA传输优化](embedded/02-UART-DMA传输优化.md) | UART帧格式、中断接收、DMA+空闲中断、数据帧协议设计 |
| 3 | [STM32中断系统深度解析：NVIC优先级管理](embedded/03-STM32-NVIC中断优先级管理.md) | 中断向量表、优先级分组、抢占/子优先级、EXTI配置、ISR编写规范 |
| 4 | [I2C与SPI通信协议实战对比](embedded/04-I2C-SPI协议对比.md) | I2C/SPI时序、HAL实现、四线vs两线对比、选型决策树 |

## 数据结构 | Data Structures

| # | 文章 | 核心内容 |
|---|------|----------|
| 5 | [单链表的完整实现：从原理到C语言实战](data-structures/05-单链表完整实现.md) | 链表vs数组、增删改查、反转算法、嵌入式任务链表应用 |
| 6 | [环形缓冲区设计：嵌入式高效数据管理](data-structures/06-环形缓冲区设计.md) | 留空法实现、位掩码优化、DMA+环形缓冲、批量操作 |
| 7 | [哈希表原理与C语言实现](data-structures/07-哈希表原理与C语言实现.md) | DJB2/FNV-1a哈希函数、开放寻址vs链地址、命令解析表实战 |

## Qt开发 | Qt Framework

| # | 文章 | 核心内容 |
|---|------|----------|
| 8 | [Qt信号与槽机制深度解析](qt/08-Qt信号与槽机制深度解析.md) | moc编译器原理、元对象系统、跨线程通信、连接类型 |
| 9 | [使用Qt5开发跨平台串口调试助手](qt/09-Qt5串口调试助手开发.md) | QSerialPort、模块化架构、QSS样式美化、跨平台编译 |
| 10 | [Qt Model/View架构与自定义委托实战](qt/10-Qt-Model-View架构与自定义委托.md) | QAbstractTableModel、自定义委托、进度条渲染、QSortFilterProxyModel |

---

## About

- **Author**: 蔡浩宇 (Cai Haoyu)
- **Focus**: Embedded Systems (STM32), Data Structures & Algorithms, Qt/C++ Desktop Applications
- **Contact**: [GitHub Profile](https://github.com/jun-chy)

## License

All articles are original work. Feel free to reference with attribution.
