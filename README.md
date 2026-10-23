# Offset Register 协议与示例

本仓库公开 `orsim/v1` 协议和少量 JSON case，用于讨论 offset register 的数据流表示、COPY 成本以及控制流汇合时的相对索引对齐。完整的指令字段、整数边界和执行语义见 [PROTOCOL.md](PROTOCOL.md)。

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

建议先阅读基础示例，再看控制流 case。所有 case 固定使用 `bit_width=3`。本仓库只提供协议与示例数据。
