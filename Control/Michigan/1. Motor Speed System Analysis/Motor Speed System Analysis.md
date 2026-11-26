# 1. Modeling

> [!info] 参考资料
> - 来源：[DC Motor Speed: System Modeling (UMich CTMS)](https://ctms.engin.umich.edu/CTMS/index.php?example=MotorSpeed&section=SystemModeling)
> - 对象：**电枢控制直流电机（Armature-controlled DC Motor）**

---

## 1.1 Physical Setup（物理模型与参数）

电枢控制直流电机，励磁磁场恒定（Fixed field）：
- **Electrical（电气）**：电枢电压 $v$、电阻 $R$、电感 $L$、电流 $i$、反电动势 $e$
- **Mechanical（机械）**：转动惯量 $J$、粘性阻尼 $b$、转角 $\theta$、角速度 $\omega = \dot{\theta}$、转矩 $T$

![376](assets/Motor%20Speed%20System%20Analysis/file-20261126230708922.png)

### 物理关系（机电耦合）
> 磁场恒定条件下：
> - 转矩 $T$ 正比于电流 $i$
> - 反电动势 $e$ 正比于角速度 $\dot{\theta}$

$$
\begin{aligned}
T &= K_t i \\
e &= K_e \dot{\theta}
\end{aligned}
$$

> [!note] 为什么 $K_t = K_e$？（能量守恒）
> 电源总功率 $V i$ 分成三部分（KVL 乘以 $i$）：
> $$
> V i = i^2 R + \frac{d}{dt}\left(\frac{1}{2}Li^2\right) + e i
> $$
> - $i^2 R$：电阻发热（铜损）
> - $\frac{d}{dt}(\frac{1}{2}Li^2)$：电感储能变化
> - **$e i$：克服反电动势、真正转为机械功的电功率**
>
> 由能量守恒，这部分转换电功率 $P_{\text{conv}}$ 等于电磁机械功率 $P_{\text{mech}}$：
> $$
> \begin{aligned}
> P_{\text{conv}} &= e i = (K_e \dot{\theta}) i \\
> P_{\text{mech}} &= T \dot{\theta} = (K_t i) \dot{\theta}
> \end{aligned}
> $$
> 令 $P_{\text{conv}} = P_{\text{mech}}$，得：
> $$
> K_e \dot{\theta} i = K_t i \dot{\theta} \implies K_e = K_t = K
> $$
> SI 单位下 $1\ \text{V}\cdot\text{s/rad} = 1\ \text{N}\cdot\text{m/A}$，数值相等。

### 系统参数表

|  符号   | 物理含义                                      |   数值   | 单位（SI）                               |
| :---: | :---------------------------------------- | :----: | :----------------------------------- |
|  $J$  | 转子转动惯量（Moment of inertia of the rotor）    | $0.01$ | $\text{kg}\cdot\text{m}^2$           |
|  $b$  | 电机粘性阻尼系数（Motor viscous friction constant） | $0.1$  | $\text{N}\cdot\text{m}\cdot\text{s}$ |
| $K_e$ | 反电动势常数（Electromotive force constant）      | $0.01$ | $\text{V/rad/sec}$                   |
| $K_t$ | 电机转矩常数（Motor torque constant）             | $0.01$ | $\text{N}\cdot\text{m/Amp}$          |
|  $R$  | 电枢电阻（Electric resistance）                 |  $1$   | $\Omega$                             |
|  $L$  | 电枢电感（Electric inductance）                 | $0.5$  | $\text{H}$                           |

- **Input**：电枢电压 $V$
- **Output**：转轴角速度 $\dot{\theta}$（或 $\omega$）

---

## 1.2 System Equations

### 1.2.1 时域微分方程

#### 1. 机械部分（牛顿第二定律）
转子合外力矩等于转动惯量乘以角加速度：

$$
J\ddot{\theta} = \sum \tau = T - T_{\text{friction}}
$$

代入 $T = Ki$ 和阻尼力矩 $T_{\text{friction}} = b\dot{\theta}$：

$$
J\ddot{\theta} + b\dot{\theta} = Ki
$$

#### 2. 电气部分（KVL）
电枢回路满足基尔霍夫电压定律：

$$
V = V_R + V_L + e = Ri + L\frac{di}{dt} + K\dot{\theta}
$$

整理得：

$$
L\frac{di}{dt} + Ri = V - K\dot{\theta}
$$

#### 3. 方程汇总

$$
\begin{aligned}
J\ddot{\theta} + b\dot{\theta} &= Ki \\
L\frac{di}{dt} + Ri &= V - K\dot{\theta}
\end{aligned}
$$

---

### 1.2.2 Transfer Function

> 在 $s$ 域描述输入-输出关系，把微分方程化为代数方程
> 适用于经典控制（极点分析、PID、频域/根轨迹设计）

零初始条件下做**拉普拉斯变换**：

$$
\begin{aligned}
s(Js + b)\Theta(s) &= KI(s) \\
(Ls + R)I(s) &= V(s) - Ks\Theta(s)
\end{aligned}
$$

消去 $I(s)$：

$$
s(Js + b)\Theta(s) = K \cdot \frac{V(s) - Ks\Theta(s)}{Ls + R}
$$

得开环传递函数（输出角速度 $\dot{\Theta}(s) = s\Theta(s)$）：

$$
P(s) = \frac{\dot{\Theta}(s)}{V(s)} = \frac{K}{(Js + b)(Ls + R) + K^2} \quad \left[\frac{\text{rad/sec}}{\text{V}}\right]
$$

---

### 1.2.3 State-Space

> 在时域用一阶矩阵方程保留内部状态（如 $\dot{\theta}$ 与 $i$）
> 支持非零初始条件、多输入多输出（MIMO）及现代控制（如状态反馈、LQR）

取状态向量 $\mathbf{x} = \begin{bmatrix} \dot{\theta} \\ i \end{bmatrix}$，输入 $u = V$，输出 $y = \dot{\theta}$。

改写为一阶方程组：

$$
\begin{aligned}
\ddot{\theta} &= -\frac{b}{J}\dot{\theta} + \frac{K}{J}i \\
\frac{di}{dt} &= -\frac{K}{L}\dot{\theta} - \frac{R}{L}i + \frac{1}{L}V
\end{aligned}
$$

矩阵形式（$\dot{\mathbf{x}} = A\mathbf{x} + Bu,\ y = C\mathbf{x} + Du$）：

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

针对 **$1\ \text{rad/sec}$ 阶跃输入**：

| 性能指标 | 要求 | 说明 |
| :--- | :---: | :--- |
| **Settling time（调节时间）** | $< 2\ \text{s}$ | 快速稳定 |
| **Overshoot（超调量）** | $< 5\%$ | 防止过冲损坏设备 |
| **Steady-state error（稳态误差）** | $< 1\%$ | 稳态精确跟踪 |

> [!tip] 为什么需要控制器？
> 开环直流增益 $P(0) = \dfrac{0.01}{0.1001} \approx 0.1$（输入 $1\ \text{V}$ 仅得 $\approx 0.1\ \text{rad/sec}$）
> 远未达到设计要求，需引入反馈控制
