下面是适合你做 **STM32 循迹小车**时需要掌握的 **PWM 知识点**。

# PWM 知识点

## 一、PWM 是什么

PWM 全称是：

```text
Pulse Width Modulation
脉冲宽度调制
```

简单理解：

PWM 就是让一个 GPIO 引脚高速地输出：

```text
高电平、低电平、高电平、低电平……
```

通过改变**高电平所占的比例**，来控制外设的平均功率。

在循迹小车中，PWM 主要用来控制：

```text
左电机速度
右电机速度
舵机角度
蜂鸣器音调
LED 亮度
```

你这个项目中，PWM 用于控制左右电机速度：

```text
左电机 PWM：PA6 → TIM3_CH1
右电机 PWM：PA7 → TIM3_CH2
PWM 频率：10kHz
```

项目硬件说明中也明确了这两个 PWM 引脚配置。

---

# 二、PWM 的三个核心参数

## 1. 周期

PWM 周期就是一轮高低电平变化所用的时间。

例如：

```text
高电平一段时间
低电平一段时间
然后重复
```

一整个高低电平循环，就是一个周期。

---

## 2. 频率

频率表示 PWM 一秒钟重复多少次。

公式：

```text
频率 = 1 / 周期
```

例如：

```text
PWM 频率 = 10kHz
```

意思是：

```text
1 秒钟输出 10000 次 PWM 波形
```

电机控制一般常用：

```text
5kHz ~ 20kHz
```

你这个项目使用 10kHz，比较适合直流电机调速。

---

## 3. 占空比

占空比是 PWM 最重要的概念。

占空比表示：

```text
高电平时间 / 整个周期时间
```

公式：

```text
占空比 = 高电平时间 / 周期时间 × 100%
```

例如：

```text
占空比 0%：一直低电平
占空比 25%：25% 时间高电平，75% 时间低电平
占空比 50%：一半高电平，一半低电平
占空比 100%：一直高电平
```

对电机来说，可以简单理解为：

```text
占空比越大，电机转得越快
占空比越小，电机转得越慢
```

---

# 三、PWM 控制电机的原理

直流电机不是直接用 GPIO 高低电平控制速度，而是用 PWM 控制平均电压。

例如电机电源是 6V：

```text
占空比 100%：平均电压约 6V
占空比 50%：平均电压约 3V
占空比 25%：平均电压约 1.5V
```

所以：

```text
PWM 占空比越大 → 电机平均电压越高 → 电机转速越快
PWM 占空比越小 → 电机平均电压越低 → 电机转速越慢
```

注意：这是简化理解，真实电机还会受到负载、摩擦、电池电压、电机差异影响。

---

# 四、PWM 和 GPIO 的关系

PWM 信号是从 GPIO 引脚输出的，但它不是普通 GPIO 输出。

普通 GPIO 输出：

```text
程序控制高电平
程序控制低电平
```

PWM 输出：

```text
由定时器自动控制高低电平切换
```

所以 PWM 引脚需要配置成：

```text
复用推挽输出 GPIO_Mode_AF_PP
```

例如 STM32F103 标准外设库中：

```c
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOA, &GPIO_InitStructure);
```

其中：

```text
PA6 → TIM3_CH1
PA7 → TIM3_CH2
```

---

# 五、STM32 定时器产生 PWM 的基本原理

STM32 里 PWM 通常由定时器 TIM 产生。

定时器里有几个重要参数：

```text
PSC：预分频器
ARR：自动重装载值
CCR：比较值
```

## 1. PSC：预分频器

PSC 用来降低定时器计数频率。

例如 STM32 主频是 72MHz，定时器频率也可能是 72MHz。

如果不分频，计数太快。

PSC 可以让定时器变慢：

```text
定时器计数频率 = 定时器时钟 / (PSC + 1)
```

---

## 2. ARR：自动重装载值

ARR 决定 PWM 的周期。

定时器从 0 开始计数，数到 ARR 后重新开始。

所以：

```text
ARR 越大，周期越长，频率越低
ARR 越小，周期越短，频率越高
```

PWM 频率公式：

```text
PWM频率 = 定时器时钟 / ((PSC + 1) × (ARR + 1))
```

---

## 3. CCR：比较值

CCR 决定占空比。

在 PWM 模式下，定时器计数值和 CCR 比较。

可以简单理解：

```text
CCR 越大，高电平时间越长，占空比越大
CCR 越小，高电平时间越短，占空比越小
```

占空比公式：

```text
占空比 = CCR / (ARR + 1)
```

例如：

```text
ARR = 999
CCR = 500
占空比 = 500 / 1000 = 50%
```

---

# 六、STM32F103 产生 10kHz PWM 示例

假设定时器时钟是 72MHz，希望 PWM 频率为 10kHz。

可以这样设置：

```text
PSC = 71
ARR = 99
```

计算：

```text
定时器计数频率 = 72MHz / (71 + 1)
               = 1MHz

PWM频率 = 1MHz / (99 + 1)
        = 10kHz
```

如果：

```text
CCR = 50
ARR = 99
```

那么：

```text
占空比 = 50 / 100 = 50%
```

---

# 七、PWM 初始化基本步骤

使用 STM32 标准外设库 SPL 时，大致步骤如下：

```text
1. 开启 GPIOA 时钟
2. 开启 TIM3 时钟
3. 配置 PA6、PA7 为复用推挽输出
4. 配置 TIM3 的 PSC 和 ARR
5. 配置 TIM3_CH1、TIM3_CH2 为 PWM 输出模式
6. 设置 CCR 初始值
7. 使能 TIM3
```

---

# 八、PWM 初始化代码示例

下面是一个适合 STM32F103 的简化示例：

```c
void PWM_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    TIM_OCInitTypeDef TIM_OCInitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);

    // PA6 TIM3_CH1, PA7 TIM3_CH2
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // 72MHz / 72 = 1MHz
    // 1MHz / 100 = 10kHz
    TIM_TimeBaseStructure.TIM_Period = 99;
    TIM_TimeBaseStructure.TIM_Prescaler = 71;
    TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM3, &TIM_TimeBaseStructure);

    // PWM 模式配置
    TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
    TIM_OCInitStructure.TIM_Pulse = 0;
    TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;

    TIM_OC1Init(TIM3, &TIM_OCInitStructure);
    TIM_OC2Init(TIM3, &TIM_OCInitStructure);

    TIM_OC1PreloadConfig(TIM3, TIM_OCPreload_Enable);
    TIM_OC2PreloadConfig(TIM3, TIM_OCPreload_Enable);
    TIM_ARRPreloadConfig(TIM3, ENABLE);

    TIM_Cmd(TIM3, ENABLE);
}
```

---

# 九、设置 PWM 占空比

如果 ARR = 99，那么 CCR 范围是：

```text
0 ~ 99
```

设置左电机 PWM：

```c
TIM_SetCompare1(TIM3, duty);
```

设置右电机 PWM：

```c
TIM_SetCompare2(TIM3, duty);
```

例如：

```c
TIM_SetCompare1(TIM3, 50);  // 左电机 50%
TIM_SetCompare2(TIM3, 50);  // 右电机 50%
```

---

# 十、封装成函数

更推荐写成函数：

```c
void Pwm_SetDuty(uint8_t channel, float duty)
{
    uint16_t compare;

    if(duty < 0.0f) duty = 0.0f;
    if(duty > 1.0f) duty = 1.0f;

    compare = (uint16_t)(duty * 99);

    if(channel == 1)
    {
        TIM_SetCompare1(TIM3, compare);
    }
    else if(channel == 2)
    {
        TIM_SetCompare2(TIM3, compare);
    }
}
```

使用时：

```c
Pwm_SetDuty(1, 0.5f);  // 左电机 50%
Pwm_SetDuty(2, 0.7f);  // 右电机 70%
```

---

# 十一、PWM 在循迹小车中的用法

循迹小车一般不是单独给左右轮设置固定 PWM，而是根据循迹误差动态调整。

基本公式：

```text
左轮 PWM = 基础速度 + 转向修正量
右轮 PWM = 基础速度 - 转向修正量
```

例如：

```text
basePwm = 0.5
turn = 0.2

左轮 = 0.5 + 0.2 = 0.7
右轮 = 0.5 - 0.2 = 0.3
```

这样左右轮速度不同，小车就会转向。

你这个项目中的控制策略就是：

```text
left = basePwm + turn
right = basePwm - turn
```

并且会把速度限制在 `[0, speedMax]` 范围内。

---

# 十二、PWM 和电机方向的关系

PWM 只控制速度，不直接决定方向。

电机方向由 TB6612 的 IN1、IN2 决定：

```text
IN1 = 1, IN2 = 0    正转
IN1 = 0, IN2 = 1    反转
IN1 = 1, IN2 = 1    刹车
IN1 = 0, IN2 = 0    滑行
```

PWM 控制快慢：

```text
PWM 占空比大：转得快
PWM 占空比小：转得慢
```

所以完整电机控制应该是：

```text
方向引脚控制正反转
PWM 引脚控制速度
```

---

# 十三、PWM 常见问题

## 1. 电机不转

可能原因：

```text
PWM 没输出
占空比太低
TB6612 没使能
电机电源没接
电机电源和 STM32 没共地
方向引脚状态错误
```

---

## 2. 电机一直满速

可能原因：

```text
CCR 设置过大
占空比计算错误
PWM 引脚模式配置错
电机驱动模块接线错误
```

---

## 3. 电机速度变化不明显

可能原因：

```text
PWM 频率不合适
占空比范围太小
电池电压不足
电机负载过大
电机本身质量差异明显
```

---

## 4. 小车左右跑偏

可能原因：

```text
左右电机性能不同
左右轮摩擦不同
左右 PWM 实际输出不同
底盘机械安装不对称
```

解决方法：

```text
给左右轮设置不同的基础补偿
例如左轮 basePwm = 0.50
右轮 basePwm = 0.53
```

---

## 5. 小车循迹时疯狂摆动

可能原因：

```text
basePwm 太大
Kp 太大
Kd 太小
传感器误差计算方向反了
PWM 输出限幅不合理
```

---

# 十四、PWM 学习重点总结

做循迹小车，PWM 重点掌握这些：

```text
1. PWM 是高低电平快速切换
2. 占空比决定平均输出功率
3. 占空比越大，电机越快
4. PWM 频率由 PSC 和 ARR 决定
5. 占空比由 CCR 决定
6. PWM 引脚要配置为复用推挽输出
7. STM32F103 可以用 TIM3_CH1/CH2 输出 PWM
8. PA6、PA7 可用于左右电机 PWM
9. PWM 控制速度，GPIO 控制方向
10. 循迹小车通过左右轮 PWM 差速实现转向
```

---

# 十五、最小掌握版本

如果只为了先让小车跑起来，先记住这几句话：

```text
PWM 不是普通高低电平，而是一种高速开关信号。
PWM 的占空比越大，电机转得越快。
STM32 用定时器产生 PWM。
PSC 和 ARR 决定 PWM 频率。
CCR 决定 PWM 占空比。
PA6 和 PA7 可以用 TIM3 输出两路 PWM。
电机方向由 TB6612 的 IN1/IN2 控制，速度由 PWM 控制。
```

一句话总结：

```text
GPIO 管方向，PWM 管速度，左右 PWM 不一样，小车就能转弯。
```