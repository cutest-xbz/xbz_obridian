下面是适合你做 **STM32 循迹小车**时需要掌握的 GPIO 知识点。

# GPIO 知识点

## 一、GPIO 是什么

GPIO 全称是：

General Purpose Input Output  
通用输入输出引脚

简单理解：

GPIO 就是单片机上的普通引脚，可以通过程序控制它：

```text
作为输入：读取外部信号
作为输出：控制外部设备
```

在循迹小车中，GPIO 主要用来：

```text
读取红外循迹传感器
控制电机驱动方向
控制 LED、蜂鸣器等外设
```

例如你这个项目中，8 路红外传感器接在 `PD0~PD7`，电机方向控制使用 `PD9/PD11`、`PD12/PD10`。

---

# 二、GPIO 的输入功能

## 1. 输入模式的作用

输入模式就是让 STM32 读取外部电平。

例如红外传感器输出：

```text
高电平：1
低电平：0
```

单片机通过 GPIO 读取这个电平，就能判断当前传感器是否检测到黑线。

---

## 2. 常见输入模式

### 1. 浮空输入

```text
GPIO_Mode_IN_FLOATING
```

特点：

```text
引脚没有默认电平
容易受干扰
不推荐用于普通按键或传感器
```

如果外部没有稳定信号，可能会一会儿读到 0，一会儿读到 1。

---

### 2. 上拉输入

```text
GPIO_Mode_IPU
```

特点：

```text
默认读到高电平 1
外部拉低时读到 0
适合按键、红外数字输出
```

循迹小车的红外传感器一般可以使用上拉输入。

---

### 3. 下拉输入

```text
GPIO_Mode_IPD
```

特点：

```text
默认读到低电平 0
外部拉高时读到 1
```

---

### 4. 模拟输入

```text
GPIO_Mode_AIN
```

用于 ADC 模拟采样，例如：

```text
电位器
光敏电阻
模拟传感器
```

如果红外循迹模块输出的是数字信号，一般不用模拟输入。

---

# 三、GPIO 的输出功能

## 1. 输出模式的作用

输出模式就是让 STM32 主动输出高低电平。

例如：

```text
输出高电平：点亮 LED / 控制电机方向
输出低电平：熄灭 LED / 改变电机方向
```

---

## 2. 常见输出模式

### 1. 推挽输出

```text
GPIO_Mode_Out_PP
```

特点：

```text
可以输出高电平
可以输出低电平
驱动能力较强
最常用
```

适合控制：

```text
LED
蜂鸣器
电机驱动模块方向引脚
```

TB6612 的方向控制引脚一般就用推挽输出。

---

### 2. 开漏输出

```text
GPIO_Mode_Out_OD
```

特点：

```text
只能主动输出低电平
不能主动输出高电平
需要外部上拉电阻
```

常用于：

```text
I2C 总线
多个设备共用一条线
```

普通循迹小车初学阶段用得较少。

---

### 3. 复用推挽输出

```text
GPIO_Mode_AF_PP
```

特点：

```text
引脚不再作为普通 GPIO 使用
而是交给外设控制
```

比如 PWM 输出时，GPIO 要设置成复用推挽输出。

你的项目中：

```text
PA6 → TIM3_CH1 → 左电机 PWM
PA7 → TIM3_CH2 → 右电机 PWM
```

所以 PA6、PA7 要设置为复用推挽输出。

---

# 四、GPIO 高低电平

## 1. 高电平

在 STM32F103 中，GPIO 高电平一般接近：

```text
3.3V
```

程序中通常表示为：

```text
1
SET
Bit_SET
```

---

## 2. 低电平

低电平接近：

```text
0V
```

程序中通常表示为：

```text
0
RESET
Bit_RESET
```

---

# 五、GPIO 常用函数

以 STM32 标准外设库 SPL 为例。

## 1. 开启 GPIO 时钟

使用 GPIO 前必须先开时钟。

例如使用 GPIOA：

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
```

使用 GPIOD：

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, ENABLE);
```

如果没有开时钟，GPIO 初始化和读写都可能无效。

---

## 2. 初始化 GPIO

基本格式：

```c
GPIO_InitTypeDef GPIO_InitStructure;

GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

GPIO_Init(GPIOD, &GPIO_InitStructure);
```

---

## 3. 读取输入电平

读取某一个引脚：

```c
GPIO_ReadInputDataBit(GPIOD, GPIO_Pin_0);
```

返回值：

```text
Bit_SET    高电平
Bit_RESET  低电平
```

例如：

```c
if(GPIO_ReadInputDataBit(GPIOD, GPIO_Pin_0) == Bit_RESET)
{
    // 检测到低电平
}
```

---

## 4. 输出高电平

```c
GPIO_SetBits(GPIOD, GPIO_Pin_9);
```

---

## 5. 输出低电平

```c
GPIO_ResetBits(GPIOD, GPIO_Pin_9);
```

---

## 6. 写一个引脚状态

```c
GPIO_WriteBit(GPIOD, GPIO_Pin_9, Bit_SET);
GPIO_WriteBit(GPIOD, GPIO_Pin_9, Bit_RESET);
```

---

# 六、GPIO 在循迹小车中的应用

## 1. 读取 8 路红外传感器

假设 8 路红外接在：

```text
PD0 PD1 PD2 PD3 PD4 PD5 PD6 PD7
```

那么可以一次性读取 GPIOD 的低 8 位。

简单理解：

```text
PD0 → S1
PD1 → S2
PD2 → S3
PD3 → S4
PD4 → S5
PD5 → S6
PD6 → S7
PD7 → S8
```

每一路都表示一个传感器状态。

---

## 2. 控制电机方向

TB6612 控制一个电机需要两个方向引脚。

例如左电机：

```text
IN1 = 1, IN2 = 0    正转
IN1 = 0, IN2 = 1    反转
IN1 = 1, IN2 = 1    刹车
IN1 = 0, IN2 = 0    滑行
```

所以 GPIO 输出高低电平，就能控制电机方向。

---

## 3. PWM 引脚不是普通输出

电机速度通常不是用普通 GPIO 控制，而是用 PWM 控制。

例如：

```text
PA6 → 左电机 PWM
PA7 → 右电机 PWM
```

这两个引脚需要配置为：

```text
复用推挽输出 GPIO_Mode_AF_PP
```

因为 PWM 信号由定时器 TIM3 自动产生，不是手动 `SetBits` 或 `ResetBits` 输出。

---

# 七、GPIO 初始化示例

## 1. 红外传感器输入初始化

```c
void Tracker_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2 | GPIO_Pin_3 |
                                  GPIO_Pin_4 | GPIO_Pin_5 | GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_Init(GPIOD, &GPIO_InitStructure);
}
```

---

## 2. 电机方向引脚初始化

```c
void Motor_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9 | GPIO_Pin_10 | GPIO_Pin_11 | GPIO_Pin_12;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_Init(GPIOD, &GPIO_InitStructure);
}
```

---

## 3. PWM 引脚初始化

```c
void PWM_GPIO_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;

    GPIO_Init(GPIOA, &GPIO_InitStructure);
}
```

---

# 八、GPIO 常见错误

## 1. 忘记开启时钟

错误：

```c
GPIO_Init(GPIOD, &GPIO_InitStructure);
```

但是前面没有：

```c
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOD, ENABLE);
```

结果：

```text
GPIO 不工作
读取值异常
输出没有反应
```

---

## 2. 输入输出模式设置错

例如：

```text
红外传感器应该设置为输入，却设置成输出
电机方向引脚应该设置为输出，却设置成输入
PWM 引脚应该设置成复用推挽，却设置成普通推挽
```

都会导致功能异常。

---

## 3. 没有共地

单片机、传感器、电机驱动模块必须共地。

```text
STM32 GND
传感器 GND
TB6612 GND
电池负极
```

这些要连在一起。

否则 GPIO 读取的高低电平没有统一参考，可能读不准。

---

## 4. 电平不匹配

STM32F103 是 3.3V 单片机。

需要注意：

```text
GPIO 输出高电平约为 3.3V
不要随便把 5V 信号直接输入到不耐 5V 的引脚
```

部分 STM32F103 引脚可以容忍 5V，但不是所有引脚都可以，实际接线时要看芯片手册。

---

## 5. 引脚被其他功能占用

有些引脚可能默认用于：

```text
JTAG
SWD
晶振
复位
特殊复用功能
```

如果你把这些引脚当普通 GPIO 用，可能会冲突。

常见情况：

```text
PA13、PA14 是 SWD 下载调试引脚，不建议随便占用
PB3、PB4、PA15 和 JTAG 有关
```

---

# 九、GPIO 学习重点总结

你做循迹小车，GPIO 最重要学会这些：

```text
1. GPIO 是什么
2. 输入和输出的区别
3. 上拉输入、下拉输入、浮空输入
4. 推挽输出、开漏输出
5. 复用推挽输出
6. GPIO 初始化步骤
7. 如何读取传感器电平
8. 如何控制电机方向
9. PWM 引脚为什么要配置成复用功能
10. 共地、电平匹配、时钟使能这些常见问题
```

---

# 十、最小掌握版本

如果只为了先把循迹小车跑起来，可以先重点掌握：

```text
GPIO 输入：
用于读取 8 路红外传感器 PD0~PD7

GPIO 输出：
用于控制 TB6612 电机方向 PD9/PD10/PD11/PD12

GPIO 复用输出：
用于 TIM3 输出 PWM，PA6/PA7
```

记住一句话：

```text
传感器用输入，电机方向用输出，PWM 引脚用复用输出。
```