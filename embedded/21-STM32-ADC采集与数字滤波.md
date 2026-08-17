# STM32 ADC 采集与数字滤波：从逐次逼近到滑动平均与卡尔曼

> 作者：蔡浩宇 | 嵌入式开发笔记

## 引言

几乎所有真实世界的嵌入式项目，最终都要面对一个绕不开的环节：**把模拟世界的物理量（温度、压力、电流、声音）变成数字量，再从中提取出可信的信号**。前者靠 ADC，后者靠数字滤波。

但很多初学者在这里会踩两个坑：一是**以为 ADC 读出来的值就是真实值**，忽略了参考电压、采样保持时间、阻抗匹配带来的误差；二是**迷信"平均值"**，在工频干扰、脉冲噪声、动态跟踪这些不同场景里，一律套用滑动平均，结果要么滤波滞后严重，要么对尖峰噪声无能为力。

本文从 STM32 的逐次逼近型 ADC 原理讲起，覆盖 DMA 连续采集的工程架构，再到四种最实用的数字滤波算法（滑动平均、一阶低通、中值、卡尔曼），帮你建立"从原始采样到可信信号"的完整链路。

---

## 1. ADC 原理：逐次逼近型（SAR）

STM32 内置的 ADC 是**逐次逼近型（SAR, Successive Approximation Register）**。它的核心思想是**二分试探**：

```
假设要量化一个 12 位结果（0~4095），输入电压 V_in：
  1. 先把试探值设成 2048（一半量程），比较 V_in 是否大于它
  2. 若大，则最高位置 1，试探值加到 3072；若小，则置 0，试探值降到 1024
  3. 重复 12 次，逐位确定 —— 这就是"逐次逼近"
```

SAR 型 ADC 的特点：

| 特性 | 说明 | 工程含义 |
|------|------|---------|
| 转换速度 | 中速（STM32F4 可达 2.4 Msps） | 适合大多数传感器 |
| 精度 | 12 位（STM32F4）/ 12/16 位（F7/H7） | 分辨率 = Vref / 2^n |
| 结构 | 一个 DAC + 比较器 + 逐次逼近逻辑 | 需要**采样保持** |
| 关键 | 转换前必须先**采样** | 采样时间不足 → 精度下降 |

**采样保持时间**是最容易忽略的参数。SAR ADC 内部有一个采样电容，切换通道后需要时间充电到与输入电压一致。如果采样时间太短，读到的就是"没充饱"的错误值。

```c
// STM32F4 的采样时间配置（针对不同输入阻抗选择）
// 输入阻抗高（如分压电阻大）→ 需要更长的采样时间
sConfig.SamplingTime = ADC_SAMPLETIME_480CYCLES;  // 高阻源，最长采样时间
// ADC_SAMPLETIME_3CYCLES   —— 低阻源、高速采集
// ADC_SAMPLETIME_480CYCLES —— 高阻源、最高精度
```

**经验法则**：输入阻抗越大（信号源越"软"），采样时间要越长，否则读数会偏小且不稳定。

---

## 2. STM32 ADC 配置：单次 / 连续 / 扫描 / DMA

ADC 有四种工作模式的组合，理解它们才能设计出正确的采集架构：

| 模式 | 行为 | 典型场景 |
|------|------|---------|
| 单次转换 | 触发一次，转换一个通道后停止 | 手动触发、低功耗 |
| 连续转换 | 触发后不断循环转换 | 配合 DMA 持续采集 |
| 扫描模式 | 一次触发按顺序转换多个通道 | 多传感器轮询 |
| 非扫描模式 | 只转换一个通道 | 单传感器 |

**工程最佳实践**：**连续转换 + 扫描 + DMA**，让 CPU 完全从"搬数据"中解放出来。

```c
// 关键配置：ADC + DMA 连续采集（HAL 库）
// 1) 配置 ADC：连续 + 扫描 + 软件触发
hadc.Instance = ADC1;
hadc.Init.ContinuousConvMode = ENABLE;        // 连续转换
hadc.Init.ScanConvMode = ENABLE;              // 扫描多通道
hadc.Init.NbrOfConversion = 2;                // 2 个通道
hadc.Init.ExternalTrigConv = ADC_SOFTWARE_START;
HAL_ADC_Init(&hadc);

// 2) 配置 DMA：循环模式，数据自动搬进数组
hdma.Instance = DMA2_Stream0;
hdma.Init.Direction = DMA_PERIPH_TO_MEMORY;
hdma.Init.Mode = DMA_CIRCULAR;                // 循环模式：满了从头覆盖
hdma.Init.PeriphInc = DMA_PINC_DISABLE;
hdma.Init.MemInc = DMA_MINC_ENABLE;
HAL_DMA_Init(&hdma);
__HAL_LINKDMA(&hadc, DMA_Handle, hdma);

// 3) 启动：DMA 持续把 ADC 结果写入 adc_buf
HAL_ADC_Start_DMA(&hadc, (uint32_t *)adc_buf, BUF_SIZE);
```

**为什么要用 DMA**：ADC 在 2.4 Msps 时每 0.4 µs 就产生一个结果，如果用中断或轮询，CPU 会被完全占用。DMA 让数据"自动流入"内存，CPU 只在缓冲区满时处理一次，效率提升几个数量级。

---

## 3. 采样频率与混叠：滤波的前提

在谈滤波之前，必须先懂**奈奎斯特采样定理**：采样频率 `fs` 必须大于信号最高频率 `f_max` 的 2 倍，否则会发生**混叠（aliasing）**——高频信号"伪装"成低频信号混进数据，滤波也无法挽回。

```
真实信号频率 f_sig = 900 Hz，采样率 fs = 1000 Hz
  → 采样点看起来像 100 Hz 的低频信号（混叠）
  → 无论怎么数字滤波，都无法恢复真实信号
```

**工程对策**：

1. **硬件抗混叠**：ADC 输入前加一个低通 RC 滤波器，把高于 fs/2 的分量先滤掉。
2. **过采样**：用远高于需要的采样率采样，再数字降采样，等效提高分辨率（过采样 + 平均，每 4 倍过采样约增加 1 位有效分辨率）。
3. **采样率选择**：对温度、压力这类慢变量，几十 Hz 就够；对音频、振动，要上 kHz。

---

## 4. 数字滤波一：滑动平均（Moving Average）

滑动平均是最常用的滤波：取最近 N 个样本的平均作为输出。它本质是一个**低通滤波器**。

```c
// 滑动平均：维护一个长度为 N 的环形窗口
#define WIN 8
static uint32_t win_buf[WIN];
static uint8_t  win_idx = 0;
static uint32_t win_sum = 0;

uint32_t moving_average(uint32_t sample)
{
    win_sum -= win_buf[win_idx];        // 减去最旧的值
    win_buf[win_idx] = sample;          // 存入新值
    win_sum += sample;                  // 加上新值
    win_idx = (win_idx + 1) % WIN;      // 环形推进
    return win_sum / WIN;
}
```

**特性**：

| 优点 | 缺点 |
|------|------|
| 实现极简、无乘法 | 窗口越大滞后越大 |
| 对高斯白噪声效果好 | 对脉冲噪声（尖峰）无能为力 |
| 内存占用 O(N) | 截止频率与窗口长度耦合 |

**滞后问题**：N=64 的滑动平均，对一个阶跃信号的响应要"爬"满 64 个点。如果你在测一个快速变化的信号，N 太大会把真实变化也"抹平"了。所以**窗口长度要在"平滑度"和"响应速度"之间权衡**。

---

## 5. 数字滤波二：一阶低通（指数平滑）

滑动平均需要存 N 个历史值，一阶低通滤波器只需一个状态变量，是**内存最优**的滤波：

```c
// 一阶低通滤波（IIR）：y[n] = α·x[n] + (1-α)·y[n-1]
// α 越小，滤波越强、越平滑，但滞后越大
float low_pass(float x, float *prev, float alpha)
{
    float y = alpha * x + (1.0f - alpha) * (*prev);
    *prev = y;
    return y;
}
```

它和硬件 RC 低通滤波是数学上的对应关系：`α = 1 / (1 + 2π·f_c·T)`，其中 `f_c` 是截止频率，`T` 是采样周期。

**如何选 α**：

```c
// 示例：采样周期 T = 1ms，想滤掉 50Hz 工频干扰，截止频率取 10Hz
// α = 2π·f_c·T / (1 + 2π·f_c·T) ≈ 2π·10·0.001 ≈ 0.059
float alpha = 0.06f;   // 近似
```

**对比滑动平均**：一阶低通内存 O(1)、计算 O(1)，且可通过 α 精确控制截止频率；但对脉冲噪声同样不敏感。它最适合**慢变信号 + 内存紧张**的场景，如电池电压、温度。

---

## 6. 数字滤波三：中值滤波（Median Filter）

滑动平均和一阶低通都怕**脉冲噪声**（比如电机打火、继电器切换产生的偶发尖峰）。中值滤波专治这种"椒盐噪声"：取窗口内所有样本的**中位数**，尖峰直接被"无视"。

```c
// 中值滤波：取 N 个样本的中位数
#define MEDIAN_N 5
static uint32_t buf[MEDIAN_N];

uint32_t median_filter(uint32_t sample)
{
    // 移入新样本（环形覆盖）
    memmove(buf, buf + 1, (MEDIAN_N - 1) * sizeof(uint32_t));
    buf[MEDIAN_N - 1] = sample;

    // 求中位数（小窗口直接插入排序，取中间值）
    uint32_t tmp[MEDIAN_N];
    memcpy(tmp, buf, sizeof(buf));
    for (int i = 1; i < MEDIAN_N; ++i) {
        uint32_t key = tmp[i];
        int j = i - 1;
        while (j >= 0 && tmp[j] > key) { tmp[j + 1] = tmp[j]; --j; }
        tmp[j + 1] = key;
    }
    return tmp[MEDIAN_N / 2];
}
```

**特性**：

- **对脉冲噪声近乎免疫**：只要尖峰数量少于窗口的一半，中位数就完全不受影响。
- **不模糊真实阶跃**：与滑动平均不同，中值滤波能保留陡峭的边缘（一个真实的阶跃不会被"抹平"，只会在窗口一半处体现）。
- **缺点**：计算量较大（需排序），且对高斯噪声不如均值滤波有效。

**工程组合拳**：实际项目中常常**中值滤波 + 滑动平均级联**——先用中值滤波吃掉尖峰，再用滑动平均抑制高斯噪声，二者各司其职。

---

## 7. 数字滤波四：卡尔曼滤波（Kalman Filter）

前面三种都是"无模型"滤波，卡尔曼滤波则引入了**系统状态模型**，能在信号快速变化时依然良好跟踪，是动态系统（如姿态解算、目标跟踪）的首选。

一维卡尔曼滤波（标量版本）只有五条公式，非常适合资源有限的 MCU：

```c
// 一维卡尔曼滤波（标量版）
// 状态：真实值 x；观测：ADC 读数 z
// Q：过程噪声方差（越大越"信任"变化，跟踪越快）
// R：测量噪声方差（越大越"不信任"测量，滤波越强）
typedef struct {
    float x;    // 状态估计
    float p;    // 估计协方差
    float q;    // 过程噪声
    float r;    // 测量噪声
} Kalman1D;

void kalman_init(Kalman1D *k, float x0, float q, float r)
{
    k->x = x0; k->p = 1.0f; k->q = q; k->r = r;
}

float kalman_update(Kalman1D *k, float z)
{
    // 预测
    float x_pred = k->x;               // 一维恒值模型
    float p_pred = k->p + k->q;

    // 更新
    float kg = p_pred / (p_pred + k->r);   // 卡尔曼增益
    k->x = x_pred + kg * (z - x_pred);
    k->p = (1.0f - kg) * p_pred;
    return k->x;
}
```

**调参指南**——这是卡尔曼滤波的核心难点：

| 参数 | 调大效果 | 适用 |
|------|---------|------|
| Q（过程噪声） | 更信任测量变化，跟踪更快，但更抖 | 快速变化的信号 |
| R（测量噪声） | 更不信任测量，滤波更平滑，但滞后 | 噪声大的信号 |

```c
Kalman1D k;
// 温度场景：变化慢 → Q 小、R 适中
kalman_init(&k, 25.0f, 0.01f, 1.0f);
// 姿态/角速度场景：变化快 → Q 大、R 小
// kalman_init(&k, 0.0f, 0.1f, 0.5f);

float filtered = kalman_update(&k, (float)adc_read());
```

**小结**：卡尔曼滤波在"信号动态变化 + 噪声大"的场景下明显优于滑动平均和一阶低通，代价是计算量和调参复杂度。嵌入式里用**一维标量版**即可覆盖大多数传感器滤波需求。

---

## 8. 工程实战：DMA + 双缓冲 + 滤波的完整架构

把上面的知识串起来，一个稳健的 ADC 采集系统长这样：

```c
// 双缓冲：DMA 写一个缓冲时，CPU 处理另一个，避免数据被覆盖
#define HALF_SIZE 128
uint16_t adc_buf[2 * HALF_SIZE];       // 两个半区

// ADC 采集 + 滤波流水线
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    // 处理前半区
    process_half(adc_buf, HALF_SIZE);
}

void HAL_ADC_ConvHalfCpltCallback(ADC_HandleTypeDef *hadc)
{
    // 处理后半区
    process_half(adc_buf + HALF_SIZE, HALF_SIZE);
}

void process_half(uint16_t *buf, uint16_t len)
{
    for (int i = 0; i < len; ++i) {
        float raw = (float)buf[i] * VREF / 4096.0f;  // 换算成电压
        float filtered = kalman_update(&k, raw);      // 卡尔曼滤波
        push_to_queue(filtered);                      // 交给上层任务
    }
}
```

**完整链路**：

```
传感器 → 硬件RC低通(抗混叠) → ADC(SAR) → DMA(双缓冲) 
      → 中值滤波(去尖峰) → 卡尔曼/低通(平滑) → 业务逻辑
```

**关键设计点**：

1. **双缓冲**：DMA 循环模式下，用 `ConvCpltCallback` 和 `ConvHalfCpltCallback` 分别在"满"和"半满"时处理，保证处理期间新数据不会覆盖旧数据。
2. **量纲换算**：原始 ADC 值先换算成物理量（电压、温度），滤波在物理量域进行，而不是在原始码值域。
3. **定点优化**：对计算敏感的场景，把浮点卡尔曼改成 Q 格式定点运算，可大幅降低 MCU 负载。

---

## 结语

ADC 采集与数字滤波，是一个"原理简单、细节致命"的领域。回顾本文的完整链路，最值得记住的是这四点：

1. **采样时间、参考电压、输入阻抗**决定了 ADC 读数的"可信度上限"，滤波只能锦上添花，不能无中生有。
2. **混叠是滤波的"不可逆红线"**——采样率不够，再好的滤波器也救不回来。
3. **没有万能滤波器**：滑动平均/一阶低通怕尖峰，中值滤波怕高斯噪声、计算量大，卡尔曼滤波要调参——要根据噪声类型和信号动态性选型。
4. **架构比算法更重要**：DMA + 双缓冲 + 中断回调的采集流水线，才能让滤波算法有稳定、完整的数据可处理。

本系列嵌入式专题已覆盖 GPIO（#01）、UART/DMA（#02）、NVIC 中断（#03）、I2C/SPI（#04）、定时器（#12）、FreeRTOS（#18）与本篇的 ADC 采集。下一篇，我们将继续深入嵌入式，探讨**看门狗与系统可靠性**——如何让设备在死机、跑飞、掉电后依然能"自我救赎"。

---

*本文代码基于 STM32F4 HAL 库，已在 STM32F407 上验证；滤波算法为纯 C，可移植到任意 MCU。*
