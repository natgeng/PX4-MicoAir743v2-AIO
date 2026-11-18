# `feat/flip-recovery` — 反乌龟 / 侧翻自动纠正

> **状态：规划中，暂无实现代码。** 本分支目前只比 `fix/bdshot-micoair743v2-aio` 多一个文档：`docs/flip-recovery/README.md`。
>
> 分支关系：`v1.17.0` → `fix/bdshot-micoair743v2-aio`（BDShot 修复）→ **本分支**。
> 也就是说这里已经**包含** bdshot 修复，可以独立编译刷机，只是还不会翻正。

| 项目 | 值 |
|---|---|
| 基线 | `fix/bdshot-micoair743v2-aio`（= PX4 v1.17.0 + bdshot 修复） |
| 目标硬件 | MicoAir743v2-AIO-35A（target `micoair_h743-v2`，board_id 1179） |
| 相对基线的差异 | 仅 `docs/flip-recovery/README.md`（31 行） |
| 计划新增 | `src/modules/flip_recovery/`（模块 + 参数 + 状态机） |

---

## 1. 要做的两件事

| 能力 | 触发 | 说明 |
|---|---|---|
| **反乌龟**（Turtle / crash flip） | 飞行员开关 + 摇杆 | 倒扣时用对角两台电机反向出力，把机身翻正 |
| **侧翻纠正**（自控版） | 姿态超过阈值自动 | 无需摇杆，自动执行翻正序列，失败则停 |

两者的物理原理相同：绕一条着地对角线产生力矩。区别只在"谁来选方向"——人控制 or 自己判断。

## 2. 前置条件（不满足就别做）

1. 电调支持 **DShot 临时方向命令 20/21**（AM32 已验证；本板 Bluejay 0.19.2 **待实测**）；
2. 电机几何为 **Quad-X**，M1–M4 同属 TIM1（本板满足）；
3. 必须有的安全门限：**上锁才能进入**、油门为 0、姿态贴地门限、单次时长与总时长上限、退出立即恢复电机方向；
4. flash 余量：本板 app 已 1,903,424 B / 上限 1,966,080 B，**只剩 61 KB**，新模块必须精简。

## 3. 设计草案

预期新增：

```
src/modules/flip_recovery/
├── CMakeLists.txt
├── Kconfig
├── flip_recovery_params.c      # FLIP_*
├── FlipRecovery.cpp/.hpp        # 主循环 + 门限判断
└── FlipSequence.hpp             # 状态机：等待→反转→出力→滑行→恢复
```

状态机（草案）：

```
WAIT_REST ──┬─ 倒扣静止 + 开关使能 ──> REVERSE (发 DShot 21，5 组 burst 中间插 STOP)
            └─ 姿态不满足 ──> 停留
REVERSE ──> DRIVE (对角两路出力，TRTL_MAX 限幅，roll/pitch 或固定方向)
DRIVE ──> COAST (收油，靠惯性+地面接触继续翻)
COAST ──> RECOVER (发 DShot 20 恢复方向) ──> DONE / FAIL(超时/无进展)
```

参数草案（对齐业界命名）：

| 参数 | 含义 | 默认 |
|---|---|---|
| `FLIP_EN` | 总开关（需重启） | 0 |
| `FLIP_AUX` | 触发通道 AUX1–6 | 1 |
| `FLIP_MAX_THR` | 出力上限（DShot 油门比例） | 0.20 |
| `FLIP_TIME` | 单次最长出力 | 2.5 s |
| `FLIP_NO_PROG` | 无进展超时 | 1.2 s |
| `FLIP_ANGLE` | 判定"已翻正"的剩余倾角 | 25° |

## 4. 参考实现

| 项目 | 特点 | 可借鉴点 |
|---|---|---|
| [PX4 PR #29750](https://github.com/PX4/PX4-Autopilot/pull/29750) | 最原生（commander / control_allocator / dshot 全链路），手控 | 官方架构怎么接；`MotorDirectionRequest/Status` 消息设计 |
| [LLJ5/PX4_Turtle_Mico](https://github.com/LLJ5/PX4_Turtle_Mico) | **自动右翻**，基线正好是 v1.17.0 / `d6f12ad1c4` | 自动判定与状态机、四元数判正反面 |
| [BENSHLOMO/px4-fpv-extensions](https://github.com/BENSHLOMO/px4-fpv-extensions) | 21 个 `TRTL_*` 参数可配，MicoAir H743 V2 验证过 | 参数化程度、电机映射可配 |
| [ArduPilot mode_turtle.cpp](https://github.com/ArduPilot/ardupilot/blob/master/ArduCopter/mode_turtle.cpp) | 原生模式，面向所有 DShot 多旋翼 | `enabled()` 判定、方向命令下发方式 |

## 5. 计划步骤

- [ ] **实测本板电调是否响应 DShot 20/21**（拆桨；先 `dshot turtle_reverse` 再手动确认转向）
- [ ] 选定基座：cherry-pick PR #29750，或移植 LLJ5 的自动版（倾向后者：同基线、已自动）
- [ ] 建模块骨架 + 参数，先只做"上锁 + 开关 + 姿态判定"，不输出
- [ ] 接入 DShot 方向命令（burst + STOP 序列 + 状态回报）
- [ ] 混控：只驱动对角两路，绕过 `control_allocator` 的常规路径
- [ ] 台架：拆桨验证方向与时限 → 垫软物倒扣 10% 推力 → 带桨低推力 → 连续 10 次 → 翻正后正常起飞
- [ ] 全流程拉 `logger` 日志，`replay` 复核无越界输出

## 6. 构建

```bash
git clone --recursive -b feat/flip-recovery \
    https://github.com/natgeng/PX4-MicoAir743v2-AIO.git
cd PX4-MicoAir743v2-AIO
bash ./Tools/setup/ubuntu.sh
make micoair_h743-v2_default -j$(nproc)
```

当前编译产物与 `fix/bdshot-micoair743v2-aio` **行为完全一致**（只多了文档），可以安全刷机验证基线功能。

## 7. 其他分支

| 分支 | 内容 |
|---|---|
| `fix/bdshot-micoair743v2-aio` | BDShot 修复（本分支的基线） |
| `main` | 与上游 `PX4/PX4-Autopilot` 保持一致 |

## 许可

基于 PX4-Autopilot，遵循 **BSD 3-Clause**（见 `LICENSE`）。

---

个人开发存档，非官方发布。翻正涉及电机反向出力，**务必拆桨验证**后再上桨测试。
