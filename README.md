# `fix/bdshot-micoair743v2-aio` — 双向 DShot（BDShot）修复

> 本分支 = **PX4 v1.17.0 + 1 个改动**：修复 MicoAir743v2-AIO-35A 上 BDShot 转速回传失效。
> 分支内容直接可编译可刷写；其他改动在别的分支，见文末。
>
> （说明：`README.md` 是**跟随分支**的文件，切到哪条分支就看到哪条的说明，不是全仓库唯一一份。）

| 项目 | 值 |
|---|---|
| 基线 | PX4 v1.17.0 / `d6f12ad1c4f70ad3230afd7d86e971421e02fef4` |
| 目标硬件 | MicoAir743v2-AIO-35A（target `micoair_h743-v2`，board_id **1179**） |
| 改动文件 | `boards/micoair/h743-v2/src/timer_config.cpp`、`boards/micoair/h743-aio/src/timer_config.cpp` |
| 状态 | 代码完成，**待上机实测** |

---

## 1. 问题现象

开启双向 DShot 后，**M3（PE11 / TIM1_CH2）丢失转速回传，甚至不转**；同一套硬件在 ArduPilot 下正常。

对应上游 issue：[PX4/PX4-Autopilot#29468](https://github.com/PX4/PX4-Autopilot/issues/29468)，同款板、同款症状。

## 2. 根因

双向 DShot 不是"发完就完"——飞控发完油门帧后要**立刻把同一根信号线切成输入**，用定时器 + DMA 捕获电调回传的 GCR 编码窄脉冲（DShot300 下 bit0 仅约 1.25 µs 高电平）。

原代码里四路电机输出用的是默认 GPIO 配置：**50 MHz 摆率、无上拉**。边沿过陡，加上 AIO 板上通往四合一电调的长走线与输入电容，回传脉冲产生振铃 / 过冲，被捕获成错误宽度甚至判为非法帧 → eRPM 丢失、电调误判信号异常。

## 3. 改动内容

把 **M1–M4（TIM1 CH4/3/2/1）** 的输出与捕获配置都改为 **25 MHz 摆率 + 内部上拉**：

```cpp
static constexpr timer_io_channels_t motorOutput(timer_io_channels_t channel)
{
	channel.gpio_out = (channel.gpio_out & ~GPIO_SPEED_MASK) | GPIO_SPEED_25MHz | GPIO_PULLUP;
	channel.gpio_in  = (channel.gpio_in  & ~GPIO_SPEED_MASK) | GPIO_SPEED_25MHz | GPIO_PULLUP;
	return channel;
}
```

`timer_io_channels` 表值变化：

| 通道 | 引脚 | 官方 v1.17.0 | 本分支 |
|---|---|---|---|
| M1 | PE14 | `0x0008184E` | `0x0009144E` |
| M2 | PE13 | `0x0008184D` | `0x0009144D` |
| M3 | PE11 | `0x0008184B` | `0x0009144B` |
| M4 | PE9  | `0x00081849` | `0x00091449` |
| M5–M8 | PB1 / PB0 / PD12 / PD13 | `0x00082811/10/3C/3D` | **不变** |

### 为什么是 25 MHz + 上拉，而不是上游的 2 MHz

上游 [PR #29469](https://github.com/PX4/PX4-Autopilot/pull/29469) 直接降到 `GPIO_SPEED_2MHz`（不加拉），且只改 `h743-v2`。而微空官方 `bdshot-fixed` 固件用的是 **25 MHz + 内部上拉**——这个结论来自二进制比对：解压官方 v1.17.0 release 的 `.px4` 镜像，逐字节定位 `timer_io_channels` 表（每项 16 字节：`gpio_out, gpio_in, timer_index, timer_channel, masks, ccr_offset`），差异就在这两个位段。

本分支按厂商实测可用的配置复刻，而不是采用上游未经该板验证的方案。

## 4. 影响范围

- 只影响 **M1–M4**（TIM1 一组）；M5–M8 保持默认。
- **DShot 位时序不变**：码率由定时器产生，与 GPIO 摆率无关，DShot300/600 照常可用。
- 只影响 `micoair_h743-v2` 与 `micoair_h743-aio` 两个 target。
- 为什么只有 4 路：PX4 的 BDShot 实现**只允许一个定时器**（Burst 模式每个 timer 占一个 DMA 通道），M1–M4 同属 TIM1，所以只有它们能回传转速。

## 5. 验证

1. 编译后核对二进制里表值是否为 `0x0009144E/4D/4B/49`；
2. 上机设 `DSHOT_BIDIR_EN=1`（需重启），看 QGC 或 `listener esc_status` 四路 RPM；
3. 拆桨逐个 `actuator_test`，确认 M3 不再丢转速。

## 6. 构建与刷写

```bash
git clone --recursive -b fix/bdshot-micoair743v2-aio \
    https://github.com/natgeng/PX4-MicoAir743v2-AIO.git
cd PX4-MicoAir743v2-AIO
bash ./Tools/setup/ubuntu.sh
make micoair_h743-v2_default -j$(nproc)
# 产物 build/micoair_h743-v2_default/micoair_h743-v2_default.px4
```

- 刷 `.px4`（QGC → Firmware → Custom firmware file）只写 app 区 `0x08020000` 起，**不动 bootloader**；
- 别用 CubeProgrammer 刷整片 2 MiB `.bin`（会覆盖 BL）；
- 参数存在 SD 卡 `/fs/microsd/params`，刷固件不丢校准，但必须插卡；
- 别把本 target 固件刷到 `micoair_h743` / `micoair_h743-aio`（board_id 不同）。

## 7. 其他分支

| 分支 | 内容 | 状态 |
|---|---|---|
| [`feat/flip-recovery`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/tree/feat/flip-recovery) | 反乌龟 / 侧翻自动纠正（本分支 + 新增飞行模式，含结果回传与[实测报告](https://github.com/natgeng/PX4-MicoAir743v2-AIO/blob/feat/flip-recovery/docs/flip-recovery/TEST-REPORT.md)） | 模式/算法/电调换向已验证，驱动时序待接入 |
| [`main`](https://github.com/natgeng/PX4-MicoAir743v2-AIO/tree/main) | 项目总览：两个分支的入口、参数与使用说明 | 上游 PX4 + 本项目 README |

分支关系：`v1.17.0` -> **本分支** -> `feat/flip-recovery`

## 8. 变更记录

| 日期 | 提交 | 内容 |
|---|---|---|
| 2026-11-18 | `78d168033c` | bdshot GPIO 上拉 + 25 MHz 摆率（M1–M4），对齐厂商 `bdshot-fixed` |

## 许可

基于 PX4-Autopilot，遵循 **BSD 3-Clause**（见 `LICENSE`）。

---

个人开发存档，非官方发布。刷机有风险，请先拆桨台架验证。
