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

## COPY 压力与调度对照

| Case | 计算与 COPY 来源 | 静态 COPY |
| --- | --- | ---: |
| [内部刷新](benchmarks/copy_pressure_internal.json) / [外部重读](benchmarks/copy_pressure_reload.json) | 同一段 128 步累加：前者用 COPY 延续内部值，后者在对应位置重新 GET 外部寄存器。 | 18 / 0 |
| [多状态 DAG](benchmarks/dataflow_dag.json) | 三个状态交叉更新，共享中间值并复用一个 seed。 | 17 |
| [五状态原始调度](benchmarks/rolling_dag.json) / [零 COPY 调度](benchmarks/rolling_dag_zero_copy.json) | 相同源码、相同输入输出与 8 轮计算；只改变 ADD 顺序，COPY 从 57 降为 0。 | 57 / 0 |

这些 COPY 计数对应文件中写定的指令顺序；特别是五状态对照说明高 COPY 数不等于计算本身必须复制。建议先阅读基础示例，再看控制流和压力 case。所有 case 固定使用 `bit_width=3`。本仓库只提供协议与示例数据。
