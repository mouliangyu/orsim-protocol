# Offset Register 协议与示例

本仓库公开 `orsim/v1` 协议和精选 JSON case，用于讨论 offset register 的数据流表示、COPY 成本以及控制流汇合时的相对索引对齐。完整的指令字段、整数边界和执行语义见 [PROTOCOL.md](PROTOCOL.md)。

## 核心概念

OR 指令的结果依次进入临时值历史。在当前指令中，`t#1` 引用实际执行路径上最近产生的值，`t#2` 引用再前一个值。`bit_width=3` 时，最大可引用偏移是 `t#8`。`COPY` 从历史中读取一个值，再把它追加为最新值；它既可延长内部值的可引用时间，也可调整控制流汇合处的相对位置。

每个 case 的 `source` 展示计算意图，`ir`（如有）用具名变量和指令表达数据流，`instructions` 是使用 `t#n` 的 OR 指令序列。IR 中的 `MOV` 是变量赋值；OR 中的 `COPY` 是实际产生新临时值的指令。JSON 数组中的每条指令各占一行。

## Case

| 文件 | 内容 |
| --- | --- |
| [example.json](example.json) | `(r1 + r2)^2` 写回 `r1`；同时展示 source、IR 和 OR。 |
| [control_flow_alignment.json](benchmarks/control_flow_alignment.json) | 两次分支与汇合，展示路径间的相对索引对齐。 |
| [cfg_two_layer_alignment.json](benchmarks/cfg_two_layer_alignment.json) | 两层四路控制流，多个基本块参与汇合对齐。 |
| [cfg_copy_explosion.json](benchmarks/cfg_copy_explosion.json) | 两层四路控制流与交叉数据依赖；同时包含具名变量 IR 和 OR。 |
| [cfg_loop_alignment.json](benchmarks/cfg_loop_alignment.json) | 三状态循环、奇偶分支和回边相对索引对齐；固定排序需要 29 COPY，参考调度为 27 COPY。 |
| [cfg_loop_stress.json](benchmarks/cfg_loop_stress.json) | 三状态循环，每轮两次条件分支和汇合，参考调度 49 COPY。 |
| [cfg_loop_stress_4.json](benchmarks/cfg_loop_stress_4.json) | 四状态循环，每轮两次条件分支和汇合，参考调度 79 COPY。 |
| [cfg_loop_stress_4x4.json](benchmarks/cfg_loop_stress_4x4.json) | 四状态循环，每轮四次条件分支和汇合，参考调度 147 COPY。 |

## COPY 压力与调度对照

| Case | 计算与 COPY 来源 | 静态 COPY |
| --- | --- | ---: |
| [内部刷新](benchmarks/copy_pressure_internal.json) / [外部重读](benchmarks/copy_pressure_reload.json) | 同一段 128 步累加：前者用 COPY 延续内部值，后者在对应位置重新 GET 外部寄存器。 | 18 / 0 |
| [多状态 DAG](benchmarks/dataflow_dag.json) | 三个状态交叉更新，共享中间值并复用一个 seed。 | 17 |
| [五状态原始调度](benchmarks/rolling_dag.json) / [零 COPY 调度](benchmarks/rolling_dag_zero_copy.json) | 相同源码、相同输入输出与 8 轮计算；只改变 ADD 顺序，COPY 从 57 降为 0。 | 57 / 0 |
| [六路交错](benchmarks/straight_fanout_interleaved.json) / [六路分组](benchmarks/straight_fanout_grouped.json) | 相同内部 seed、六个 lane 各更新 12 次；改变调度后 COPY 从 11 增至 216。 | 11 / 216 |

## 交叉依赖调度题

[CROSS_DAG.md](benchmarks/CROSS_DAG.md) 介绍四道新题。每道题独立提供 `source`、具名 IR 和一份已验证的 OR 参考解，均使用 `bit_width=3`。题目规模为 43 至 83 条 IR 指令；参考解分别有 14、4、16、34 条 COPY。当前四候选贪心调度器在前两题分别产生 31、16 条 COPY，在后两题未找到可行解。参考解不是最优性证明。

模拟器仓库的默认调度器增加束搜索后，四题分别得到 8、4、9、16 条 COPY。题目 JSON 仍保留原参考解，方便复现和比较；这些结果也不是最优性证明。

两个新增 case 均有 `source`、`ir`、OR 指令；研究背景和后续 benchmark 方向见 [STRAIGHT_RESEARCH.md](benchmarks/STRAIGHT_RESEARCH.md)。这些 COPY 计数对应文件中写定的指令顺序，不代表计算本身必须复制。所有 case 固定使用 `bit_width=3`。本仓库只提供协议与示例数据。

## 分层交叉依赖调度题

[题目说明](benchmarks/LAYERED_DAG.md) 提供三道 `bit_width=3` 压力题：六路十二层的默认调度需要 22 COPY，较宽搜索给出 1 COPY 的合法参考解；两道七路题使束宽 16 的快速搜索找不到可行解；模拟器仓库现在使用完整搜索兜底，默认调度器也能找到可行解。参考指令流分别使用 7 和 5 COPY。每道题包含 `source`、具名 IR 和 OR；数字是算法表现或可行上界，不是最优性证明。

新增七路二十层与八路四层三道压力题。束宽 16 快速搜索对三题均失败，束宽 512 的合法参考解分别需要 21、25、25 COPY；本机计时与完整调度器结果见 [分层题说明](benchmarks/LAYERED_DAG.md)。

## 规则化调度题

[交替步长环形数据流](benchmarks/RING_STRIDE.md) 以七状态、交替步长 1/3 的确定性递推构造共享依赖。8 轮与 12 轮均有 `source`、具名 IR 和合法 OR 参考解；它们测试同一固定数据流上调度选择对 COPY 数的影响。随机依赖图仅作为补充覆盖。

每道 benchmark JSON 现包含 `time_limit_ms`，单位为毫秒；逐题预算与计时口径见[时间限制说明](benchmarks/TIME_LIMITS.md)。七道控制流题已用 CFG 调度器实测，其中包含四题有环 CFG；计时与限制见时间限制说明。
