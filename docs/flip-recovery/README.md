# 侧翻自动纠正 flip_recovery

> PX4 v1.17.0 · MicoAir743v2-AIO-35A
> 一个原生飞行模式：落地侧翻或倒扣后，按当前姿态自动选出高位/低位电机，把机身翻回正立。

**当前状态**：模式接入、姿态算法、参数体系、状态观测全部完成，可直接编译刷写并验证；
**DShot 出力层尚未实现**（见文末第 8 节），所以现在刷进去只能观测，桨不会转。

---

# 使用说明（必读）

## 1. 先满足前置条件

| 条件 | 要求 |
|---|---|
| 机架 | Quad-X（4 电机），`CA_ROTOR_COUNT = 4` |
| 电调 | 支持 DShot，且能执行**方向命令 20/21**。本板 v2-AIO-35A 是 Bluejay 0.19.2，官方把命令 21 归为 turtle mode 功能，支持；45A 版是 AM32，另外还支持 3D 油门 |
| 姿态估计 | EKF2 正常输出 `vehicle_attitude`（静止时即可，不需要 GPS） |
| 遥控 | 可选。纯上位机触发时可以不接 |
| 机身状态 | **未解锁**、**已落地**、油门摇杆最低 |

## 2. 编译与刷写

```bash
make micoair_h743-v2_default
```

刷写时只刷 app，**不要动 bootloader**：

```
QGC → 固件设置 → 选择 .px4 文件 → 勾选 "仅刷写飞控固件"
或
px4 upload --port /dev/ttyACM0 build/micoair_h743-v2_default/micoair_h743-v2_default.px4
```

## 3. 参数配置

### 必须先设的两项

```
param set FLIP_EN 1            # 总开关，0 = 拨进模式也不转桨
param set COM_FLTMODE5 17      # 把第 5 档模式开关映射成 Flip Recovery
param save
```

> `COM_FLTMODE1..6` 是模式槽位，改成 17 的那一档就变成"翻正模式"。
> 不用拨杆的话这一项可以跳过，直接发指令。

### 常用可调参数

| 参数 | 默认 | 范围 | 作用 |
|---|---|---|---|
| **`FLIP_EN`** | 0 | bool | **总开关**。0 时模式可进、状态可见，但不转桨 |
| **`FLIP_THR_HIGH`** | 0.20 | 0.02–0.6 | **高位组油门**。唯一真正决定翻正快慢的参数，从 0.15 起步 |
| **`FLIP_THR_LOW`** | 0.10 | 0–0.6 | **低位组油门**。只负责卸压/减摩擦，正常应明显低于高位 |
| **`FLIP_LOW_MODE`** | 0 | 0–3 | 低位转向调试：`0`=Auto（永远抬）、`1`=强制正转、`2`=强制反转、`3`=关闭 |
| **`FLIP_MODE`** | 0 | 0–1 | 翻正策略：`0`=翻过去（90–180° 抬高位翻过 180°）、`1`=压回去（全程压高位，路径最短） |
| `FLIP_ENTRY_ANG` | 75 | 60–180 | 超过这个倾角才出力 |
| `FLIP_STOP_ANG` | 60 | 0–120 | 低于这个倾角收油，交给重力 |
| `FLIP_EXIT_MODE` | 2 | 0–8 | 翻完自动切回的模式（2 = Position） |
| `FLIP_RC_CONFIRM` | 0 | 0–6 | 0 = 免二次确认；1–6 = 还要对应 aux 通道在高位 |
| `FLIP_SLEW` | 0.50 | 0.05–2.0 | 油门上升率（1/s），越小越温和 |
| `FLIP_TIME` | 3.0 | 0.2–5.0 | 总出力时长上限（s） |
| `FLIP_MIN_PROG` | 3.0 | 0–30 | 进度检查阈值（度） |
| `FLIP_PROG_T` | 0.80 | 0.2–2.0 | 多久后开始查进度（s） |
| `FLIP_OUT_DIRECT` | 0 | bool | 1 = 台架直发 `actuator_motors` |

### 典型配置组合

```
# 保守起步（推荐第一次）
param set FLIP_THR_HIGH 0.15
param set FLIP_THR_LOW  0.08
param set FLIP_LOW_MODE 0
param set FLIP_MODE     0

# 只让高位转，验证选边
param set FLIP_LOW_MODE 3

# 怀疑低位在拖后腿
param set FLIP_LOW_MODE 1     # 强制正转，看是不是翻不动
param set FLIP_LOW_MODE 2     # 强制反转
```

## 4. 触发方式（任选一种）

四条路径最终都落到同一个 `vehicle_status.nav_state = 16 (NAVIGATION_STATE_FLIP_RECOVERY)`。

### 方式 A：遥控器拨杆

上面 `COM_FLTMODE5 = 17` 设好后，把模式开关拨到第 5 档即可。

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
    0,      # param3 = sub_mode
    0, 0, 0, 0
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
msg.source_system = 1
msg.source_component = 1
msg.from_external = True
self.publisher.publish(msg)
```

更直接的写法（跳过 custom_mode 换算）：

```python
msg.command = 10071          # VEHICLE_CMD_SET_NAV_STATE
msg.param1 = 16.0            # NAVIGATION_STATE_FLIP_RECOVERY
```

### 方式 E：命令行（台架/MAVLink Console）

```
commander mode flip
```

## 5. 退出

| 方式 | 结果 |
|---|---|
| 拨杆拨回 / 发 `SET_MODE` 到任意其它模式 | 下一周期立即收油 |
| 翻正完成或失败 | 自动切回 `FLIP_EXIT_MODE`（默认 Position） |
| `param set FLIP_EN 0` | 立即停 |

## 6. 观测

```
listener flip_recovery_control      # 倾角、zone、掩码、逐路油门、状态机
flip_recovery status                # 一次打印全部内部状态
listener vehicle_status             # 确认 nav_state 是否 = 16
```

正常会看到：

```
flip state 1: ramping up (tilt 121.3)
flip zone 2 (tilt 121): HIGH(m0c) reversed @ 0.20 [lifts] | LOW(m03) reversed @ 0.10
flip state 3: tilt below stop angle, gravity takes over
flip done, switching back to mode 2
```

`[lifts]` = 高位正在把机身抬起来翻过去，`[presses]` = 高位在压回去。

## 7. 安全约束（已内置，不需要你配）

| 约束 | 实现 |
|---|---|
| 该模式下**禁止解锁** | `mode_req_prevent_arming` |
| 只在未解锁时出力 | `_safe` 里检查 `!armed` |
| 必须已落地 | `vehicle_land_detected.landed` |
| 有遥控时油门必须在最低位 | `manual_control_setpoint.throttle < -0.8` |
| 出力超时保护 | `FLIP_TIME`，默认 3 s |
| 方向配反自动停 | `FLIP_PROG_T` 内倾角没减小就 `Failed` |
| 本次失败后必须退出模式才能重试 | `_rearm_needed` |
| 姿态被重置（切换估计器）立即放弃 | `quat_reset_counter` 变化检测 |

---

# 实现原理

## 1. 它是什么

| 是 | 不是 |
|---|---|
| 一个正式的 `nav_state`，和 Position/Acro 平级 | 一个独立后台进程（没有模式时不参与控制） |
| 只在未解锁 + 已落地时出力 | 飞行中的姿态保护 |
| 靠两路电机推力差的**力矩**翻正 | 靠反扭矩（yaw）翻正 |
| 翻完自动切回常规模式 | 翻完留在原地等指令 |

进入该模式后，`vehicle_control_mode` 里所有 `flag_control_*` 都是 `false`，
`flight_mode_manager`、`mc_att_control`、`mc_pos_control` 全部不启动，
`control_allocator` 也不再发布 `actuator_motors`——执行器写入权完整地让给了本模块。

## 2. 架构

```
src/modules/flip_recovery/              ← 模块外壳：uORB 订阅/发布、安全判定、参数
        │
        ▼
src/lib/flip_recovery/FlipPlanner.hpp   ← 纯逻辑：姿态数学、选边、方向映射、状态机
```

拆成两层的理由：算法不依赖 uORB，可以脱离飞控跑数值验证。
调度用 `ScheduledWorkItem` 挂在 `lp_default` 工作队列，20 ms 周期，不占独立线程。

## 3. 数据流

```
BMI088 / BMI270（SPI, 1 kHz）
   │
   ▼
sensors ── 校准、温补、低通
   │
   ▼
vehicle_imu ── 多 IMU 聚合、时间对齐
   │
   ├─► EKF2 ── 陀螺积分预测 + 加速度计重力观测 + 磁力计航向
   │      └─► vehicle_attitude.q（四元数，机体系 → NED）
   │
   └─► vehicle_angular_velocity
   │
   ▼
flip_recovery（20 ms）
   │
   ▼
flip_recovery_control（逐路带符号油门）
   │
   ▼
DShot 驱动（待实现）→ 电调 → 电机
```

**为什么不用原始加速度计直接算倾角**：翻正时机身在转、有振动，`atan2` 出来会抖十几度。
EKF2 已融合陀螺和加速度，输出平滑且延迟只有 5~10 ms。

## 4. 算法

### 4.1 倾角：为什么用 `R(2,2)`

```cpp
alignment = Dcmf(q)(2, 2);          // 机体 +Z 在世界天向上的投影
tilt      = acos(constrain(alignment, -1, 1));
```

| 姿态 | `R(2,2)` | `tilt` |
|---|---|---|
| 正立 | `+1` | 0° |
| 侧躺 | `0` | 90° |
| 完全倒扣 | `−1` | 180° |

不用欧拉角的三个理由：

1. **无万向锁** —— pitch = ±90° 时 roll 无定义，而 90° 恰好是分界点
2. **无组合歧义** —— `pitch=90° + roll=90°` 的总倾角不是 127°
3. **一次三角运算**，对 0°/180° 数值稳定

### 4.2 选边：高位与低位

```cpp
h[i] = R(2,0) * px[i] + R(2,1) * py[i];    // 电机相对重心的世界竖直高度
```

`px/py` 来自 PX4 自己的 `CA_ROTORn_PX/PY`，所以**电机号天然与 PX4 一致**，不需要另建映射表。

按 `h` 排序，最小的两路 = **低位组**，剩下 = **高位组**。

**低位组的连线就是翻正转轴**（机身倾斜时贴地的那条棱）：

```
        高位组（离转轴最远，力臂最大）
       ●───────────●
       │           │
       │     CG    │
       ●───────────●
     ═════════════════════   ← 地面
        低位组（就在转轴上，力臂 ≈ 0）
```

| 组 | 力臂 | 职责 |
|---|---|---|
| **高位** | 最大 | 唯一产生翻正力矩，全额油门 |
| **低位** | ≈ 0 | 只负责"抬一点"，卸掉地面正压力、减小绕轴摩擦 |

**低位为什么不能压**：压下去 → 正压力增大 → 摩擦锁住转轴 → 高位转不动。

### 4.3 方向映射：三个概念必须分清

| 概念 | 含义 | 参照系 |
|---|---|---|
| **桨对空气** | 螺旋桨把空气往哪甩 | 机体 ±Z |
| **机身受力 `Push`** | 空气对机身的反作用力，永远与"桨对空气"相反 | 世界系 |
| **电机转向** | `Normal` 出厂 / `Reversed` 反着转 | 机体系 |

`Normal` 的定义基准：正常飞行（tilt=0）时桨朝下甩空气 → 机身获得向上的升力。

| 倾角 | `R(2,2)` | 机体 −Z 指向 | `Normal` 桨对空气 | `Normal` 机身受力 |
|---|---|---|---|---|
| 60–90° | `> 0` | 朝下偏侧 | 朝下 | **朝上** |
| 90–180° | `< 0` | 朝上偏侧 | **朝上** | **朝下** |

代码里只用一行统一换算，绝不手写区间 `if/else`：

```cpp
const bool normal_is_up = (R(2, 2) > 0.f);
const bool high_normal  = ((high_push == Push::Up) == normal_is_up);
```

### 4.4 两种策略

**策略 A「翻过去」`FLIP_MODE=0`（默认）**

| 倾角 | 高位目标 | 高位电机 | 低位目标 | 低位电机 | 效果 |
|---|---|---|---|---|---|
| 60–90° | `Push::Down`（压） | Reversed | `Push::Up`（抬） | Normal | tilt 减小，回正 |
| 90–180° | `Push::Up`（抬） | Reversed | `Push::Up`（抬） | Reversed | tilt 增大，翻过 180° |

高位全程 Reversed，只有低位在 90° 换向。

**策略 B「压回去」`FLIP_MODE=1`**

| 倾角 | 高位 | 低位 | 效果 |
|---|---|---|---|
| 60–180° | `Push::Down` | `Push::Up` | 始终朝 0° 拉 |

代价：90–120° 段策略 A 要转 240–270°，策略 B 只要 90–120°；但策略 B 高位要在 90° 换向。

### 4.5 状态机

```
        ┌───────────────────────────────────────────────┐
   Idle ─┤ mode_active && safe && enabled && tilt≥entry  │
        ▼                                               │
      Ramp ──(油门到目标)──► Hold ──(tilt≤stop)──► Coast ──(稳定/1.5s)──► Done
        │                     │                                          │
        │                     └─(超时 / 无进度)──► Failed                │
        └─(安全丢失 / 模式退出)──────────────────► Failed                 │
                                                                         │
   Done / Failed 置 _rearm_needed，必须退出模式再进才能重试 ◄──────────────┘
```

三个退出条件：

| 条件 | 判据 | 作用 |
|---|---|---|
| 成功收油 | `tilt ≤ FLIP_STOP_ANG` | 已经足够正，交给重力 |
| 超时 | 出力 > `FLIP_TIME` | 电机/电调过热保护 |
| 无进度 | `FLIP_PROG_T` 内倾角减少 < `FLIP_MIN_PROG` | 抓"选错边/方向配反/推力不够" |

第三条最关键：如果电机方向配反，倾角会**变大**，0.8 s 内就能发现并停机。

### 4.6 油门

```cpp
thr_high = min(FLIP_THR_HIGH, 0.60);
thr_low  = low_enabled ? min(FLIP_THR_LOW, 0.60) : 0;
_thr     = min(max(thr_high, thr_low), _thr + FLIP_SLEW * dt);   // 斜坡
control[i] = ±min(_thr, 该组目标油门);
```

- **斜坡**默认 0.5/s，0.4 s 到顶，避免电流尖峰
- **硬上限 0.60** 写死在 `HARD_MAX_THROTTLE`，参数填再大也超不过
- `_thr` 是全局进度，每组取 `min(_thr, 自己的目标)`

**为什么必须逐路输出**：90–180° 段高位和低位同为 Reversed，但油门不同（0.20 vs 0.10），
两个标量（"正转组油门/反转组油门"）表达不了，所以消息里是 `float32[4] control`。

---

# 代码改动清单

## 新增

| 文件 | 内容 |
|---|---|
| `msg/FlipRecoveryControl.msg` | 控制请求：`state` / `action` / `zone` / `fwd_mask` / `rev_mask` / `float32[4] control` / `tilt_deg` |
| `src/lib/flip_recovery/FlipPlanner.hpp` | 纯逻辑：倾角、选边、方向映射、状态机、逐路油门 |
| `src/modules/flip_recovery/FlipRecovery.hpp` | 模块类：订阅/发布、参数声明 |
| `src/modules/flip_recovery/FlipRecovery.cpp` | 主循环：取数 → 安全判定 → 调规划器 → 发布 → 收尾切模式 |
| `src/modules/flip_recovery/flip_recovery_params.c` | 14 个 `FLIP_*` 参数 |
| `src/modules/flip_recovery/CMakeLists.txt` / `Kconfig` | 构建与 `MODULES_FLIP_RECOVERY` 开关 |

## 修改

| 文件 | 改了什么 | 为什么 |
|---|---|---|
| `msg/versioned/VehicleStatus.msg` | `NAVIGATION_STATE_FREE1=16` → `NAVIGATION_STATE_FLIP_RECOVERY=16` | 复用空闲槽位，`MAX` 保持 31，位掩码/ROS2 消息/外部工具零影响 |
| `msg/CMakeLists.txt` | 注册 `FlipRecoveryControl.msg` | 不注册就不生成头文件 |
| `src/modules/commander/px4_custom_mode.h` | 加 `PX4_CUSTOM_MAIN_MODE_FLIP_RECOVERY`（=12）；`get_px4_custom_mode()` 加反查 | MAVLink 的 `custom_mode` 字节需要它；反查用于心跳上报 |
| `src/lib/modes/ui.hpp` | `getValidNavStates()` 加位；`nav_state_names[16]` = "Flip Recovery" | 不加位 → 拨杆切换被判非法；不改名 → OSD 显示 `16: UNUSED2` |
| `ModeUtil/control_mode.cpp` | 该模式 case 内**所有 flag 保持 false** | `ControlAllocator` 用 `flag_control_allocation_enabled` 决定是否发布 `actuator_motors`，留 false 就等于让位 |
| `ModeUtil/mode_requirements.cpp` | attitude + angular_velocity + manual_control + **prevent_arming** | 前三条保证算法有输入；最后一条保证不能带着翻正模式解锁 |
| `Commander.cpp` | `DO_SET_MODE` 解析分支；CLI `commander mode flip`；usage 文本 | 所有上位机路径和自动切回都汇到这个 switch |
| `commander/module.yaml` | `COM_FLTMODE` 枚举加 `17: Flip Recovery` | QGC 参数页下拉选项 |
| `manual_control/ManualControl.cpp` | `navStateFromParam` 加 `case 17` | PX4 1.16 起拨杆逻辑搬到 manual_control，漏这里拨杆失效 |
| `boards/.../default.px4board` | `CONFIG_MODULES_FLIP_RECOVERY=y` | 编译进固件 |
| `boards/.../init/rc.board_extras` | 改为无条件 `flip_recovery start` | 模式化后必须常驻，否则改参数要重启飞控才能用 |

### 让它"让位"的关键链路

```
Commander::updateControlMode() 先 _vehicle_control_mode = {}
  → control_mode.cpp 的 FLIP_RECOVERY case 不设任何 flag
  → ControlAllocator.cpp:372  _publish_controls = flag_control_allocation_enabled → false
  → ControlAllocator::publish_actuator_controls() 开头 if (!_publish_controls) return;
  → 分配器停止发布 actuator_motors，写入权归 flip_recovery
```

---

# 调试流程（拆桨）

1. **只观测**：`FLIP_EN=0`，`commander mode flip`，`listener flip_recovery_control` 看 `tilt_deg / zone / masks`。
   摆 80°、120°、180°，确认低位掩码选对。
2. **验模式链路**：拨杆切进切出，`listener vehicle_status` 看 `nav_state` 是否变 16。再试 CLI 和 MAVLink。
3. **验禁止解锁**：停在翻正模式，`commander arm` 应被拒，提示 "mode not suitable for arming"。
4. **验让位**：`listener actuator_motors`，翻正模式下应无新数据；切回 Position 恢复。
5. **单组转桨**：`FLIP_EN=1`、`FLIP_LOW_MODE=3`（只高位）、`FLIP_THR_HIGH=0.10`，垫软物摆 80°。
6. **加低位**：`FLIP_LOW_MODE=0`、`FLIP_THR_LOW=0.05`。
7. **对照**：`FLIP_LOW_MODE=1/2` 看低位正转/反转哪个更顺。
8. **带桨低推力**：`FLIP_THR_HIGH=0.15`、`FLIP_THR_LOW=0.08`，连续 10 次查温度。

---

# 8. DShot 出力层（进行中）

拆成两层：**能不能输出**（软件门控）和**输出什么方向**（电调能力）。

## 8.1 已完成：未解锁也允许出力

PX4 原本在 `MixingOutput` 里有三道门把未解锁时的输出全部抹成 0：

```cpp
// 1. 值被抹成 NaN
if (_armed.armed || (_armed.prearmed && _functions[i]->allowPrearmControl())) {
    outputs[i] = _functions[i]->value(...);
} else {
    outputs[i] = NAN;
}

// 2. 油门标志只有两种情况下为 true
_throttle_armed = (_armed.armed && !_armed.lockdown) || _armed.in_esc_calibration_mode;

// 3. 状态机强制归零
case OutputLimitState::OFF:
    for (int i = 0; i < num_channels; i++) {
        _current_output_value[i] = _disarmed_value[i];
    }
```

现在加了一条与 `actuator_test` **平行**的通路，完全绕开 `_functions[]`：

```cpp
void MixingOutput::setRecoveryOutputs(bool active, const float outputs[MAX_ACTUATORS]);

// update() 里，_actuator_test.update() 之后
if (_recovery_active && !_armed.armed && !_armed.kill
    && !_armed.lockdown && !_armed.termination) {
    _reversible_mask = 0;      // 不走 3D 编码

    for (int i = 0; i < _max_num_outputs; ++i) {
        const float value = _recovery_outputs[i];
        _current_output_value[i] = (PX4_ISFINITE(value) && value > 0.f)
                                   ? output_limit_calc_single(i, value)
                                   : _disarmed_value[i];
    }

    if (_interface.updateOutputs(_current_output_value, _max_num_outputs, has_updates)) {
        ... 发布 actuator_outputs ...
    }

    return true;
}
```

三个安全条件缺一不可：**调用方要求接管** + **确实未解锁** + **没有 kill/lockdown/termination**。
默认 `_recovery_active = false`，对其它机型、其它模式零影响。

调用方是 `DShot::handle_flip_recovery()`：

```cpp
void DShot::handle_flip_recovery()
{
    flip_recovery_control_s msg;
    const bool updated = _flip_recovery_sub.update(&msg);

    // 话题超时（模块退出/重启）-> 立即放手
    if (!updated) {
        if (_mixing_output.recoveryActive() && hrt_elapsed_time(&_last_flip_recovery) > 200_ms) {
            _mixing_output.setRecoveryOutputs(false, nullptr);
        }
        return;
    }

    _last_flip_recovery = msg.timestamp;

    const bool drive = (msg.action == flip_recovery_control_s::ACTION_DRIVE);

    float outputs[MAX_ACTUATORS] {};
    if (drive) {
        for (unsigned i = 0; i < 4; ++i) {
            const float control = msg.control[i];
            outputs[i] = (control > 0.f) ? control : -control;   // 取绝对值
        }
    }

    _mixing_output.setRecoveryOutputs(drive, outputs);
}
```

`200 ms` 超时是兜底：万一 `flip_recovery` 崩了或被杀，电机不会一直转。

## 8.2 未完成：反向

现在 `control[i]` 的**符号被丢弃**了，所有电机都按出厂方向转。要走完整翻正还需要：

1. **换向时机**：DShot 规范要求方向命令在电机**停转**时发送。
   `DShot::updateOutputs()` 里只有 `output == DSHOT_DISARM_VALUE(0)` 才会发命令帧，
   所以必须先停机、发命令、等电调处理、再给油门。
   参考 LLJ5 的做法：停机 500 ms → 连发 10 帧 → 3 s 超时。
2. **命令通道**：PX4 原生已有 `DShot_cmd_spin_direction_normal(20)` / `reversed(21)`，
   `_current_command` 机制可直接复用，但一次只能发一个 motor_mask。

建议的实现：

```cpp
// ACTION_DRIVE 的第一个阶段：油门保持 0，对 rev_mask 发一次 21，对 fwd_mask 发 20
if (drive && phase == Direction && !_current_command.valid()) {
    _current_command.command = msg.rev_mask ? DShot_cmd_spin_direction_reversed
                                            : DShot_cmd_spin_direction_normal;
    _current_command.motor_mask = msg.rev_mask;
    _current_command.num_repetitions = 10;
    phase = Throttle;
}

// 第二阶段才真正给油门
if (drive && phase == Throttle) { ... }

// 退出：对全部发一次 20，把方向复位
if (!drive && _direction_dirty) { ... }
```

## 8.3 测试方法

### 先测电调是否支持方向命令（不需要任何新代码）

PX4 原生就有这个 CLI：

```
dshot reverse -m 1      # 对 Motor1 发命令 21（反转），重复 10 帧
dshot normal  -m 1      # 发命令 20（恢复）
```

**拆桨**，执行后用手摸电机轴或听声音（`dshot beep1 -m 1` 也能辅助判断）。
如果转向真的反了，说明电调支持，8.2 可以做；如果没反应，再检查是不是命令发在了非停转状态。

**本板已知信息**（v2-AIO-35A = Bluejay 0.19.2）：Bluejay 的功能表把命令 21 明确列为
"Turtle mode, supported"，所以预期是支持的，这一步主要是确认电调固件版本没被改过。

### 35A 与 45A 的差别（影响实现路线）

| | v2-AIO-35A | v2-AIO-45A |
|---|---|---|
| 电调 | 4×35A **Bluejay** | 4×45A **AM32** |
| MCU | 8-bit SiLabs EFM8 | 32-bit AT32F421 |
| 3D 油门（负油门反转） | **不支持** | **支持** |
| 方向命令 20/21 | 支持 | 支持 |

所以本板只能走方向命令，`MixingOutput` 里清 `_reversible_mask` 是必须的。
注意别和老款 `MicoAir743-AIO-35A`（非 v2，AM32）搞混。

### 再测未解锁出力

```
param set FLIP_EN 1
commander mode flip
```

用 `listener flip_recovery_control` 确认模块在发 `ACTION_DRIVE`，
然后看电机是否在**未解锁**状态下按 `control[]` 的绝对值转起来。

**这一步只能验证"能不能转"，方向还不对**——因为符号被丢弃了。
正确做法是先按 8.3 第一段确认电调能力，再补 8.2。

# 9. Flash 占用

v1.17.0 基线在 `micoair_h743-v2` 上剩余约 61 KB。

| 项 | 估算 |
|---|---|
| `FlipRecovery.cpp` + 模块骨架 | +0.8 KB |
| `FlipPlanner.hpp`（模板/内联） | +0.4 KB |
| 新消息（含 `control[4]`） | +0.2 KB |
| Commander / ui.hpp / mode_requirements 分支 | +0.5 KB |
| 参数（14 个） | +0.6 KB |
| **合计** | **约 +2.5 KB** |

---

# 参考

| 项目 | 说明 |
|---|---|
| PX4 PR #29750 | 原生反乌龟方案（commander/allocator/dshot 全链路），手控触发 |
| [LLJ5/PX4_Turtle_Mico](https://github.com/LLJ5/PX4_Turtle_Mico) | 自动右翻，基线同为 v1.17.0 |
| [BENSHLOMO/px4-fpv-extensions](https://github.com/BENSHLOMO/px4-fpv-extensions) | 21 个 `TRTL_*` 参数，MicoAir H743 V2 验证过 |
| ArduPilot `mode_turtle.cpp` | 原生 turtle 模式 |
