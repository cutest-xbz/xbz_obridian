下面是做 **STM32 循迹小车**必须掌握的 **PD 控制知识点**。

# PD 控制知识点

## 一、PD 控制是什么

PD 控制是 PID 控制的一种简化形式。

PID 分别是：

```text
P：比例控制
I：积分控制
D：微分控制
```

PD 控制就是只使用：

```text
P + D
```

不用 I 项。

在循迹小车中，PD 控制常用于根据红外传感器计算出来的偏差 `error`，生成转向修正量 `turn`。

你的项目中使用的就是类似 PD 控制：

```text
u(k) = Kp × e(k) + Kd × [e(k) - e(k-1)]
```

其中 `Ki = 0`，默认参数为 `Kp=1.2`，`Kd=0.4`。

---

# 二、PD 控制在循迹小车中的作用

循迹小车的控制流程是：

```text
红外传感器检测黑线位置
        ↓
计算小车偏离黑线的误差 error
        ↓
PD 控制器根据 error 计算 turn
        ↓
左右轮差速转向
```

简单说：

```text
error 表示车偏了多少
PD 控制器决定应该修正多少
turn 表示左右轮速度差
```

---

# 三、P 控制：比例项

P 是 Proportional，比例控制。

公式：

```text
P = Kp × error
```

含义：

```text
偏差越大，修正越大
偏差越小，修正越小
```

例如：

```text
error = 0.5
Kp = 1.2

P = 1.2 × 0.5 = 0.6
```

说明小车偏得比较明显，需要较大的转向修正。

---

## P 项的作用

P 项主要负责：

```text
让小车快速回到黑线中心
```

如果没有 P 项，小车即使偏离黑线，也不会主动修正。

---

## Kp 太小的现象

```text
小车反应慢
修正不及时
弯道容易冲出去
车身慢慢偏离黑线
```

解决方法：

```text
适当增大 Kp
```

---

## Kp 太大的现象

```text
小车左右摆动严重
走 S 形
转向过猛
甚至来回震荡
```

解决方法：

```text
适当减小 Kp
或者增加 Kd 抑制摆动
```

---

# 四、D 控制：微分项

D 是 Derivative，微分控制。

离散程序中一般不用真正求导，而是用：

```text
D = Kd × (当前误差 - 上一次误差)
```

也就是：

```text
D = Kd × (error - lastError)
```

---

## D 项的含义

D 项关注的不是“现在偏了多少”，而是：

```text
偏差变化得有多快
```

例如：

```text
上一次 error = 0.1
当前 error = 0.6
```

说明小车偏离黑线的速度很快，需要提前抑制。

---

## D 项的作用

D 项主要负责：

```text
抑制小车左右摆动
让转向更平稳
减少过度修正
```

可以理解为：

```text
P 项负责拉回来
D 项负责别拉过头
```

---

## Kd 太小的现象

```text
小车容易左右摆动
修正后又冲过中心线
走线不稳定
```

解决方法：

```text
适当增大 Kd
```

---

## Kd 太大的现象

```text
小车反应变迟钝
转向不够自然
对传感器噪声特别敏感
电机输出容易抖动
```

解决方法：

```text
适当减小 Kd
```

---

# 五、为什么循迹小车常用 PD，而不是完整 PID

循迹小车通常只需要让车快速回到黑线附近。

所以：

```text
P 项负责根据偏差修正方向
D 项负责抑制摆动
I 项通常可以不用
```

I 项是积分项，主要用来消除长期稳定误差。

但是循迹小车中使用 I 项容易带来问题：

```text
积分累积过多
过弯时反应慢
脱线后积分异常增大
容易导致小车突然大幅转向
```

所以初学循迹小车时，建议先用：

```text
PD 控制
```

也就是：

```text
Ki = 0
```

---

# 六、PD 控制公式

循迹小车常用公式：

```text
turn = Kp × error + Kd × (error - lastError)
```

其中：

```text
turn：转向修正量
Kp：比例系数
Kd：微分系数
error：当前偏差
lastError：上一次偏差
```

计算完以后，要更新：

```text
lastError = error
```

---

# 七、PD 控制和差速转向的关系

PD 控制算出来的是 `turn`，不是直接控制电机。

电机控制还需要差速公式：

```text
left  = basePwm + turn
right = basePwm - turn
```

完整关系：

```text
error → PD → turn → 差速转向 → PWM 输出
```

例如：

```text
basePwm = 0.5
error = 0.3
lastError = 0.1
Kp = 1.2
Kd = 0.4
```

计算：

```text
turn = 1.2 × 0.3 + 0.4 × (0.3 - 0.1)
     = 0.36 + 0.08
     = 0.44
```

左右轮：

```text
left  = 0.5 + 0.44 = 0.94
right = 0.5 - 0.44 = 0.06
```

说明需要比较明显地向一侧修正。

---

# 八、PD 控制代码示例

```c
typedef struct
{
    float kp;
    float kd;
    float lastError;
    float outputMax;
} PD_Controller;
```

初始化：

```c
void PD_Init(PD_Controller *pd, float kp, float kd, float outputMax)
{
    pd->kp = kp;
    pd->kd = kd;
    pd->lastError = 0.0f;
    pd->outputMax = outputMax;
}
```

计算函数：

```c
float PD_Calc(PD_Controller *pd, float error)
{
    float p;
    float d;
    float output;

    p = pd->kp * error;
    d = pd->kd * (error - pd->lastError);

    output = p + d;

    pd->lastError = error;

    if(output > pd->outputMax)
    {
        output = pd->outputMax;
    }
    else if(output < -pd->outputMax)
    {
        output = -pd->outputMax;
    }

    return output;
}
```

使用：

```c
error = Tracker_CalcError();
turn = PD_Calc(&pd, error);

left  = basePwm + turn;
right = basePwm - turn;
```

---

# 九、为什么要输出限幅

PD 计算出来的 `turn` 可能过大。

例如：

```text
turn = 1.5
```

但是 PWM 通常只允许：

```text
0.0 ~ 1.0
```

如果不限制，可能导致：

```text
一侧电机满速
另一侧电机为负数
小车突然急转
循迹不稳定
```

所以要限制：

```text
turn 最大不超过某个值
```

你的项目中 PID 输出限幅为 `±0.8`。

---

# 十、PD 控制调参顺序

## 第一步：先只调 P

先设置：

```text
Kd = 0
```

然后慢慢调 Kp。

从小到大增加 Kp：

```text
Kp 太小：车不怎么修正
Kp 合适：能跟线，但有轻微摆动
Kp 太大：左右疯狂摆动
```

找到一个刚好能跟线但开始有点摆动的 Kp。

---

## 第二步：再加 D

在 Kp 基本能跟线后，逐渐增加 Kd。

```text
Kd 太小：摆动明显
Kd 合适：车身平稳
Kd 太大：反应迟钝，甚至抖动
```

---

## 第三步：调整 basePwm

PD 参数不是孤立的，车速也会影响效果。

```text
basePwm 越大，需要更强的修正能力
basePwm 越小，小车更容易稳定
```

建议先低速调：

```text
basePwm = 0.3 ~ 0.5
```

调稳定后再提高速度。

---

# 十一、PD 控制常见问题

## 1. 小车左右疯狂摆动

可能原因：

```text
Kp 太大
Kd 太小
basePwm 太大
红外误差方向反了
```

解决：

```text
减小 Kp
增大 Kd
降低 basePwm
检查 error 正负方向
```

---

## 2. 小车转弯不及时

可能原因：

```text
Kp 太小
turn 限幅太小
basePwm 太大
传感器装得太靠后
```

解决：

```text
增大 Kp
适当增大输出限幅
降低 basePwm
把红外模块尽量装在车头前方
```

---

## 3. 小车抖动很厉害

可能原因：

```text
Kd 太大
红外传感器读数不稳定
黑线边缘检测跳变
采样周期太短或不稳定
```

解决：

```text
减小 Kd
检查红外模块阈值
给 error 做简单滤波
保证控制周期稳定
```

---

## 4. 小车总是向反方向修正

可能原因：

```text
error 正负方向反了
turn 正负方向反了
left/right 差速公式写反了
左右电机接反了
```

解决：

```text
把 error 取反
或者把 turn 取反
或者检查左右电机方向
```

---

# 十二、PD 控制和控制周期

PD 控制要固定周期执行。

你的项目是：

```text
SysTick 1ms → 调度器 → 每 10ms 执行 CarDrive_Run()
```

也就是控制周期为 10ms。

控制周期稳定很重要：

```text
周期稳定：D 项计算稳定
周期忽快忽慢：D 项容易异常
```

所以不要在控制循环里写很长的阻塞延时。

---

# 十三、PD 控制调试步骤

建议按这个顺序调：

```text
1. 先确认红外读取正确
2. 确认 error 方向正确
3. 确认左右电机方向正确
4. 确认差速转向方向正确
5. 设置 Kd = 0，只调 Kp
6. 小车能跟线后，再慢慢增加 Kd
7. 加输出限幅，防止急转
8. 调整 basePwm
9. 测试直线、弯道、急弯
10. 最后记录一组稳定参数
```

---

# 十四、PD 控制最小掌握版本

只为了把循迹小车跑起来，重点记住：

```text
1. PD 控制用来根据 error 计算 turn
2. P 项负责快速修正偏差
3. D 项负责抑制左右摆动
4. turn = Kp × error + Kd × (error - lastError)
5. Kp 太小转不过来，Kp 太大疯狂摆动
6. Kd 太小摆动明显，Kd 太大容易抖动或迟钝
7. 先调 Kp，再调 Kd，最后调 basePwm
8. PD 输出要限幅
9. 控制周期要固定
10. 初学循迹小车一般先不用 I 项
```

一句话总结：

```text
P 项让车回到黑线，D 项让车别冲过头；PD 控制算出 turn，再通过左右轮差速完成循迹。
```