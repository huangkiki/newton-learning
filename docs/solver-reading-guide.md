# 读懂 DexLab 的 Newton Physics 配置

[Sim Learning 共同概念](https://github.com/huangkiki/sim-learning/tree/main/docs/solver-guide) · [本仓课程](curriculum.md) · [版本边界](versions.md) · [迁移与静态核对](learning-migration.md)

面向已有 VLA、WAM 或 RL 经验的工程师、硕博读者：目标是把配置标签还原成可以追踪的计算路径，知道换了什么、该观察什么，而不是先背一串 solver 名。建议先沿总仓的 [Physics-Based Simulation 学习主线](https://github.com/huangkiki/sim-learning/blob/main/docs/physics-based-simulation.md) 建立离散化、能量与优化基础；教材中的 IPC 等方法不自动等于 Newton Physics 各后端的默认接触模型。

本文对应 DexLab 固定快照 [`7e4a077`](https://github.com/huangkiki/Dexlab/tree/7e4a0774efb4f8b332e866e1495e032b4969c8aa)，以其[角色词典](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/solver-taxonomy.json)为标签清单。原生实现按 Newton **1.6.1**、源码 [`713fecd`](https://github.com/newton-physics/newton/tree/713fecdc41caf0c9d726f5c016939f36e66e3dff) 解读。正文只有来源核对与纸面练习，没有原生导入、JIT、步进、训练或新增物理验收。

## 1. 先把一行配置读成一句话

`Newton Physics / Kamino / DVI / Euler` 可以读成：在 Newton Physics 引擎里，使用 Kamino 刚体系统，以 DVI 后端求动力学约束，再用指定的 Euler 路径推进状态。DVI 与 Euler 不是两个可以相互替换的 solver。引擎名中的 **Newton** 也不表示所有后端都使用数值优化中的 Newton method。

```text
Newton Physics 引擎
  ├─ 刚体/关节实现：Featherstone、SemiImplicit、XPBD、VBD、Kamino
  │    ├─ VBD：compliant ALM 或 legacy AVBD 分支
  │    └─ Kamino：DVI 或 PADMM 动力学后端；另选积分器
  ├─ 材料/柔性实现：Style3D、ImplicitMPM
  └─ 每个实际分支：自己的时间步、迭代预算、读回和能力边界
```

上图是阅读索引，不是互斥的功能分类：VBD 也处理粒子/布料/软体，SemiImplicit 与 Featherstone 也有粒子材料路径。是否支持某类关节、接触或梯度要继续查消费者，不能由公开基类相同推断等价。[官方入口与能力矩阵](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/docs/solvers/index.rst#L38-L146)

| DexLab 标签 | 角色与直觉 | 原生定位与必须保留的限制 |
|---|---|---|
| **Featherstone** | 实现/系统；用关节广义坐标描述树形多体 | `SolverFeatherstone`；固定实现装配关节空间质量矩阵并作 Cholesky 求解，接触走 penalty。名字不证明这里采用某个教科书算法的全部细节 |
| **SemiImplicit**（词典别名 `semi_implicit`） | 实现/系统；在当前状态装配力，再推进速度和位姿 | `SolverSemiImplicit`；刚体用 maximal coordinates，接触 penalty；没有可任意套用的接触残差迭代循环 |
| **XPBD** | 实现/系统；预测之后迭代修正位置约束 | `SolverXPBD`；刚体接触与关节修正的参数、量纲和读回并不完全相同 |
| **VBD** | 实现/系统；按顶点/刚体 block 更新离散问题 | `SolverVBD`；刚体与粒子需要对应着色；单写 VBD 尚未交代刚体分支 |
| **VBD compliant** | VBD 的 compliant ALM 分支 | 显式 `rigid_compliant_alm=True`；有限材料刚度与数值 ALM 参数分开 |
| **VBD legacy** | VBD 的旧 AVBD 分支 | 显式 `rigid_compliant_alm=False`；该固定版本已标弃用，不能把旧默认推成将来保证 |
| **Kamino** | 实现/系统；用 maximal 刚体变量处理机构和约束 | `SolverKamino`；实验性系统，动力学后端与积分器是独立配置 |
| **DVI** | Kamino 动力学求解后端 | `Config(dynamics_solver="dvi")`；bilateral 块直接解与 unilateral 接触/限位投影迭代交替 |
| **PADMM** | Kamino 动力学求解后端 | `Config(dynamics_solver="padmm")`；分裂变量、投影与对偶更新，有自己的线性子问题与残差 |
| **Euler** | 积分配置 | 对已登记 Kamino 路径定位 `integrator="euler"`；此处先更新速度再推进位姿，不能只凭名称称作所有项显式的 forward Euler |
| **100 sweeps** | 迭代预算标签 | 斜面报告将它附在 XPBD、VBD legacy/compliant；不是物理时长、精度或收敛证明，也不应抄成 Kamino 通用配置 |
| **Style3D** | 专用柔性实现 | `SolverStyle3D`；本固定树中的 Projective Dynamics 布料路径，含 PCG 线性子求解，不是另一个外部 Style3D 程序 |
| **ImplicitMPM** | 专用材料实现 | `SolverImplicitMPM`；粒子携带材料状态，借网格计算与更新；不独立推进刚体关节，也不是普通 tetrahedral soft-body solver 的别名 |

深入对应：[E3 接触/求解/读回](contact-solvers-forces.md)、[E6 材料、VBD、Style3D、MPM 与 Kamino](extensions-boundaries.md)。下面只展开做配置判断必需的差异。

## 2. 从一次状态更新看 solver 选择

对一个平移自由度，把半隐式更新写成

$$v_{n+1}=v_n+h f_n/m,\qquad x_{n+1}=x_n+h v_{n+1}.$$

这里 $h$ 单位 s、$m$ 为 kg、$f$ 为 N、$x$ 为 m；这只是当前力已知时的标量解释，实际刚体还有角动量、四元数、约束与阻尼。SemiImplicit 的 penalty 接触会把穿透转为力，因此更大的 stiffness 和更大的 $h$ 并不是互相独立的安全旋钮。对于无阻尼线性弹簧这个特例，稳定范围是 $h\sqrt{k/m}<2$；它不是完整接触任务的统一阈值。[SemiImplicit 力到积分](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/semi_implicit/solver_semi_implicit.py#L122-L215)

Featherstone 换成广义坐标后，树形关节结构进入运动学与惯量表示；固定实现中的质量矩阵装配/Cholesky 是**动力学线性求解**，不是一个摩擦互补迭代。提高 contact iterations 不是此路径的统一操作。[装配与求解](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/featherstone/solver_featherstone.py#L765-L982)

XPBD 则在预测后反复处理约束。原生 `iterations` 控制同一个步内的位置修正预算；刚体恢复速度处理还有另一项 `rigid_contact_restitution_iterations`。加前者不等于多次采样 policy，不等于刷新整个接触集合，更不意味着后者同步变大。[构造参数](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/xpbd/solver_xpbd.py#L117-L163)、[位置循环](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/xpbd/solver_xpbd.py#L710-L856)

## 3. VBD：材料柔顺与求解预算分别改变什么

VBD 的局部 block 更新可帮助理解“大系统如何拆成局部小问题”。实际刚体分支还会保存约束历史，因此不能把它等同于只做一次顶点平滑。固定实现中的 compliant ALM 将材料刚度 $k$ 和数值 metric $\rho$ 组合为

$$k_{\mathrm{eff}}=\frac{k\rho}{k+\rho}.$$

在一条平移弹性约束上可以把二者都按刚度尺度理解；旋转/其他约束行应按各自坐标定义核对。即使 $\rho\to\infty$，$k_{\mathrm{eff}}\to k$，不会变成无限硬材料。增加 iterations 也不能把作者设置的有限 compliance 自动变成刚性约束。[两种刚体分支](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/vbd/solver_vbd.py#L144-L175)

以下是供阅读的**独立候选构造片段**，假设调用者已建立并正确着色 `model`；只检查语法和固定签名，未构造或运行对象，也不是完整复现实验脚本：

```python
import newton

# 分别考虑这些候选，不是让它们依次积分同一份状态。
xpbd = newton.solvers.SolverXPBD(model, iterations=100)
vbd_compliant = newton.solvers.SolverVBD(
    model, iterations=100, rigid_compliant_alm=True
)
vbd_legacy = newton.solvers.SolverVBD(
    model, iterations=100, rigid_compliant_alm=False
)
```

这里刻意写出 True/False：固定 1.6.1 源码中省略参数仍回落 legacy 并发弃用警告；“官方推荐 compliant”不等于每个旧实验已启用它。`builder.color()` 的调用时机在建模阶段，而不是漏掉后用 iterations 补救。[VBD 构造与预算](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/vbd/solver_vbd.py#L302-L362)、[省略值分支](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/vbd/solver_vbd.py#L576-L588)、[着色要求](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/vbd/solver_vbd.py#L228-L236)

## 4. Kamino：DVI/PADMM、Euler 和收敛是三件事

**DVI** 在这里是具体实现后端名。固定实现交替处理 bilateral 关节块与 unilateral 限位/摩擦接触；前一块是直接解，后一块使用着色 projected Gauss–Seidel。**PADMM** 则用自己的分裂/投影/对偶更新链；它与 E6 中连接不同 solver 的 coupled ADMM 不在同一层。[DVI 循环](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/kamino/_src/solvers/dvi/solver.py#L745-L857)、[PADMM 循环](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/kamino/_src/solvers/padmm/solver.py#L381-L420)

```python
import newton

# 源码核对片段：model 由调用者准备，此处未执行。
config = newton.solvers.SolverKamino.Config(
    dynamics_solver="dvi", integrator="euler"
)
kamino = newton.solvers.SolverKamino(model, config=config)
```

在构造 Config 时选后端，让相关默认一起初始化；不能先构造默认 PADMM 配置，再只改一个字符串就假设其余字段正确。固定实现要求 DVI 关闭 dynamics preconditioning；PADMM 的 adaptive penalty 对 sparse dynamics 另有条件。[独立配置项](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/kamino/solver_kamino.py#L266-L280)、[拒绝条件](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/kamino/solver_kamino.py#L441-L459)

Euler 更新器先更新 twist，再用更新后的速度推进 pose，并对旋转使用 log-map 路径。它回答时间离散怎么做，不回答 DVI/PADMM 在步内是否已收敛。[实际 Euler 更新](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/kamino/_src/integrators/euler.py#L46-L87)

`status.converged`、`iterations`、`r_p/r_d/r_c` 要按后端解释。DVI 的 cone/dual-cone 距离与 PADMM 的 consensus/dual 残差不是同一个定义；字段同名不足以共享阈值。迭代达到上限只说明停止，不自动说明误差达标；即使数值收敛，也还需检查物理协议与观测有效性。[逐字段定义](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/docs/solvers/kamino.rst#L75-L116)

## 5. Style3D 与 ImplicitMPM：先选模型，再看数值解法

Style3D 的固定布料实现采用 Projective Dynamics 与 PCG：`iterations` 是非线性预算，`linear_iterations` 是每轮线性预算；两者不是同一旋钮。建模前需要注册对应 custom attributes，并使用匹配的 cloth 构造入口。它还会原地修改 `state_in.particle_q`，不能沿用“所有 solver 输入状态都只读”的假设。[原生类与参数](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/style3d/solver_style3d.py#L36-L147)、[状态契约](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/style3d/solver_style3d.py#L166-L179)

ImplicitMPM 的粒子携带材料历史，网格承担计算；不同流变参数对应不同材料问题。`SolverImplicitMPM.register_custom_attributes(builder)` 应在构建前调用。固定 step 不消费通用 `Control` 或调用者提供的 `Contacts`，不能照抄一个刚体控制循环就宣称得到同样的受控耦合。[属性注册](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/implicit_mpm/solver_implicit_mpm.py#L1188-L1220)、[step 契约](https://github.com/newton-physics/newton/blob/713fecdc41caf0c9d726f5c016939f36e66e3dff/newton/_src/solvers/implicit_mpm/solver_implicit_mpm.py#L1901-L1928)

因此，“MPM 比 VBD 好吗”还没有定义比较问题：先说明是布料、体材料、刚体接触还是耦合任务，确认几何、材料、控制与观测都能表达，再讨论离散和求解预算。详细材料公式与耦合所有权沿 [E6](extensions-boundaries.md) 继续读。

## 6. 用已有证据练习诊断

| 看到的现象或标签 | 下一步应该核对什么 | 不能据此直接宣称什么 |
|---|---|---|
| `100 sweeps` 后仍滑动或抖动 | 时间步、摩擦消费者、几何与材料、实际残差/状态有效性；明确哪个循环有 100 次 | “100 次已收敛，所以必定是 policy 错” |
| VBD 力与动量变化对不上 | 返回力的 body/参考点、上一姿态是否正确、最终 primal/dual 的采样阶段 | “加 iterations 一定修复” |
| 两个 solver 配相同 `mu` | 两侧材料组合律、接触集合、摩擦近似是否相同 | “相同物理问题已经公平对齐” |
| MPM 或 Style3D 无法沿用原控制输入 | 实际字段消费者、建模属性与耦合接口 | “统一 step 签名意味着能力完全相同” |
| Kamino `converged=True` | 后端残差定义、状态表示、力读回和原协议负例 | “整个任务已通过物理验收” |

DexLab 的[固定斜面报告](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/newton-incline-results.md)记录的是 Newton 1.6.1 / Warp 1.18.0、CPU FP32、40 mm / 64 g 立方体的特定协议。其 VBD 诊断保留了奇偶迭代预算模式及原始失败；诊断探针没有替换原得分。这是一个“增加预算未自动消除观测问题”的阅读案例，不是本教程独立复现，也不是任意场景的引擎排名。

复读时依次打开报告、[profile 与摘要索引](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/evidence/newton-incline/profiles.json)、[VBD 诊断记录](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/docs/evidence/newton-incline/vbd-diagnosis.json)，分别记下配置、可用性判据、物理阈值与诊断条件；不要把四种记录合成一个“成功率”。

## 7. 版本对照：课程来源与历史运行各有身份

| 项目 | 此处能确认什么 | 仍然不能推出什么 |
|---|---|---|
| Newton 1.6.1 / `713fecdc…` | 课程固定源码与上述斜面 profiles 的 `source_commit` 一致 | 本机已经安装、每个导出 solver 都被实验覆盖 |
| Warp 1.18.0 | DexLab 上述协议记录该版本；课程另有 [Warp 固定来源](e5-warp-sources.json) | 当前工作机的组合运行已验收 |
| 历史 Newton `1.7.0.dev0 @ 2dee3234` | DexLab [总览快照](https://github.com/huangkiki/Dexlab/blob/7e4a0774efb4f8b332e866e1495e032b4969c8aa/README.md)登记了另一个版本身份 | 能用 1.6.1 的默认值、API 或本页片段回填那次运行 |
| 包装层 `SolverMuJoCo` | 是 Newton 中接到独立 MuJoCo 后端的路径，课程 [E3](contact-solvers-forces.md)单列其依赖 | “名称包含 MuJoCo”就已确认历史原生路径、算法和物理资格 |

本文没有逐项重建 `2dee3234` 的历史配置，因此该部分保留为**登记身份，历史设置未在本页重新核验**。官方最新文档、当前安装包和实验当时的构建/配置不可互相替代。

## 8. 阅读练习与参考答案

1. 一次 policy action 保持 20 ms，物理步长为 1 ms，XPBD 每步 100 iterations。这个区间有多少物理步？把 iterations 改成 200 会延长模拟时间吗？
2. 平移约束 $k=1000$ N/m、$\rho=1000$ N/m 时 $k_{eff}$ 是多少？把 $\rho$ 无限增大能否得到无限刚体？
3. `Kamino / DVI / Euler` 与 `VBD compliant / 100 sweeps` 可以按标签数量宣称相同计算预算吗？
4. 从当前 1.6.1 源码看到 VBD 省略开关回落 legacy，能否给历史 `1.7.0.dev0 @ 2dee3234` 记录补写 legacy？

参考答案：

1. 20 个物理步；100 对应每步求解预算，这里合计 2000 轮位置迭代。改成 200 后仍是 20 ms，但预算翻倍；算力开销和实际误差仍需单独测量。
2. 500 N/m；极限为 1000 N/m。数值 metric 的极限不消除有限材料柔顺。
3. 不能。前者包含系统、后端与积分器，后者包含系统分支与预算；未知的时间步、残差判据、矩阵规模和每轮工作都影响成本。
4. 不能。必须读取该历史版本与实际配置/运行记录；当前默认只能解释当前固定源码。

## English summary

This source-only reading guide decodes every Newton Physics label in the pinned DexLab taxonomy. Solver implementations, VBD branches, Kamino dynamics backends, time integration and iteration budgets occupy different roles. It connects native configuration signatures to the existing E3/E6 lessons, with diagnosis prompts and answered exercises for readers experienced in VLA, WAM or RL. The source baseline is Newton 1.6.1 at `713fecdc…`; the separate historical `1.7.0.dev0 @ 2dee3234` record is not reconstructed from current defaults. No native simulation or new physical qualification was performed. E7 remains paused.
