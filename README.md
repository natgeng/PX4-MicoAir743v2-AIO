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
| 电调换向 | **4/4 电机支持 DShot 方向命令 20/21**（2026-11-19 实测，见第一节第 8 小节） |

> 注意：45A 版是 **AM32**（32-bit），电调能力和本分支的假设不同。
> 也不要把第一代 `MicoAir743-AIO-35A`（非 v2，AM32）混进来。

---

# 分支导航

| 分支 | 内容 | 状态 |
|---|---|---|
| [`fix/bdshot-micoair743v2-aio`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/tree/fix/bdshot-micoair743v2-aio) | 双向 DShot 修复 —— 让本板的 RPM 遥测能用 | 可用 |
| [`feat/flip-recovery`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/tree/feat/flip-recovery) | 侧翻 / 倒扣自动翻正 —— 新增一个飞行模式 | 模式/算法/电调换向已验证，驱动时序待接入 |

两个分支各自独立编译，都**已包含** `bdshot` 修复。

```
v1.17.0
  └── fix/bdshot-micoair743v2-aio   （双向 DShot 修复）
        └── feat/flip-recovery      （+ 侧翻自动纠正）
```

---

# 一、侧翻自动纠正 flip_recovery

> 完整文档：[`docs/flip-recovery/README.md`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/blob/feat/flip-recovery/docs/flip-recovery/README.md)
> 实测报告：[`docs/flip-recovery/TEST-REPORT.md`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/blob/feat/flip-recovery/docs/flip-recovery/TEST-REPORT.md)

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

### 3. 触发

四种方式，最终都落到 `vehicle_status.nav_state = 16`。

#### A. QGC 参数页（不用写代码）

1. QGC 左侧 **齿轮 -> Parameters**，搜索 `COM_FLTMODE`
2. 把 `COM_FLTMODE5` 设成 **Flip Recovery**，点 Save
3. 有遥控就拨到第 5 档；没遥控就走 **MAVLink Console** 输入 `commander mode flip`

> 下拉里能出现 "Flip Recovery"，是因为固件在 `commander/module.yaml` 里把这个枚举描述成
> `17: Flip Recovery`。
>
> **QGC 的 "Flight Modes" 页面里没有这个模式** —— 那个页面的列表是 QGC 硬编码的，
> 必须走 **Parameters** 页面。

#### B. 遥控器拨杆

先在 QGC -> **Vehicle Setup -> Radio** 完成遥控器和开关校准，然后：

| 开关位置 | 对应参数 |
|---|---|
| 第 1 档 | `COM_FLTMODE1` |
| 第 2 档 | `COM_FLTMODE2` |
| ... | ... |
| 第 6 档 | `COM_FLTMODE6` |

把想用的那一档设成 **Flip Recovery**，拨过去即可；拨回任意其它档立即收油。

可选二次确认（防误碰）：`FLIP_RC_CONFIRM = 6` 要求 aux6 通道同时在高位。

#### C. ROS2 话题（和发降落一样简单）

```python
from px4_msgs.msg import VehicleCommand

cmd = VehicleCommand()
cmd.timestamp        = int(self.get_clock().now().nanoseconds / 1000)
cmd.command          = 10071     # VEHICLE_CMD_SET_NAV_STATE
cmd.param1           = 16.0      # NAVIGATION_STATE_FLIP_RECOVERY
cmd.target_system    = 1
cmd.target_component = 1
cmd.from_external    = True
self.cmd_pub.publish(cmd)
```

对比发降落只差一行：那边是 `cmd.command = VehicleCommand.VEHICLE_CMD_NAV_LAND`。
`10071` 直接给 `nav_state` 数值，不用换算 `custom_mode` 位域。

#### D. 板载命令行

```
commander mode flip
```

### 4. 执行结果回传

翻正结束（成功或失败）后模块持续发布 `flip_recovery_status`，上位机订阅它就能发下一步：

```python
from px4_msgs.msg import FlipRecoveryStatus

def status_cb(self, msg):
    if msg.result == FlipRecoveryStatus.RESULT_SUCCESS:
        # 用时 msg.duration_s，结束倾角 msg.tilt_deg
        # 下一步：解锁 / 起飞 / 切回 Position
        pass
    elif msg.result == FlipRecoveryStatus.RESULT_FAILED:
        # msg.reason: 1=TIMEOUT 2=NO_PROGRESS 3=SAFETY_LOST 4=MODE_EXITED 5=ATTITUDE_RESET
        pass
```

`result` 会一直保持到最后一次结果，直到下次开始出力才回到 `RESULT_NONE`，
所以上位机要按**跳变**去重，或者用 `session_id` 区分不同的尝试。

### 5. 观测与退出

```
listener flip_recovery_control      # 倾角、zone、掩码、逐路油门
listener flip_recovery_status       # 执行结果、时长、失败原因
flip_recovery status                # 全部内部状态
```

退出：拨杆拨回 / 在 QGC 切到其它模式 / 翻完自动切回 `FLIP_EXIT_MODE` / `FLIP_EN=0`。

### 6. 安全约束（已内置）

| 约束 | 实现 |
|---|---|
| 该模式下禁止解锁 | `mode_req_prevent_arming` |
| 只在未解锁 + 已落地时出力 | 模块内 `_safe` 判定 |
| 出力超时 | `FLIP_TIME`，默认 3 s |
| 方向配反自动停 | 0.8 s 内倾角没减小就 `Failed` |
| 失败后必须退出模式才能重试 | `_rearm_needed` |

### 7. 当前限制

- **桨能转，但方向只有出厂方向** —— DShot 反向的命令通路已验证（见下），
  但驱动层的「停机 → 发方向命令 → 再给油门」时序还没接进 `DShot.cpp`。
- 只支持 Quad-X（`CA_ROTOR_COUNT = 4`）。

### 8. 电调换向：已实测确认（2026-11-19）

在这块板子（v2-AIO-35A，Bluejay 0.19.2）上拆桨实测，**4/4 电机都能反向并立即恢复**：

| 电机 | 正常方向 | `dshot reverse` 后 | 正常转向（俯视） |
|---|---|---|---|
| 1 | +233 | −267 | 逆时针 |
| 2 | +271 | −328 | 逆时针 |
| 3 | −349 | +275 | 顺时针 |
| 4 | −290 | +202 | 顺时针 |

数值是无桨时起步的机身反作用角速度 `zgyro`（mrad/s），符号即转向——无桨电机肉眼难辨，
BSHOT 遥测关闭，所以用机身反力矩做客观测量。方法、原始波形和两种失败的测量方案见
[测试报告](https://github.com/natgeng/PX4-MicoAir743v2-AIO/blob/feat/flip-recovery/docs/flip-recovery/TEST-REPORT.md)。

顺带确认：**电机编号与 PX4 标准 Quad-X 一致**（1/2 同向、3/4 同向且相反），
翻正算法用 `CA_ROTORn_PX/PY` 选边不会错位。

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
