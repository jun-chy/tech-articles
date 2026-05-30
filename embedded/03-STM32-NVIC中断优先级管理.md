# STM32中断系统深度解析：NVIC优先级管理与实战

> 作者：蔡浩宇 | 嵌入式开发笔记

## 引言

中断是嵌入式系统实现"实时响应"的核心机制。没有中断，CPU只能轮询——效率低下且响应延迟不可控。STM32基于ARM Cortex-M内核，提供了灵活的嵌套向量中断控制器（NVIC）。然而，中断优先级配置、抢占与响应的关系、中断服务程序（ISR）的编写规范，这些知识点在实际项目中常常被忽视，导致各种难以排查的Bug。本文将系统性地解析STM32中断系统。

---

## 1. ARM Cortex-M中断架构

### 1.1 中断向量表

STM32的中断向量表位于Flash起始地址（通常为0x08000000），每个中断对应一个4字节（32位）的函数指针：

```
地址           内容                    说明
0x08000000  ┌─────────────────┐
             │  初始栈指针(SP)  │ ← 系统启动后SP自动加载此值
0x08000004  │  Reset_Handler   │ ← 复位向量
0x08000008  │  NMI_Handler     │ ← 不可屏蔽中断
0x0800000C  │  HardFault_Handler│ ← 硬件错误
             │  ...             │
0x0800003C  │  SysTick_Handler │ ← 系统滴答定时器
             │  WWDG_IRQHandler │ ← 窗口看门狗中断（#0）
0x08000040  │  PVD_IRQHandler  │ ← 电源电压检测（#1）
             │  ...             │
             │  EXTI0_IRQHandler│ ← 外部中断线0（#6）
             │  USART1_IRQHandler│ ← 串口1中断（#37）
             │  ...             │
             └─────────────────┘
```

### 1.2 异常优先级层次

```
┌─────────────────────────────────┐
│          异常优先级              │
├─────────────────────────────────┤
│  -3 │ Reset (最高)              │
│  -2 │ NMI                      │
│  -1 │ HardFault                │
│  0  │ Memory Management Fault   │
│  1  │ Bus Fault                │
│  2  │ Usage Fault              │
│  -  │ SVCall (可编程)           │
│  -  │ Debug Monitor            │
│  -  │ PendSV (可编程)           │
│  -  │ SysTick (可编程)          │
│  ≥0 │ 外部中断 IRQ 0~239 (可编程)│ ← 我们主要配置这些
└─────────────────────────────────┘
```

---

## 2. NVIC优先级机制

### 2.1 优先级分组

STM32使用4位（0~15）表示中断优先级，这4位通过**优先级分组**分为**抢占优先级**和**子优先级**：

```c
/* HAL库优先级分组宏 */
HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_2);
```

| 分组 | 抢占位 | 子优先位 | 取值范围 |
|------|--------|----------|----------|
| NVIC_PRIORITYGROUP_0 | 0位 | 4位 | 抢占:0, 子:0~15 |
| NVIC_PRIORITYGROUP_1 | 1位 | 3位 | 抢占:0~1, 子:0~7 |
| NVIC_PRIORITYGROUP_2 | 2位 | 3位 | 抢占:0~3, 子:0~3 |
| NVIC_PRIORITYGROUP_3 | 3位 | 1位 | 抢占:0~7, 子:0~1 |
| NVIC_PRIORITYGROUP_4 | 4位 | 0位 | 抢占:0~15, 子:0 |

### 2.2 抢占优先级 vs 子优先级

```
场景：优先级分组 = NVIC_PRIORITYGROUP_2（2位抢占 + 2位子优先级）

配置：
  串口1中断：抢占=1, 子=0   → 数值 (1<<4)|0 = 0x10
  串口2中断：抢占=1, 子=1   → 数值 (1<<4)|1 = 0x11
  定时器中断：抢占=0, 子=0   → 数值 (0<<4)|0 = 0x00

结果：
  定时器中断（抢占=0）可以打断 串口1 和 串口2 的中断处理
  串口1和串口2（抢占相同）不能互相打断
  串口1和串口2同时挂起时，子优先级高的（串口1, 子=0）先执行
```

> **实用建议**：大多数项目使用 `NVIC_PRIORITYGROUP_4`（全部4位用于抢占优先级），简化优先级管理。子优先级仅在需要精确控制同级中断响应顺序时使用。

### 2.3 优先级配置代码

```c
void Interrupt_Config(void)
{
    /* 设置优先级分组：4位抢占，0位子优先级 */
    HAL_NVIC_SetPriorityGrouping(NVIC_PRIORITYGROUP_4);

    /* 配置外部中断0（按键） - 优先级2 */
    HAL_NVIC_SetPriority(EXTI0_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);

    /* 配置串口1中断 - 优先级1（高于按键） */
    HAL_NVIC_SetPriority(USART1_IRQn, 1, 0);
    HAL_NVIC_EnableIRQ(USART1_IRQn);

    /* 配置SysTick中断 - 优先级0（最高） */
    HAL_NVIC_SetPriority(SysTick_IRQn, 0, 0);
}
```

---

## 3. 外部中断（EXTI）实战

### 3.1 EXTI配置流程

```c
void EXTI_Key_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStruct = {0};

    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_SYSCFG_CLK_ENABLE();

    /* 配置PA0为输入模式（上拉） */
    GPIO_InitStruct.Pin  = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // 下降沿触发
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    /* 配置EXTI线0映射到PA0 */
    HAL_EXTI_D1_EventInputConfig(EXTI_GPIOA, EXTI_GPIO_PIN0,
                                  HAL_EXTI_COMMON_SOF_EVENTOFF);

    /* 配置NVIC */
    HAL_NVIC_SetPriority(EXTI0_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(EXTI0_IRQn);
}

/* EXTI0中断服务函数 */
void EXTI0_IRQHandler(void)
{
    /* 判断是否为EXTI线0中断 */
    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET)
    {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);  // 清除中断标志

        /* 最小化ISR中的操作 */
        g_key_pressed = 1;  // 只设标志，逻辑在主循环处理
    }
}
```

### 3.2 EXTI与GPIO的映射关系

```
每个EXTI线可映射到多个GPIO端口的同号引脚：

EXTI0  ← PA0, PB0, PC0, PD0, ...
EXTI1  ← PA1, PB1, PC1, PD1, ...
  ...
EXTI15 ← PA15, PB15, PC15, PD15, ...

SYSCFG_EXTICR1 寄存器控制 EXTI0~3 的端口映射
SYSCFG_EXTICR2 寄存器控制 EXTI4~7 的端口映射
SYSCFG_EXTICR3 寄存器控制 EXTI8~11 的端口映射
SYSCFG_EXTICR4 寄存器控制 EXTI12~15 的端口映射
```

---

## 4. ISR编写规范

### 4.1 黄金法则

```c
/* ❌ 错误示范：ISR中执行耗时操作 */
void USART1_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart1);
    printf("Received data: %s\r\n", rx_buf);  // ❌ printf很慢！
    HAL_Delay(100);                           // ❌ 绝不能在ISR中延时！
    for (int i = 0; i < 1000; i++) process(); // ❌ 长循环阻塞其他中断！
}

/* ✅ 正确示范：ISR中只做最小操作 */
void USART1_IRQHandler(void)
{
    HAL_UART_IRQHandler(&huart1);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        g_rx_data_ready = 1;     // ✅ 只设标志位
        g_rx_data_len = rx_len; // ✅ 保存长度
        HAL_UART_Receive_IT(&huart1, rx_buf, 1); // ✅ 重新开启接收
    }
}

/* 主循环中处理实际业务逻辑 */
while (1)
{
    if (g_rx_data_ready)
    {
        g_rx_data_ready = 0;
        Process_Data(rx_buf, g_rx_data_len);  // 在非中断上下文处理
    }
}
```

### 4.2 ISR编写要点清单

| 规则 | 说明 |
|------|------|
| 执行时间 < 10us | 越短越好，避免阻塞低优先级中断 |
| 不调用阻塞API | 禁止HAL_Delay、osDelay等 |
| 变量声明为volatile | 防止编译器优化掉共享变量访问 |
| 不可重入函数 | ISR中避免调用printf、malloc等 |
| 注意中断安全 | 共享资源需关中断保护临界区 |

---

## 5. FreeRTOS与中断的协作

在引入RTOS后，中断与任务之间的通信变得尤为重要：

```c
/* ISR中唤醒任务（二值信号量） */
void EXTI0_IRQHandler(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    if (__HAL_GPIO_EXTI_GET_IT(GPIO_PIN_0) != RESET)
    {
        __HAL_GPIO_EXTI_CLEAR_IT(GPIO_PIN_0);

        /* 释放信号量，唤醒处理任务 */
        xSemaphoreGiveFromISR(xKeySemaphore, &xHigherPriorityTaskWoken);
    }

    /* 如果有更高优先级任务就绪，触发任务切换 */
    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}

/* 按键处理任务 */
void KeyTask(void *pvParameters)
{
    for (;;)
    {
        if (xSemaphoreTake(xKeySemaphore, portMAX_DELAY) == pdTRUE)
        {
            // 在任务上下文中安全处理按键
            HandleKeyPress();
        }
    }
}
```

---

## 6. 总结

| 知识点 | 核心内容 |
|--------|----------|
| 向量表 | 中断号与ISR函数的映射表 |
| 优先级分组 | 4位优先级的拆分方式，决定抢占/子优先级位数 |
| 抢占机制 | 高抢占优先级中断可打断低抢占优先级中断 |
| ISR规范 | 短小精悍、无阻塞、volatile变量 |
| 中断-任务通信 | RTOS中通过信号量/队列实现安全通信 |

中断系统是嵌入式实时性的基石，掌握NVIC优先级管理是写出可靠嵌入式软件的必要条件。

---

*本文首发于个人技术博客，欢迎交流讨论。*
