1. 输入量测流,
> 1. 内部维护"上一时刻估计" + “算法特有状态”,
> 2. 输出当前最优估计

## 1. concept
### 1.1 `旋转`和`姿态`在四元数等价
- 一个**旋转动作**可由**单位四元数**表示
- 一个**姿态状态**可以看作是**body系转至world系的旋转**

### 1.2 Hamilton Product

| 方式      | 复合两次旋转的代价                  | 万向锁 |
| ------- | -------------------------- | --- |
| 欧拉角     | 要转矩阵或重算三角                  | ❌ 有 |
| 旋转矩阵    | 3×3 相乘 ≈ 27 次乘法            | ✅ 无 |
| **四元数** | **一次 Hamilton 积 = 16 次乘法** | ✅ 无 |
**"先施加 q2 的旋转，再施加 q1 的旋转":**
```
q1 ⊗ q2
```

| 限制            | 说明                                               |
| ------------- | ------------------------------------------------ |
| **不满足交换律**    | 旋转先后顺序不同对应不同旋转                                   |
| **左右乘不同**     | 1. `q⊗r` = 在**body系**叠加  2.`r⊗q` = 在**world系**叠加 |
| **只对单位四元数成立** | 非单位四元数的乘法代数上能算，但**不对应纯旋转**                       |
PS: **叠加旋转获得姿态**是一种**手段**
Extend:
1. q, -q表示**同一个旋转**
2. 满足分配律

### 1.3 Mahony与EKF差异
1. Mahony修正correct在角速度omega, 与外推predict操作耦合
```cpp
estimator<Mahony>::predict 耦合外推+修正
```
2. EKF修正correct与predict外推解耦
```cpp
estimator::correct()
estimator::predict()
```

### 1.4 `cos(θ/2), û·sin(θ/2)` -> `(1, ½ω·dt)`
前置公式知识:
1. 旋转 θ 弧度、绕轴 û 的四元数 :
```
( cos(θ/2),  û·sin(θ/2) )
```
2. 这一帧转过的角度:
```
θ = |ω|·dt
```
3. ω 可写成"长度 × 方向": 
```
ω = |ω|·û
```
综合: +前置条件: θ 很小，趋近于0
```
û·sin(θ/2)
    ≈ û·(θ/2)     一阶近似
    = (θ·û)/2
    = (ω·dt)/2      
    = ½·ω·dt        
```
> `û·sin(θ/2)` 里 `û` 是方向、`sin(θ/2)` 是长度
> 而 `ω·dt` **本身就是"方向 û、长度 θ"**，所以可互换

### 1.5 单位四元数共轭
**单位四元数的共轭 = 单位四元数的逆**

## 2. 真实代码实现 与 理论
### 2.1 外推
1. **理论:** 通过在旧姿态值叠加此刻q_dot, 获得新值q_k+1
```
q_k + q_dot*dt ---> q_k+1
```

2. **代码实现:** 
```
新姿态 = 老姿态 ⊗ (这一帧的旋转)
       = q ⊗ ( cos(θ/2), û·sin(θ/2) )     ← ① 精确，但每帧要算 cos/sin（慢）
       ≈ q ⊗ ( 1, ½·ω·dt )                ← ② 近似！丢掉 cos/sin 高阶项
       = q ⊗ (1,0,0,0) + q ⊗ (0, ½·ω·dt)  ← ③ 精确恒等式（乘法分配律）
       = q + ½·q ⊗ (0, ω) · dt            ← ④ 精确
       → normalize                        ← ⑤ 收拾 ② 的残留
```

```cpp
math::Quat::integrate(const Vec3<T>& omega, T dt) {
	T half = T(0.5) * dt;
    T wx = omega.x_, wy = omega.y_, wz = omega.z_;
    // q_dot =  1/2 * q ⊗ (0, w1, w2, w3)
    // q += q_dot * dt
    T q0 = q0_ + (-q1_ * wx - q2_ * wy - q3_ * wz) * half;
    T q1 = q1_ + ( q0_ * wx + q2_ * wz - q3_ * wy) * half;
    T q2 = q2_ + ( q0_ * wy - q1_ * wz + q3_ * wx) * half;
    T q3 = q3_ + ( q0_ * wz + q1_ * wy - q2_ * wx) * half;

    q0_ = q0; q1_ = q1; q2_ = q2; q3_ = q3;
    return normalize();
}
```
- 没有q_dot
- 没有两个时刻的q_k, q_k+1, 只有一个q自我更新

### 2.2 归一化 (模长回一)

```cpp
T norm_squared() const { return q0_ * q0_ + q1_ * q1_ + q2_ * q2_ + q3_ * q3_; }
T norm() const { return std::sqrt(norm_squared()); }
Quat& normalize() {
    T n2 = norm_squared();
    if (n2 > T(0)) {
        T inv = inv_sqrt(n2);
        q0_ *= inv; q1_ *= inv; q2_ *= inv; q3_ *= inv;
    }
    else { *this = identity(); }        // 零四元数无意义，复位为单位元
    return *this;
}
```
原因在于[外推时q新叠加量的一阶近似](#1.4%20`cos(θ/2),%20û·sin(θ/2)`%20->%20`(1,%20½ω·dt)`)丢失了cos/sin高阶项
> **一阶近似**使 结果四元数**模长大于1**
> 而**只有单位四元数才可以表示旋转/姿态**

### 2.3 estimator::predict (唯一改变q_的函数)
1. 把量测修正 折进角速度 (消费correct)
2. 用该角速度把姿态往前推dt秒(外推)
注: [Mahony在该函数下存在约定式歧义](#1.3%20Mahony与EKF差异)

### 2.4 Mahony::observe
#### 2.4.1 e_last_ & e_int_
| 账本     | 内容      | 给谁用 | 直觉说明                    |
| ------ | ------- | --- | ----------------------- |
| e_last | 此次测量的误差 | Kp  | 瞬时P项纠正                  |
| e_int  | 累计的误差   | Ki  | 持续一个方向的误差是零偏zero_offset |

## 3. replay
### 3.1 意义
- **replay = 把录好的真实数据重新"播放"一遍喂给算法，然后对答案。**
- **它解决什么**：核心算法在桌面上**没法接真实 IMU**
- 用**别人录好的数据集**一长串 IMU 读数 **+** 每个时刻对应的真值姿态
```
1. load_dataset()   读文本 → 15999 行（acc3+gyro3+mag3+真值euler3）；50Hz → 320 秒
2. Estimator<> est; est.reset(初始 yaw = 真值)   ← 使评估的是"相对漂移"而非初始偏差
3. for (i = 0; i < 15999; i++) {          ★ 一帧一帧"播放"
       IMUSample s = {-acc[i], gyro[i]};  ← 坐标系适配（z-down → z-up）
       est.observe(s, 0.02f);             ← 备料
       est.predict(s, 0.02f);             ← 推进一步
       误差 = est.euler() - 真值[i];       ← ★ 对标准答案
       累计 误差²                          ← 用平方让负数不抵消
   }
4. RMSE = √(Σ误差² / 帧数)                ← "平均每帧差了几度"
```
### 3.2 predict与observe顺序问题
- 真实项目中推荐使用:      predict在前，observe在后
- 实际回放replay样本中: observe在前，predict在后


