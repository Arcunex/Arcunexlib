# Towards Professional Tennis Styles for Humanoid Robots with Adaptive Motion Planning and Tracking

## 基本信息

| 项目 | 内容 |
|---|---|
| 简称 | AdaPT |
| 会议/年份 | CoRL 2026 |
| 作者 | Tao Huang, Ruofei Liu, Xuchen Tang 等 |
| 任务 | 具备职业选手风格的网球回合与发球 |
| 机器人 | Unitree G1、Dobot Atom P3 |
| 论文 | [arXiv:2608.20087](https://arxiv.org/abs/2608.20087) |
| 项目页 | [AdaPT](https://humanoidtennis.github.io/AdaPT/) |
| 代码 | [noitom-robotics/AdaPT](https://github.com/noitom-robotics/AdaPT) |
| 本地论文 | [paper.pdf](./paper.pdf) |

## 论文解决什么问题

现有方法往往能击中球，却难以稳定保留职业运动员的完整动作风格。纯规划-跟踪解耦虽然有利于动作风格，但真机 tracker 的误差、规划器的自回归误差和感知噪声会相互累积。AdaPT 的目标是在保留 Federer、Nadal、Djokovic 等选手风格的同时，通过显式的执行速度自适应减轻 sim-to-real 漂移。

## 数据链路

1. 从转播视频截取约 2 秒、包含完整击球和步法的片段。
2. 用 GVHMR 从单目视频恢复 SMPL 世界坐标动作，再用 GMR 重定向至机器人。
3. 针对遮挡造成的腕部误差进行风格相关修正，并添加随机腕部扰动。
4. 使用通用 motion tracker 将重定向结果校正为物理可执行轨迹。
5. 标注选手身份、正反手/发球、旋转类型、击球时刻与抛球时刻。

数据覆盖 3 位职业选手和 1 套专业 MoCap 风格；项目页给出的总时长为 21.5 小时。

## 方法链路

### 回合 Rally

- 在物理校正后的动作上训练 **MVAE 自回归运动生成器**，潜变量控制下一帧运动状态，并增加击球类型和旋转类型辅助预测。
- 高层 planner 读取机器人状态、位姿和未来球轨迹，输出 MVAE latent `z_t` 与速度系数 `α_t`。
- 低层 tracker 跟踪 MVAE 生成的参考动作。
- `α_t` 通过相邻参考插值/外推改变动作推进速度，使挥拍相位对齐来球。

### 发球 Serve

- 发球动作多样性较低且主要是自驱动过程，因此不再额外训练 MVAE。
- planner 根据机器人、球和上一动作输出速度系数。
- residual tracker 在参考跟踪动作上增加残差，以适应真实抛球偏差。
- phase mask 将强任务修正集中在关键击球阶段，降低风格被任务奖励破坏的程度。

### 自适应机制

Tracker 训练时随机采样动作执行速度，使低层具备“快一点或慢一点仍能稳住”的能力；planner 再学习选择适合球轨迹和当前跟踪能力的速度。两者分别处理执行误差和时序误差。

## 关键结果

- 真机 rally 中完整 AdaPT 通常显著优于 Vid2Player3D。例如 Federer 风格正/反手命中率由 **24% / 8%** 提升至 **64% / 48%**。
- 真机发球中，AdaPT 在三种风格上的成功率为 **66.7%、73.3%、86.7%**。
- 消融显示 adaptive tracker 与 adaptive planner 具有互补性：前者减少低层执行误差，后者修正击球时序与长期漂移。
- 论文明确指出一个重要权衡：解耦规划-跟踪更容易保持风格，但依赖更长、更准的未来球轨迹；LATENT/PULSE 式紧耦合方案反应更直接，但通常牺牲部分动作风格。

## 简单评价

**优势**：风格来源清晰且可控；将“生成什么动作”和“如何在真机执行”分离；速度成为规划器与 tracker 之间显式、可解释的适配接口；同时覆盖 rally、serve、多种数据源和两种机器人本体。

**局限**：系统链路较长，视频恢复、重定向、物理校正、MVAE、tracker 和 planner 的误差可能级联；解耦架构依赖未来球轨迹预测；论文完整能力尚未全部开源。截至建档时，官方仓库只释放了 **Stage 1 自适应发球跟踪**代码和预训练 checkpoint。

## 阅读结论

AdaPT 的主要价值不是单一 tracker，而是把**职业风格的运动生成**交给运动模型，把**物理执行**交给 tracker，再让速度变量贯穿两层来修复真机时序漂移。它更偏向**职业风格保真和层级式可控规划**。
