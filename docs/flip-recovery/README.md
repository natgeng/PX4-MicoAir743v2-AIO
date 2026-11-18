# 反乌龟 / 侧翻自动纠正（Flip Recovery）—— 待开发

本分支只放规划，暂不含实现。代码后续加入 `src/modules/flip_recovery/`。

## 目标

1. **反乌龟（Turtle / crash flip）**：倒扣时用对角两台电机反向出力把机身翻正。
2. **侧翻纠正（自控版）**：检测到倾角超过阈値后自动执行翻正序列，无需摇杆。

## 前置条件

- 电调需支持 **DShot 临时方向命令 20/21**（AM32 已验证；本板 Bluejay 0.19.2 待实测）。
- 电机几何为 Quad-X，M1–M4 同属 TIM1（本板成立）。
- 必须有：上锁才能进入、油门为 0、姿态门限、总时长/单次时长上限、退出即恢复方向。

## 候选参考实现

| 项目 | 特点 | 链接 |
|---|---|---|
| PX4 PR #29750 | 最原生（commander/allocator/dshot 全链路），手控 | https://github.com/PX4/PX4-Autopilot/pull/29750 |
| LLJ5/PX4_Turtle_Mico | **自动右翻**，基线正好是 v1.17.0 / d6f12ad1c4 | https://github.com/LLJ5/PX4_Turtle_Mico |
| BENSHLOMO/px4-fpv-extensions | 21 个 TRTL_* 参数可配，MicoAir H743 V2 验证 | https://github.com/BENSHLOMO/px4-fpv-extensions |
| ArduPilot `mode_turtle.cpp` | 原生模式，面向所有 DShot 多旋翼 | https://github.com/ArduPilot/ardupilot/blob/master/ArduCopter/mode_turtle.cpp |

## 计划步骤

- [ ] 实测本板电调是否响应 DShot 20/21（拆桨，`dshot turtle_reverse` 后手动确认转向）
- [ ] 选定基座：cherry-pick PR #29750 或移植 LLJ5 的自动版
- [ ] 参数设计：`FLIP_EN` / `FLIP_MAX_THR` / `FLIP_TIME` / `FLIP_ANGLE`（对齐 TRTL_*）
- [ ] 状态机：等待 → 反转确认 → 出力 → 滑行 → 恢复方向 → 完成/失败
- [ ] 台架测试 → 带桨低推力 → 连续 10 次 → 翻正后正常起飞
