# Tech Articles

> 嵌入式开发、数据结构与Qt技术文章合集 | Embedded Systems, Data Structures & Qt Articles

<p align="center">
  <img src="https://img.shields.io/badge/Articles-19-blue" alt="19 Articles"/>
  <img src="https://img.shields.io/badge/Topics-Embedded%20%7C%20DS%20%7C%20Qt%20%7C%20CV-orange" alt="Topics"/>
  <img src="https://img.shields.io/badge/Language-中文-lightgrey" alt="Chinese"/>
</p>

## 嵌入式软件 | Embedded Systems

| # | 文章 | 核心内容 |
|---|------|----------|
| 1 | [STM32 HAL库GPIO配置详解：从寄存器到HAL抽象](embedded/01-STM32-GPIO-HAL详解.md) | GPIO寄存器结构、HAL初始化、按键消抖、BSRR原子操作、AF复用配置 |
| 2 | [嵌入式串口通信：UART协议与DMA传输优化](embedded/02-UART-DMA传输优化.md) | UART帧格式、中断接收、DMA+空闲中断、数据帧协议设计 |
| 3 | [STM32中断系统深度解析：NVIC优先级管理](embedded/03-STM32-NVIC中断优先级管理.md) | 中断向量表、优先级分组、抢占/子优先级、EXTI配置、ISR编写规范 |
| 4 | [I2C与SPI通信协议实战对比](embedded/04-I2C-SPI协议对比.md) | I2C/SPI时序、HAL实现、四线vs两线对比、选型决策树 |
| 12 | [STM32定时器全解：从基本定时到PWM输出](embedded/12-STM32定时器全解.md) | 定时器分类、时基单元、PWM输出、输入捕获、舵机控制、高级定时器互补输出与死区 |
| 18 | [FreeRTOS任务调度与内存管理：从就绪链表到堆栈检测](embedded/18-FreeRTOS任务调度与内存管理.md) | TCB结构、任务状态机、就绪列表与优先级位图、PendSV上下文切换、heap_1~heap_5、栈溢出检测、STM32多任务实战 |

## 数据结构 | Data Structures

| # | 文章 | 核心内容 |
|---|------|----------|
| 5 | [单链表的完整实现：从原理到C语言实战](data-structures/05-单链表完整实现.md) | 链表vs数组、增删改查、反转算法、嵌入式任务链表应用 |
| 6 | [环形缓冲区设计：嵌入式高效数据管理](data-structures/06-环形缓冲区设计.md) | 留空法实现、位掩码优化、DMA+环形缓冲、批量操作 |
| 7 | [哈希表原理与C语言实现](data-structures/07-哈希表原理与C语言实现.md) | DJB2/FNV-1a哈希函数、开放寻址vs链地址、命令解析表实战 |
| 13 | [二叉搜索树与AVL平衡树：BST到自平衡的工程实现](data-structures/13-二叉搜索树与AVL平衡树.md) | BST基础操作、退化问题、AVL四种旋转、嵌入式配置管理、性能实测对比 |
| 15 | [堆与优先队列：C语言实现到嵌入式任务调度](data-structures/15-堆与优先队列C语言实现与嵌入式调度.md) | 二叉堆数组表示、sift-up/sift-down、堆排序O(1)空间、RTOS任务调度器、d-堆性能对比 |
| 17 | [图算法：从DFS/BFS遍历到嵌入式路径规划](data-structures/17-图算法DFS-BFS与最短路径.md) | 邻接表存储、DFS/BFS遍历、Dijkstra最短路径、A*启发式、AGV路径规划与拓扑排序 |

## Qt开发 | Qt Framework

| # | 文章 | 核心内容 |
|---|------|----------|
| 8 | [Qt信号与槽机制深度解析](qt/08-Qt信号与槽机制深度解析.md) | moc编译器原理、元对象系统、跨线程通信、连接类型 |
| 9 | [使用Qt5开发跨平台串口调试助手](qt/09-Qt5串口调试助手开发.md) | QSerialPort、模块化架构、QSS样式美化、跨平台编译 |
| 10 | [Qt Model/View架构与自定义委托实战](qt/10-Qt-Model-View架构与自定义委托.md) | QAbstractTableModel、自定义委托、进度条渲染、QSortFilterProxyModel |
| 14 | [Qt多线程编程：QThread与线程池实战](qt/14-Qt多线程编程QThread与线程池实战.md) | QThread两种用法、moveToThread范式、QThreadPool+QRunnable、QtConcurrent、串口采集多线程架构 |
| 16 | [Qt自定义绘制：QPainter坐标变换、离屏渲染与实时波形](qt/16-Qt自定义绘制QPainter坐标变换与实时波形.md) | 窗口-视口变换、save/restore栈、QPixmap离屏缓存、QPainterPath、降采样实时波形与性能优化 |
| 19 | [Qt事件循环与事件分发机制：从QEventLoop到自定义事件](qt/19-Qt事件循环与事件分发机制.md) | 事件循环原理、notify三级分发、事件vs信号槽、事件过滤器、自定义事件、sendEvent/postEvent、多线程事件循环、processEvents陷阱 |

## 计算机视觉 | Computer Vision

| # | 文章 | 核心内容 |
|---|------|----------|
| 11 | [OpenCV生态全景：从GitHub热门项目到实战应用](opencv/11-OpenCV生态全景.md) | GitHub项目调研、核心模块速览、算法对比、实战项目设计、学习路线 |

---

## About

- **Author**: 蔡浩宇 (Cai Haoyu)
- **Focus**: Embedded Systems (STM32), Data Structures & Algorithms, Qt/C++ Desktop Applications, Computer Vision (OpenCV)
- **Contact**: [GitHub Profile](https://github.com/jun-chy)

## License

All articles are original work. Feel free to reference with attribution.
