# 1. Modeling

> [!info] 参考资料
> - 教程来源：[DC Motor Speed: System Modeling (UMich CTMS)](https://ctms.engin.umich.edu/CTMS/index.php?example=MotorSpeed&section=SystemModeling)
> - 研究对象：**电枢控制直流电机（Armature-controlled DC Motor）** 的转速系统建模

---

## 1.1 Physical Setup（物理模型与参数）

本系统采用电枢控制的直流电机，励磁磁场恒定（Fixed field）：
- **Electrical（电气部分）**：电枢输入电压 $V$（图中标注为 $v$）、电枢电阻 $R$、电枢电感 $L$、电枢电流 $i$、反电动势 $e$
- **Mechanical（机械部分）**：转子转动惯量 $J$、粘性阻尼系数 $b$、转角 $\theta$、角速度 $\omega = \dot{\theta}$、输出电磁转矩 $T$

![376](assets/Motor%20Speed%20System%20Analysis/file-20261126230708922.png)

### 物理关系（机电耦合）
假设磁场恒定，电机输出转矩 $T$ 与电枢电流 $i$ 成正比，反电动势 $e$ 与转轴角速度 $\dot{\theta}$ 成正比：

$$
\begin{aligned}
T &= K_t i \\
e &= K_e \dot{\theta}
\end{aligned}
$$

> [!note] 国际单位制（SI）下的常数简化
> 在 SI 单位制中，转矩常数 $K_t$ 与反电动势常数 $K_e$ 数值相等，因此后续推导统一记作 $K = K_t = K_e$。

### 系统参数表

| 符号 | 物理含义 | 数值 | 单位（SI） |
| :---: | :--- | :---: | :--- |
| $J$ | 转子转动惯量（Moment of inertia of the rotor） | $0.01$ | $\text{kg}\cdot\text{m}^2$ |
| $b$ | 电机粘性阻尼系数（Motor viscous friction constant） | $0.1$ | $\text{N}\cdot\text{m}\cdot\text{s}$ |
| $K_e$ | 反电动势常数（Electromotive force constant） | $0.01$ | $\text{V/rad/sec}$ |
| $K_t$ | 电机转矩常数（Motor torque constant） | $0.01$ | $\text{N}\cdot\text{m/Amp}$ |
| $R$ | 电枢电阻（Electric resistance） | $1$ | $\Omega$ |
| $L$ | 电枢电感（Electric inductance） | $0.5$ | $\text{H}$ |

- **系统输入（Input）**：电枢电压 $V$
- **系统输出（Output）**：转轴角速度 $\dot{\theta}$（也常记作 $\omega$）

---

## 1.2 System Equations（系统微分方程与数学模型）

### 1.2.1 时域微分方程

分别对机械部分和电气部分列写动力学与电路方程：

1. **机械部分（牛顿第二定律 / 转矩平衡）**：
   $$J\ddot{\theta} + b\dot{\theta} = T = Ki$$

2. **电气部分（基尔霍夫电压定律 KVL）**：
   $$L\frac{di}{dt} + Ri = V - e = V - K\dot{\theta}$$

---

### 1.2.2 传递函数模型（Transfer Function）

假设所有初始条件为零，对上述两个方程做 **拉普拉斯变换（Laplace Transform）**：

$$
\begin{aligned}
s(Js + b)\Theta(s) &= KI(s) \\
(Ls + R)I(s) &= V(s) - Ks\Theta(s)
\end{aligned}
$$

从第二个式子解出 $I(s) = \dfrac{V(s) - Ks\Theta(s)}{Ls + R}$，代入第一个式子消去中间变量 $I(s)$：

$$
s(Js + b)\Theta(s) = K \cdot \frac{V(s) - Ks\Theta(s)}{Ls + R}
$$

整理得到输入电压 $V(s)$ 到输出角速度 $\dot{\Theta}(s) = s\Theta(s)$ 的**开环传递函数**：

$$
P(s) = \frac{\dot{\Theta}(s)}{V(s)} = \frac{K}{(Js + b)(Ls + R) + K^2} \quad \left[\frac{\text{rad/sec}}{\text{V}}\right]
$$

代入参数数值（$J=0.01,\ b=0.1,\ K=0.01,\ R=1,\ L=0.5$）：

$$
P(s) = \frac{0.01}{(0.01s + 0.1)(0.5s + 1) + 0.01^2} = \frac{0.01}{0.005s^2 + 0.06s + 0.1001}
$$

---

### 1.2.3 状态空间模型（State-Space）

选取**转轴角速度 $\dot{\theta}$** 和**电枢电流 $i$** 作为状态变量：

$$
\mathbf{x} = \begin{bmatrix} \dot{\theta} \\ i \end{bmatrix}, \quad u = V, \quad y = \dot{\theta}
$$

将时域微分方程改写为一阶微分方程组：

$$
\begin{aligned}
\ddot{\theta} &= -\frac{b}{J}\dot{\theta} + \frac{K}{J}i \\
\frac{di}{dt} &= -\frac{K}{L}\dot{\theta} - \frac{R}{L}i + \frac{1}{L}V
\end{aligned}
$$

写成标准状态空间矩阵形式 $\dot{\mathbf{x}} = A\mathbf{x} + Bu,\ y = C\mathbf{x} + Du$：

$$
\frac{d}{dt}\begin{bmatrix} \dot{\theta} \\ i \end{bmatrix} =
\begin{bmatrix}
-\dfrac{b}{J} & \dfrac{K}{J} \\[8pt]
-\dfrac{K}{L} & -\dfrac{R}{L}
\end{bmatrix}
\begin{bmatrix} \dot{\theta} \\ i \end{bmatrix} +
\begin{bmatrix} 0 \\[8pt] \dfrac{1}{L} \end{bmatrix} V
$$

$$
y = \begin{bmatrix} 1 & 0 \end{bmatrix}
\begin{bmatrix} \dot{\theta} \\ i \end{bmatrix} +
\begin{bmatrix} 0 \end{bmatrix} V
$$

---

## 1.3 Design Requirements（设计要求）

对于 **$1\ \text{rad/sec}$ 阶跃参考输入（Step Reference Input）**，闭环控制系统需要满足以下性能指标：

| 性能指标 | 目标要求 | 物理意义 |
| :--- | :---: | :--- |
| **Settling time（调节时间）** | $< 2\ \text{s}$ | 电机启动或调速后能够快速稳定到目标转速 |
| **Overshoot（超调量）** | $< 5\%$ | 避免转速过冲过大导致机械负载或设备损坏 |
| **Steady-state error（稳态误差）** | $< 1\%$ | 确保稳态输出转速精确跟踪期望参考转速 |

> [!tip] 为什么要设计控制器？
> 由 1.2.2 的数值传递函数可知，开环直流增益为 $P(0) = \dfrac{0.01}{0.1001} \approx 0.1$，即输入 $1\ \text{V}$ 电压时稳态转速只有约 $0.1\ \text{rad/sec}$，远未达到要求，因此后续需要引入反馈控制器（如 PID 等）。

---

## 1.4 MATLAB Representation（MATLAB 建模代码）

### 方法一：传递函数（`tf`）

```matlab
J = 0.01;
b = 0.1;
K = 0.01;
R = 1;
L = 0.5;

s = tf('s');
P_motor = K / ((J*s + b)*(L*s + R) + K^2);
```

### 方法二：状态空间（`ss`）

```matlab
J = 0.01;
b = 0.1;
K = 0.01;
R = 1;
L = 0.5;

A = [-b/J,  K/J;
     -K/L, -R/L];
B = [0;
     1/L];
C = [1, 0];
D = 0;

motor_ss = ss(A, B, C, D);
```
