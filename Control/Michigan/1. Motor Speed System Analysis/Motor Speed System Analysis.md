# 1. Modeling

> [!info] Reference
> - Source: [DC Motor Speed: System Modeling (UMich CTMS)](https://ctms.engin.umich.edu/CTMS/index.php?example=MotorSpeed&section=SystemModeling)
> - System: **Armature-controlled DC Motor**

---

## 1.1 Physical Setup

Armature-controlled DC motor with a fixed magnetic field:
- **Electrical**: armature voltage $v$, resistance $R$, inductance $L$, current $i$, back EMF $e$
- **Mechanical**: rotor inertia $J$, viscous friction $b$, angle $\theta$, angular speed $\omega = \dot{\theta}$, torque $T$

![376](assets/Motor%20Speed%20System%20Analysis/file-20261126230708922.png)

### Physical Relations (Electromechanical Coupling)
> With a fixed magnetic field:
> - Torque $T$ is proportional to current $i$
> - Back EMF $e$ is proportional to angular speed $\dot{\theta}$

$$
\begin{aligned}
T &= K_t i \\
e &= K_e \dot{\theta}
\end{aligned}
$$

> [!note] Why $K_t = K_e$? (Conservation of Energy)
> Total supply power $V i$ splits into three parts (multiplying KVL by $i$):
> $$
> V i = i^2 R + \frac{d}{dt}\left(\frac{1}{2}Li^2\right) + e i
> $$
> - $i^2 R$: resistive heat loss (copper loss)
> - $\frac{d}{dt}(\frac{1}{2}Li^2)$: rate of magnetic energy stored in $L$
> - **$e i$: electrical power overcoming back EMF and converted to mechanical work**
>
> By conservation of energy, the converted power $P_{\text{conv}}$ equals the mechanical power $P_{\text{mech}}$:
> $$
> \begin{aligned}
> P_{\text{conv}} &= e i = (K_e \dot{\theta}) i \\
> P_{\text{mech}} &= T \dot{\theta} = (K_t i) \dot{\theta}
> \end{aligned}
> $$
> Setting $P_{\text{conv}} = P_{\text{mech}}$ gives:
> $$
> K_e \dot{\theta} i = K_t i \dot{\theta} \implies K_e = K_t = K
> $$
> In SI units, $1\ \text{V}\cdot\text{s/rad} = 1\ \text{N}\cdot\text{m/A}$, so they are numerically equal.

### System Parameters

| Symbol | Description | Value | Units (SI) |
| :---: | :--- | :---: | :--- |
| $J$ | Moment of inertia of the rotor | $0.01$ | $\text{kg}\cdot\text{m}^2$ |
| $b$ | Motor viscous friction constant | $0.1$ | $\text{N}\cdot\text{m}\cdot\text{s}$ |
| $K_e$ | Electromotive force constant | $0.01$ | $\text{V/rad/sec}$ |
| $K_t$ | Motor torque constant | $0.01$ | $\text{N}\cdot\text{m/Amp}$ |
| $R$ | Electric resistance | $1$ | $\Omega$ |
| $L$ | Electric inductance | $0.5$ | $\text{H}$ |

- **Input**: armature voltage $V$
- **Output**: shaft angular speed $\dot{\theta}$ (or $\omega$)

---

## 1.2 System Equations

### 1.2.1 Time-Domain Differential Equations

#### 1. Mechanical Part (Newton's 2nd Law)
Net torque equals rotor inertia times angular acceleration:

$$
J\ddot{\theta} = \sum \tau = T - T_{\text{friction}}
$$

Substitute $T = Ki$ and damping torque $T_{\text{friction}} = b\dot{\theta}$:

$$
J\ddot{\theta} + b\dot{\theta} = Ki
$$

#### 2. Electrical Part (KVL)
The armature loop satisfies Kirchhoff's Voltage Law:

$$
V = V_R + V_L + e = Ri + L\frac{di}{dt} + K\dot{\theta}
$$

Rearranging gives:

$$
L\frac{di}{dt} + Ri = V - K\dot{\theta}
$$

#### 3. Summary of Equations

$$
\begin{aligned}
J\ddot{\theta} + b\dot{\theta} &= Ki \\
L\frac{di}{dt} + Ri &= V - K\dot{\theta}
\end{aligned}
$$

---

### 1.2.2 Transfer Function

> Describes the input-output relation in the $s$-domain and turns ODEs into algebraic equations
> Used for classical control (pole analysis, PID, frequency response / root locus)

> [!note] Review: Laplace Transform & Differentiation Rule
> **1. Definition**
> $$
> \mathcal{L}\{f(t)\} = F(s) = \int_0^\infty f(t)e^{-st}\,dt
> $$
>
> **2. Differentiation Property** (zero initial conditions: $f(0)=\dot{f}(0)=\cdots=0 \implies \dfrac{d^n}{dt^n} \leftrightarrow s^n$)
> $$
> \begin{aligned}
> \mathcal{L}\{\dot{f}(t)\} &= sF(s) - f(0) \\
> \mathcal{L}\{\ddot{f}(t)\} &= s^2F(s) - sf(0) - \dot{f}(0) \\
> \mathcal{L}\{f^{(n)}(t)\} &= s^nF(s) - \sum_{k=0}^{n-1} s^{n-1-k}f^{(k)}(0)
> \end{aligned}
> $$

Take the **Laplace transform** assuming zero initial conditions:

$$
\begin{aligned}
s(Js + b)\Theta(s) &= KI(s) \\
(Ls + R)I(s) &= V(s) - Ks\Theta(s)
\end{aligned}
$$

Eliminate $I(s)$:

$$
s(Js + b)\Theta(s) = K \cdot \frac{V(s) - Ks\Theta(s)}{Ls + R}
$$

Open-loop transfer function (output is angular speed $\dot{\Theta}(s) = s\Theta(s)$):

$$
P(s) = \frac{\dot{\Theta}(s)}{V(s)} = \frac{K}{(Js + b)(Ls + R) + K^2} \quad \left[\frac{\text{rad/sec}}{\text{V}}\right]
$$

> [!info] Extension: What can we do with $P(s)$?
> 1. Set denominator $= 0$ for poles $\rightarrow$ stability, settling speed, and oscillation
>    Set numerator $= 0$ for zeros $\rightarrow$ overshoot and transient shape
> 2. Set $s = j\omega$ $\rightarrow$ frequency response (Bode / Nyquist plots)
> 3. Apply Final Value Theorem ($s \to 0$) $\rightarrow$ DC gain $P(0)$ and steady-state output
> 4. Form closed-loop $T(s) = \dfrac{C(s)P(s)}{1 + C(s)P(s)}$ $\rightarrow$ design controller $C(s)$ to meet specs

---

### 1.2.3 State-Space

> Keeps internal states (e.g., $\dot{\theta}$ and $i$) via first-order matrix ODEs in the time domain
> Supports non-zero initial conditions, MIMO systems, and modern control (state feedback, LQR)

> [!note] Review: Standard State-Space Form
> **1. Two Standard Equations**
> $$
> \begin{aligned}
> \dot{\mathbf{x}} &= A\mathbf{x} + Bu \\
> y &= C\mathbf{x} + Du
> \end{aligned}
> $$
> - **State equation** ($\dot{\mathbf{x}} = A\mathbf{x} + Bu$): describes how internal state $\mathbf{x}$ evolves over time
> - **Output equation** ($y = C\mathbf{x} + Du$): describes how output $y$ is obtained from state and input
>
> **2. Four Matrices ($A, B, C, D$)**
> - **$A$ (State matrix)**: how current state $\mathbf{x}$ affects $\dot{\mathbf{x}}$ (natural dynamics and stability)
> - **$B$ (Input matrix)**: how input $u$ drives state change $\dot{\mathbf{x}}$
> - **$C$ (Output matrix)**: how output $y$ is extracted from state $\mathbf{x}$
> - **$D$ (Feedthrough matrix)**: direct effect of input $u$ on output $y$ (typically $0$ in physical systems)

Choose state vector $\mathbf{x} = \begin{bmatrix} \dot{\theta} \\ i \end{bmatrix}$, input $u = V$, output $y = \dot{\theta}$.

Rewrite as first-order differential equations:

$$
\begin{aligned}
\ddot{\theta} &= -\frac{b}{J}\dot{\theta} + \frac{K}{J}i \\
\frac{di}{dt} &= -\frac{K}{L}\dot{\theta} - \frac{R}{L}i + \frac{1}{L}V
\end{aligned}
$$

Matrix form ($\dot{\mathbf{x}} = A\mathbf{x} + Bu,\ y = C\mathbf{x} + Du$):

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

## 1.3 Design Requirements

For a **$1\ \text{rad/sec}$ step input**:

| Metric | Requirement | Note |
| :--- | :---: | :--- |
| **Settling time** | $< 2\ \text{s}$ | Fast stabilization |
| **Overshoot** | $< 5\%$ | Prevent hardware damage |
| **Steady-state error** | $< 1\%$ | Accurate steady-state tracking |

> [!tip] Why do we need a controller?
> Open-loop DC gain $P(0) = \dfrac{0.01}{0.1001} \approx 0.1$ ($1\ \text{V}$ input gives only $\approx 0.1\ \text{rad/sec}$)
> Far from meeting specs, so feedback control is required
