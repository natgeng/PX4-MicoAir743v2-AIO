# `feat/flip-recovery` — 反乌龟 / 侧翻自动纠正

> 落地侧翻或倒扣后，用一个**原生飞行模式**接管电机，按当前姿态自动选出高位/低位电机，
> 把机身翻回正立，然后自动切回常规模式。

> **状态**：模式接入、姿态算法、参数体系、状态观测全部完成，可编译刷写验证。
> **DShot 出力层尚未实现**，现在刷进去只能观测，桨不会转（见第 5 节）。

分支关系：`v1.17.0` → `fix/bdshot-micoair743v2-aio`（BDShot 修复）→ **本分支**
（已包含 bdshot 修复，可独立编译刷机）

| 项目 | 值 |
|---|---|
| 基线 | PX4 v1.17.0 `d6f12ad1c4f70ad3230afd7d86e971421e02fef4` |
| 目标硬件 | MicoAir743v2-AIO-35A（target `micoair_h743-v2`，board_id 1179） |
| 新增飞行模式 | `NAVIGATION_STATE_FLIP_RECOVERY = 16`（复用 FREE1 槽位） |
| 新增参数 | 14 个 `FLIP_*` |
| 详细文档 | [`docs/flip-recovery/README.md`](docs/flip-recovery/README.md) |

---

# 使用说明（必读）

## 1. 编译与刷写

```bash
make micoair_h743-v2_default
```

只刷 app，**不要动 bootloader**。QGC 里选 `.px4` 文件即可。

## 2. 参数配置

### 必须先设的两项

```
param set FLIP_EN 1            # 总开关，0 = 拨进模式也不转桨
param set COM_FLTMODE5 17      # 把第 5 档模式开关映射成 Flip Recovery
param save
```

> `COM_FLTMODE1..6` 是模式槽位，值 17 = Flip Recovery。
> 不用拨杆的话这一项可以跳过，直接发指令触发。

### 常用可调参数

| 参数 | 默认 | 范围 | 作用 |
|---|---|---|---|
| **`FLIP_EN`** | 0 | bool | **总开关**。0 时模式可进、状态可见，但不转桨 |
| **`FLIP_THR_HIGH`** | 0.20 | 0.02–0.6 | **高位组油门**。唯一真正决定翻正快慢的参数，从 0.15 起步 |
| **`FLIP_THR_LOW`** | 0.10 | 0–0.6 | **低位组油门**。只负责卸压/减摩擦，正常应明显低于高位 |
| **`FLIP_LOW_MODE`** | 0 | 0–3 | 低位转向调试：`0`=Auto（永远抬）、`1`=强制正转、`2`=强制反转、`3`=关闭 |
| **`FLIP_MODE`** | 0 | 0–1 | 策略：`0`=翻过去（90–180° 抬高位翻过 180°）、`1`=压回去（全程压高位，路径最短） |
| `FLIP_ENTRY_ANG` | 75 | 60–180 | 超过这个倾角才出力 |
| `FLIP_STOP_ANG` | 60 | 0–120 | 低于这个倾角收油，交给重力 |
| `FLIP_EXIT_MODE` | 2 | 0–8 | 翻完自动切回的模式（2 = Position） |
| `FLIP_RC_CONFIRM` | 0 | 0–6 | 0 = 免二次确认；1–6 = 还要对应 aux 通道在高位 |
| `FLIP_SLEW` | 0.50 | 0.05–2.0 | 油门上升率（1/s） |
| `FLIP_TIME` | 3.0 | 0.2–5.0 | 总出力时长上限（s） |
| `FLIP_MIN_PROG` | 3.0 | 0–30 | 进度检查阈值（度） |
| `FLIP_PROG_T` | 0.80 | 0.2–2.0 | 多久后开始查进度（s） |
| `FLIP_OUT_DIRECT` | 0 | bool | 1 = 台架直发 `actuator_motors` |

## 3. 触发方式（任选一种）

四条路径最终都落到同一个 `vehicle_status.nav_state = 16 (NAVIGATION_STATE_FLIP_RECOVERY)`。

### 方式 A：遥控器拨杆

`COM_FLTMODE5 = 17` 设好后，把模式开关拨到第 5 档。

### 方式 B：MAVLink `SET_MODE`（推荐给上位机）

```python
from pymavlink import mavutil

master = mavutil.mavlink_connection('udp:127.0.0.1:14550')
master.wait_heartbeat()

master.mav.set_mode_send(
    master.target_system,
    mavutil.mavlink.MAV_MODE_FLAG_CUSTOM_MODE_ENABLED,
    (12 << 16)          # custom_mode = main_mode(12) << 16 | sub_mode(0)
)
```

### 方式 C：MAVLink `MAV_CMD_DO_SET_MODE`

```python
master.mav.command_long_send(
    master.target_system, master.target_component,
    mavutil.mavlink.MAV_CMD_DO_SET_MODE, 0,
    1,      # param1 = MAV_MODE_FLAG_CUSTOM_MODE_ENABLED
    12,     # param2 = PX4_CUSTOM_MAIN_MODE_FLIP_RECOVERY
    0, 0, 0, 0, 0
)
```

### 方式 D：uORB 话题（ROS2 / px4_ros2）

```python
from px4_msgs.msg import VehicleCommand

msg = VehicleCommand()
msg.timestamp = int(self.get_clock().now().nanoseconds / 1000)
msg.command = 176            # VEHICLE_CMD_DO_SET_MODE
msg.param1 = 1.0             # CUSTOM_MODE_ENABLED
msg.param2 = 12.0            # PX4_CUSTOM_MAIN_MODE_FLIP_RECOVERY
msg.param3 = 0.0
msg.target_system = 1
msg.target_component = 1
msg.from_external = True
self.publisher.publish(msg)
```

更直接（跳过 custom_mode 换算）：

```python
msg.command = 10071          # VEHICLE_CMD_SET_NAV_STATE
msg.param1 = 16.0            # NAVIGATION_STATE_FLIP_RECOVERY
```

### 方式 E：命令行 / MAVLink Console

```
commander mode flip
```

## 4. 观测与退出

```
listener flip_recovery_control      # 倾角、zone、掩码、逐路油门、状态机
flip_recovery status                # 一次打印全部内部状态
```

正常会看到：

```
flip state 1: ramping up (tilt 121.3)
flip zone 2 (tilt 121): HIGH(m0c) reversed @ 0.20 [lifts] | LOW(m03) reversed @ 0.10
flip state 3: tilt below stop angle, gravity takes over
flip done, switching back to mode 2
```

退出方式：拨杆拨回 / 发 `SET_MODE` / 翻完自动切回 `FLIP_EXIT_MODE` / `FLIP_EN=0`。

## 5. 尚未完成：DShot 出力层

```
flip_recovery → flip_recovery_control.control[4]
                        │
                        └─✗ 没有消费方
```

两个阻塞点：

1. **未解锁不输出**：`MixingOutput::output_limit_calc(armed=false)` 走 `OutputLimitState::OFF`，
   把所有通道强制写成 `_disarmed_value`（DShot 为 0）。
2. **没有反向通道**：DShot 常规模式只有单向油门。Bluejay **不支持 3D 油门**，
   只能走方向命令 20/21，需实测。

详见 [`docs/flip-recovery/README.md` 第 8 节](docs/flip-recovery/README.md)。

---

# 这个分支改了什么

## 新增

| 文件 | 内容 |
|---|---|
| `msg/FlipRecoveryControl.msg` | 控制请求：`state` / `action` / `zone` / 掩码 / `float32[4] control` / `tilt_deg` |
| `src/lib/flip_recovery/FlipPlanner.hpp` | 纯逻辑：倾角、选边、方向映射、状态机、逐路油门 |
| `src/modules/flip_recovery/` | 模块骨架 + 14 个参数 + 构建配置 |
| `docs/flip-recovery/` | 完整文档 |

## 修改

| 文件 | 改了什么 | 为什么 |
|---|---|---|
| `msg/versioned/VehicleStatus.msg` | `FREE1=16` → `FLIP_RECOVERY=16` | 复用空闲槽位，`MAX` 保持 31 |
| `commander/px4_custom_mode.h` | 加 `MAIN_MODE_FLIP_RECOVERY`（=12） | MAVLink `custom_mode` 字节 |
| `lib/modes/ui.hpp` | 有效模式位 + 显示名 | 不加位切换被判非法 |
| `ModeUtil/control_mode.cpp` | 该模式所有 flag 保持 false | 让 `control_allocator` 停止发布 `actuator_motors` |
| `ModeUtil/mode_requirements.cpp` | 加 `prevent_arming` 等 | 禁止带着翻正模式解锁 |
| `Commander.cpp` | `DO_SET_MODE` 分支 + CLI | 所有触发路径的汇聚点 |
| `commander/module.yaml` | `17: Flip Recovery` | QGC 下拉选项 |
| `manual_control/ManualControl.cpp` | `navStateFromParam` case 17 | 拨杆链路 |
| `boards/.../default.px4board` | `CONFIG_MODULES_FLIP_RECOVERY=y` | 编译进固件 |
| `boards/.../rc.board_extras` | 无条件启动模块 | 模式化后必须常驻 |

## 核心原理（三段话）

**选边**：倾斜时贴地的那两路电机，其连线就是翻正转轴。它们力臂≈0，只负责"抬一点"卸掉地面压力；
真正干活的是高位两路，力臂最大。

**方向**：`Normal`（出厂转向）产生的机身受力方向由 `R(2,2)=cos(tilt)` 决定——
正立时朝上，倒扣后朝下。所以只在 90° 处换算一次，其余全部用 `Push::Up/Down` 表达。

**策略**：`FLIP_MODE=0`（默认）在 60–90° 压高位回正、90–180° 抬高位翻过 180°，高位全程 Reversed；
`FLIP_MODE=1` 全程压高位，路径最短但 90° 要换向。

---

# 文档

| 文件 | 内容 |
|---|---|
| [`docs/flip-recovery/README.md`](docs/flip-recovery/README.md) | 完整实现说明：架构、数据流、算法推导、逐文件改动、调试流程 |
| [`docs/flip-recovery/IMPLEMENTATION.md`](docs/flip-recovery/IMPLEMENTATION.md) | 演进记录（每次设计变更与纠错） |

---

# 刷写注意

两个分支都可以只刷 app，bootloader 不动。QGC 刷 `.px4` 文件即可，
MAVLink 刷写不会覆盖 bootloader，也不会丢电调校准。
