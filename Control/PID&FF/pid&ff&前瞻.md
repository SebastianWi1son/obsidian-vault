# 1. 速度与力矩的电气对应
- voltage(pwm) -- velocity
- current -- torque

# 2. P term & I term
- p是瞬态响应
- i是积累
- 两者工作互不干扰
- 结果互相耦合

# 3. FeedForward
>  普通Integral项职责:
>  短时快速积累 + 长期精细积累
>  两者耦合
>  前馈直接取缔 短时快速积累

# 4. 加速度环(力矩or电流环)
> 加速度 ∝ 力矩 ∝ 电流
> 加速度环需要对速度再微分
> 非常容易受噪声干扰
> 使用加速度限幅acc_limiter(加速度环青春版)


# 5. 前瞻


