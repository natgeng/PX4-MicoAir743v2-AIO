# PX4 v1.17.0 · MicoAir743v2-AIO-35A 二次开发

> 基于 PX4 v1.17.0 的两个独立改动分支，都在同一块板子上编译与验证。
> 本仓库是 [PX4/PX4-Autopilot](https://github.com/PX4/PX4-Autopilot) 的 fork，`main` 保持上游，改动全部在下面的分支里。

| 项目 | 值 |
|---|---|
| 基线 | PX4 v1.17.0（`d6f12ad1c4f70ad3230afd7d86e971421e02fef4`） |
| 硬件 | MicoAir743v2-AIO-35A（PX4 target `micoair_h743-v2`，board_id 1179） |
| 飞控 | STM32H743VIH6，BMI088 + BMI270，SPL06 |
| 电调 | 4 × 35 A **Bluejay 0.19.2**（8-bit SiLabs EFM8） |
| 固件余量 | app 约 1.9 MB / 上限 1.92 MB，**剩余约 61 KB** |

> 注意：45A 版是 **AM32**（32-bit），电调能力和本分支的假设不同。
> 也不要把第一代 `MicoAir743-AIO-35A`（非 v2，AM32）混进来。

---

# 分支导航

| 分支 | 内容 | 状态 |
|---|---|---|
| [`fix/bdshot-micoair743v2-aio`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/tree/fix/bdshot-micoair743v2-aio) | 双向 DShot 修复 —— 让本板的 RPM 遥测能用 | 可用 |
| [`feat/flip-recovery`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/tree/feat/flip-recovery) | 侧翻 / 倒扣自动翻正 —— 新增一个飞行模式 | 模式与算法完成，电机反向待接入 |

两个分支各自独立编译，都**已包含** `bdshot` 修复。

```
v1.17.0
  └── fix/bdshot-micoair743v2-aio   （双向 DShot 修复）
        └── feat/flip-recovery      （+ 侧翻自动纠正）
```

---

# 一、侧翻自动纠正 flip_recovery

> 完整文档：[`docs/flip-recovery/README.md`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/blob/feat/flip-recovery/docs/flip-recovery/README.md)

## 它做什么

落地侧翻或完全倒扣后，用一个**原生飞行模式**接管电机：按当前姿态自动选出"高位两路"和"低位两路"，
让高位产生力矩把机身翻回正立，然后自动切回常规模式。

- 高位离翻正转轴最远，力臂最大，负责产生力矩
- 低位就在转轴上，力臂≈0，只负责"抬一点"卸掉地面压力、减小摩擦

## 三步用起来

### 1. 编译刷写

```bash
git checkout feat/flip-recovery
make micoair_h743-v2_default
```

只刷 app，**不要动 bootloader**。QGC 选 `.px4` 文件即可。

### 2. 参数

```
param set FLIP_EN 1            # 总开关，0 = 拨进模式也不转桨
param set COM_FLTMODE5 17      # 把第 5 档模式开关映射成 Flip Recovery
param save
```

| 参数 | 默认 | 作用 |
|---|---|---|
| **`FLIP_EN`** | 0 | 总开关 |
| **`FLIP_THR_HIGH`** | 0.20 | 高位组油门（决定翻正快慢），从 0.15 起步 |
| **`FLIP_THR_LOW`** | 0.10 | 低位组油门（卸压/减摩擦），应明显低于高位 |
| **`FLIP_LOW_MODE`** | 0 | 低位转向：0=Auto 1=强制正转 2=强制反转 3=关闭 |
| **`FLIP_MODE`** | 0 | 0=翻过去（90–180° 抬高位翻过 180°）、1=压回去 |
| `FLIP_ENTRY_ANG` | 75 | 超过这个倾角才出力 |
| `FLIP_STOP_ANG` | 60 | 低于这个倾角收油，交给重力 |
| `FLIP_EXIT_MODE` | 2 | 翻完自动切回的模式（2 = Position） |
| `FLIP_RC_CONFIRM` | 0 | 0 = 免二次确认；1–6 = 还要对应 aux 通道在高位 |
| `FLIP_SLEW` / `FLIP_TIME` | 0.50 / 3.0 | 油门上升率 / 总出力上限 |
| `FLIP_MIN_PROG` / `FLIP_PROG_T` | 3.0 / 0.80 | 进度检查阈值 / 起始时间 |

### 3. 触发（任选一种）

四条路径最终都落到 `vehicle_status.nav_state = 16 (NAVIGATION_STATE_FLIP_RECOVERY)`。

**遥控器拨杆**：`COM_FLTMODE5 = 17` 设好后拨到第 5 档。

**MAVLink `SET_MODE`**：

```python
master.mav.set_mode_send(
    master.target_system,
    mavutil.mavlink.MAV_MODE_FLAG_CUSTOM_MODE_ENABLED,
    (12 << 16)          # custom_mode = main_mode(12) << 16 | sub_mode(0)
)
```

**MAVLink `MAV_CMD_DO_SET_MODE`**：

```python
master.mav.command_long_send(
    master.target_system, master.target_component,
    mavutil.mavlink.MAV_CMD_DO_SET_MODE, 0,
    1,      # CUSTOM_MODE_ENABLED
    12,     # PX4_CUSTOM_MAIN_MODE_FLIP_RECOVERY
    0, 0, 0, 0, 0
)
```

**uORB / ROS2**：

```python
msg.command = 176     # VEHICLE_CMD_DO_SET_MODE
msg.param1 = 1.0
msg.param2 = 12.0
msg.param3 = 0.0
# 更直接：
msg.command = 10071   # VEHICLE_CMD_SET_NAV_STATE
msg.param1 = 16.0     # NAVIGATION_STATE_FLIP_RECOVERY
```

**命令行**：

```
commander mode flip
```

### 观测与退出

```
listener flip_recovery_control
flip_recovery status
```

```
flip state 1: ramping up (tilt 121.3)
flip zone 2 (tilt 121): HIGH(m0c) reversed @ 0.20 [lifts] | LOW(m03) reversed @ 0.10
flip done, switching back to mode 2
```

退出：拨杆拨回 / 发 `SET_MODE` / 翻完自动切回 / `FLIP_EN=0`。

### 安全约束（已内置）

| 约束 | 实现 |
|---|---|
| 该模式下禁止解锁 | `mode_req_prevent_arming` |
| 只在未解锁 + 已落地时出力 | 模块内 `_safe` 判定 |
| 出力超时 | `FLIP_TIME`，默认 3 s |
| 方向配反自动停 | 0.8 s 内倾角没减小就 `Failed` |
| 失败后必须退出模式才能重试 | `_rearm_needed` |

### 当前限制

- **桨能转，但方向只有出厂方向** —— DShot 反向（命令 20/21）还没接入。
  Bluejay 0.19.2 官方支持命令 21（turtle mode），所以预期可行，但需要真机确认。
- 只支持 Quad-X（`CA_ROTOR_COUNT = 4`）。

---

# 二、双向 DShot 修复 fix/bdshot-micoair743v2-aio

## 症状

配好双向 DShot 后，`esc_status` 里没有 RPM，Betaflight 那套 RPM 滤波用不上；
PX4 侧表现为电调遥测缺失。

## 根因

M1–M4 的 GPIO 复用配置里，**输出摆率和上拉都没配**，导致 DShot 反向帧（电调回传的 GCR 编码）
在信号线上被削顶，飞控收不到。

## 修复

`boards/micoair/h743-v2/src/timer_config.cpp`（以及 `h743-aio/` 对应文件）里给 `motorOutput()`
加了封装，输出/输入两个方向都设 `GPIO_SPEED_25MHz` + `GPIO_PULLUP`：

| 通道 | 修改前 | 修改后 |
|---|---|---|
| M1 | `0x8184E` | `0x0009144E` |
| M2 | `0x8184E` | `0x0009144D` |
| M3 | `0x8184E` | `0x0009144B` |
| M4 | `0x8184E` | `0x00091449` |

## 验证

```
dshot status          # 看 bidirectional dshot 是否 enabled
listener esc_status   # 看 rpm 是否有非零值（转桨时）
```

---

# 刷写注意

| 项 | 说明 |
|---|---|
| 只刷 app | QGC 选 `.px4` 文件，勾选"仅刷写飞控固件" |
| bootloader | **不要动**。MAVLink / `.px4` 刷写都不会覆盖，也不会丢电调校准 |
| 电调固件 | 改方向命令会**持久写入电调 EEPROM**，测试完务必 `dshot normal -m <n>` 复位 |
| 螺旋桨 | 任何电调/电机相关测试前必须拆桨 |

---

# 许可

沿用 PX4 的 BSD-3-Clause。`main` 分支的上游内容版权归 PX4 Development Team。
