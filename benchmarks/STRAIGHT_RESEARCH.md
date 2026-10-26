# STRAIGHT 论文与 orsim benchmark

## 资料与边界

- Koizumi 等，《[Compiling and Optimizing Real-world Programs for STRAIGHT ISA](https://doi.org/10.1109/ICCD53106.2021.00070)》，ICCD 2021，[IEEE 页面](https://ieeexplore.ieee.org/document/9643774)。以下关于该论文的陈述依据可公开读取的摘要；本文没有把未获取到的正文细节当作已核实结论。
- Shioya 等，《[STRAIGHT: Hazardless Processor Architecture Without Register Renaming](https://doi.org/10.1109/MICRO.2018.00019)》，MICRO 2018。摘要说明源操作数由生产者指令距离指定，并讨论编译开销。
- 《[A Sound and Complete Algorithm for Code Generation in Distance-Based ISA](https://doi.org/10.1145/3578360.3580263)》，2023。摘要指出距离约束下的代码生成并非总能靠既有方法完成，尤其在寄存器数量较小时。

2021 年论文摘要明确提到：朴素代码生成器会产生额外指令；作者提出调用约定和编译优化两类方法，模拟结果中执行指令数降低 15%、性能提升 17%。这些是该论文自身实验的结果，**不是** orsim benchmark 的预期比例。

orsim 的 `t#n` 与 STRAIGHT 的“按生产者距离引用”方向相近，但现有协议只定义块内动态临时值历史、固定窗口和显式 `COPY`。它尚无函数调用、参数传递、返回、栈或内存操作；因此不能拿当前 case 评估论文的调用约定，也不能模拟重命名电路、功耗或周期级性能。

## 已实现：相同数据流的编排对照

两个 case 有相同的源码语义：六个 lane 各累加十二次同一个内部计算出的 `seed = r1 + r2`。`seed` 无法从外部寄存器直接 `GET` 回来。两个 case 均使用 `bit_width=3`，提供 `source`、具名 IR、OR 指令和独立 reference model。

| Case | 编排 | 静态 COPY | 静态 OR 指令 |
| --- | --- | ---: | ---: |
| [交错编排](straight_fanout_interleaved.json) | 每轮更新全部 lane | 11 | 98 |
| [按 lane 分组](straight_fanout_grouped.json) | 一个 lane 完成十二轮后再处理下一个 | 216 | 303 |

两者的计算指令与输入输出数量相同；差额来自窗口内维护多个存活值的 `COPY`。分组调度使其他 lane 长时间等待，同时还要保留 seed。这是一个**编排敏感性**案例，216 次 COPY 并非该数据流的下界。私有模拟器仓库中的 `benchmarks/generate_straight_fanout.py` 生成两份指令流，`benchmarks/validate_cases.py` 对多个输入分别比较源码模型、IR 与 OR 的全部寄存器输出。

## 下一批值得测的维度

| 问题 | 可构造的 benchmark | 应测指标 |
| --- | --- | --- |
| 距离窗口逼近边界 | 固定 DAG，逐步增加同时存活值数量和引用间隔；在 `t#8` 两侧取样 | 可编码率、COPY 数、最大偏移、指令数 |
| 控制流汇合对齐 | 多前继和多后继 CFG，每条路径产生不同数量且不同顺序的中间值 | 各路径动态 COPY、汇合处偏移映射、执行指令数 |
| 循环回边携带值 | 多状态迭代更新，改变回边存活状态数和循环体生产者数 | 每轮 COPY、回边对齐次数、路径正确性 |
| 代码生成完备性 | 小规模 DAG 穷举，比较贪心插 COPY 与搜索/约束求解 | 成功编码率、最少 COPY 的已证明值、求解时间 |
| 调用边界 | 扩展协议定义 CALL/RET、参数和返回值后再做递归及多返回值 case | 调用处搬运、执行指令数、不同调用约定 |

前三项可直接使用当前协议。第四项需要明确搜索空间和代价模型，才能把“最小 COPY”称为证明过的下界。第五项依赖新的协议语义，当前不应以 `GET/SET` 的外部重读来冒充函数调用。
