# 小车题第三问与陀螺仪惯导移植记录

当前实现对应：

- 第三问状态机：`src/car/empty/App/h3_task.c/.h`
- 陀螺仪串口接收：`src/car/empty/Bsp/gyro_uart.c/.h`
- 连续航向与超时检测：`src/car/empty/App/gyro_fusion.c/.h`
- 编码器里程计：`src/car/empty/App/odometry.c/.h`

## 流程

KEY2 长按循环选择模式 A/B/C，OLED 显示当前模式；KEY1 长按后进入等待松手并启动当前模式。

- A：AB 航向保持直线，到 B 停车。
- B：AB 航向保持直线、BC 半圆循迹、CD 航向保持直线、DA 半圆循迹，到 A 停车。
- C：第三问流程，包含 AC/BD 航向保持、CB/DA 半圆循迹及 B 点转向。

半圆段启用专用丢线策略：短暂丢线时保持最近弯向并继续前进，重新检测到黑线后恢复位置 PID，避免原地搜索破坏半圆轨迹。

## 当前引脚

- 陀螺仪 UART1：PA8 TX、PA9 RX，115200
- OLED 硬件 I²C1：PB2 SCL、PB3 SDA
- KEY1/2/3：PA0/PA27/PA1
- 状态 LED/蜂鸣器：PA17/PA12

## 移植顺序

先确认 OLED 能显示 `heading、dps、frame_count、valid`，再标定陀螺仪零点和正方向；之后低速验证直线航向保持，最后验证第三问状态机。`H3_GYRO_TO_MOTOR_SIGN` 和顶部速度/PD 参数需要实车标定。

## 当前控制架构

当前版本是“陀螺仪航向 PD 修正 + 电机 PWM 开环输出”：陀螺仪航向误差产生左右轮目标 PWM，`SpeedCtrl_Update()` 当前通过 `SPEED_CTRL_ENCODER_DISABLED=1` 直接输出 PWM。编码器只用于里程计，尚未作为速度内环反馈。若后续启用速度内环，需要同时启用 `Encoder_Init()`、设置速度单位和重新标定 PID。
Mode B arc exit guard (latest): the BC/DA arc is not considered complete on a
short line dropout. The heading must first reach 160 degrees (or more), then
the center sensors (S4/S5, mask `0x18`) must reacquire the straight line with
`abs(deviation) <= 10` for 5 consecutive control ticks. This prevents an
early switch to CD/AB heading hold while the vehicle is still turning out of
the semicircle. Tune `H3_MODE_B_ARC_HEADING_DEG` or
`H3_ARC_CENTER_STABLE_TICKS` in `h3_task.c` if the physical track requires it.
