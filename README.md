# PX4 1.17.0 · MicoAir743v2-AIO-35A 二次开发

这是 [PX4-Autopilot](https://github.com/PX4/PX4-Autopilot) 的 fork，用来保存针对 **MicoAir743v2-AIO-35A**（微空 H743 v2 AIO 35A 一体板）的固件改动。

所有改动按分支管理，一条分支一件事，互不干扰。

| 项目 | 值 |
|---|---|
| 基线版本 | PX4 **v1.17.0** |
| 基线提交 | `d6f12ad1c4f70ad3230afd7d86e971421e02fef4` |
| 目标硬件 | MicoAir743v2-AIO-35A（PX4 target `micoair_h743-v2`，board_id **1179**） |
| 电调 | Bluejay 0.19.2，4×35A，DShot300 / DShot600 |

---

## 分支说明

| 分支 | 基线 | 内容 | 状态 |
|---|---|---|---|
| `fix/bdshot-micoair743v2-aio` | v1.17.0 | 双向 DShot（BDShot）转速回传修复 | 代码完成，待上机实测 |
| `feat/flip-recovery` | 上述分支 + 1 提交 | 反乌龟 / 侧翻自动纠正（目前只有规划文档） | 规划中 |
| `main` | 上游 `PX4/PX4-Autopilot` | 保持与上游一致，不放大改动 | 同步用 |

---

## 改动一：双向 DShot（BDShot）修复

### 问题现象

开启双向 DShot 后，**M3（PE11 / TIM1_CH2）丢失转速回传，甚至不转**；同一套硬件在 ArduPilot 下正常。

对应上游 issue：[PX4/PX4-Autopilot#29468](https://github.com/PX4/PX4-Autopilot/issues/29468)（同款板、同款症状）。

### 根因

双向 DShot 不是"发完就完"——飞控发完油门帧后，要**立刻把同一根信号线切成输入**，用定时器 + DMA 捕获电调回传的 GCR 编码窄脉冲（DShot300 下 bit0 只有约 1.25 µs 高电平）。

原代码里四路电机输出使用默认 GPIO 配置：**50 MHz 摆率、无上拉**。边沿太陡，加上 AIO 板上通往四合一电调的长走线与输入电容，回传脉冲边沿产生振铃 / 过冲，被捕获成错误宽度甚至判为非法帧 → eRPM 丢失、电调误判信号异常。

### 改动内容

只改两个文件的板级定时器表，把 **M1–M4（TIM1 CH4/3/2/1）** 的输出与捕获配置都改为 **25 MHz 摆率 + 内部上拉**：

```
boards/micoair/h743-v2/src/timer_config.cpp
boards/micoair/h743-aio/src/timer_config.cpp
```

```cpp
static constexpr timer_io_channels_t motorOutput(timer_io_channels_t channel)
{
	channel.gpio_out = (channel.gpio_out & ~GPIO_SPEED_MASK) | GPIO_SPEED_25MHz | GPIO_PULLUP;
	channel.gpio_in  = (channel.gpio_in  & ~GPIO_SPEED_MASK) | GPIO_SPEED_25MHz | GPIO_PULLUP;
	return channel;
}
```

对应的 `timer_io_channels` 表数值变化：

| 通道 | 引脚 | 官方 v1.17.0 | 本分支 |
|---|---|---|---|
| M1 | PE14 | `0x0008184E` | `0x0009144E` |
| M2 | PE13 | `0x0008184D` | `0x0009144D` |
| M3 | PE11 | `0x0008184B` | `0x0009144B` |
| M4 | PE9  | `0x00081849` | `0x00091449` |
| M5–M8 | PB1 / PB0 / PD12 / PD13 | `0x00082811/10/3C/3D` | **不变** |

### 为什么是 25 MHz + 上拉，而不是上游 PR 的 2 MHz

上游 [PR #29469](https://github.com/PX4/PX4-Autopilot/pull/29469) 的做法是直接降到 `GPIO_SPEED_2MHz`（不加拉），且只改了 `h743-v2`。

而微空官方发布的 `bdshot-fixed` 固件，用的是 **25 MHz + 内部上拉**。这个结论来自二进制比对：把官方 v1.17.0 release 的 `.px4` 解压出镜像，逐字节定位 `timer_io_channels` 表（每项 16 字节：`gpio_out, gpio_in, timer_index, timer_channel, masks, ccr_offset`），确认差异就在这两个位段。

本分支按厂商实测可用的配置复刻，而不是采用上游未经该板验证的 2 MHz 方案。

### 影响范围

- 只影响 **M1–M4**（TIM1 一组）；M5–M8 保持默认，行为不变。
- **DShot 位时序不变**：码率由定时器产生，与 GPIO 摆率无关，DShot300 / 600 均可照常使用。
- 只影响 `micoair_h743-v2` 与 `micoair_h743-aio` 两个 target，其他板不受影响。
- 为什么只有 4 路：PX4 的 BDShot 实现**只允许一个定时器**（Burst 模式每个 timer 占用一个 DMA 通道），M1–M4 同属 TIM1，因此只有它们能回传转速。

### 如何验证

1. 编译后检查二进制中该表值是否为 `0x0009144E/4D/4B/49`（可十六进制定位 `timer_io_channels` 表核对）。
2. 上机设 `DSHOT_BIDIR_EN=1`（需重启），用 `listener esc_status` 或 QGC 看四路 RPM 是否都有数值。
3. 拆桨、逐个电机 `actuator_test`，确认 M3 不再丢转速。

---

## 改动二：反乌龟 / 侧翻自动纠正（规划中）

分支 `feat/flip-recovery` 目前**只有规划文档** `docs/flip-recovery/README.md`，没有可运行代码。

计划内容：倒扣时用对角两台电机反向出力把机身翻正（反乌龟）；检测倾角后自动执行翻正序列（侧翻纠正）。前置条件是电调支持 DShot 临时方向命令 **20/21**。

参考实现：

| 项目 | 特点 |
|---|---|
| [PX4 PR #29750](https://github.com/PX4/PX4-Autopilot/pull/29750) | 最原生（commander / control_allocator / dshot 全链路），手控 |
| [LLJ5/PX4_Turtle_Mico](https://github.com/LLJ5/PX4_Turtle_Mico) | **自动右翻**，基线正好是 v1.17.0 / `d6f12ad1c4` |
| [BENSHLOMO/px4-fpv-extensions](https://github.com/BENSHLOMO/px4-fpv-extensions) | 21 个 `TRTL_*` 参数可配，MicoAir H743 V2 上验证过 |
| [ArduPilot mode_turtle.cpp](https://github.com/ArduPilot/ardupilot/blob/master/ArduCopter/mode_turtle.cpp) | 原生模式，面向所有 DShot 多旋翼 |

---

## 构建

```bash
# Linux / WSL / Ubuntu
git clone --recursive -b fix/bdshot-micoair743v2-aio \
    https://github.com/natgeng/PX4-Autopilot.git
cd PX4-Autopilot

bash ./Tools/setup/ubuntu.sh                 # 首次：安装工具链
make micoair_h743-v2_default -j$(nproc)      # 编译
```

产物：`build/micoair_h743-v2_default/micoair_h743-v2_default.px4`

> 子模块地址来自 `.gitmodules`（指向 PX4 官方 29 个子模块仓库），`--recursive` 不会从本 fork 拉取；国内建议先配代理。

## 刷写注意

- 刷 **`.px4`** 只写 app 区（`0x08020000` 起），**不会动 bootloader**。
- 只有 DFU 刷整片 2 MiB `.bin`、或 QGC「Flash Bootloader」才会覆盖 bootloader（本机原厂 BL 是 PX4 v1.15.4 重编）。
- 这块板的参数存在 **SD 卡** `/fs/microsd/params`（`CONFIG_BOARD_PARAM_FILE`），刷固件不会丢校准，但**必须插 SD 卡**。
- 不要把本 target 的固件刷到 `micoair_h743` / `micoair_h743-aio` 上（board_id 不同）。

## 变更记录

| 日期 | 分支 | 内容 |
|---|---|---|
| 2026-11-18 | `fix/bdshot-micoair743v2-aio` | bdshot GPIO 上拉 + 25 MHz 摆率（M1–M4），对齐厂商 `bdshot-fixed` 固件 |
| 2026-11-18 | `feat/flip-recovery` | 新建分支，仅规划文档 |

## 许可

基于 PX4-Autopilot，遵循 **BSD 3-Clause**（见 `LICENSE`）。本仓库改动同样以 BSD 3-Clause 发布。

## 免责声明

个人开发存档，非官方发布。刷机有风险：请先拆桨台架验证，再低推力试飞。因使用本仓库代码造成的任何损失由使用者自行承担。
