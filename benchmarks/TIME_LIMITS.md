# Benchmark 调度时间限制（暂定）

每个 `benchmarks/*.json` 的 `time_limit_ms` 是单次调度预算，单位为毫秒。它是评测元数据，不改变 OR 或 IR 语义。计时从调度器收到已解析的 case 开始，到完整候选结果生成结束；JSON 解析、进程启动、模拟验证和结果评分不计入。先判定结果是否合法，再在限时内按 COPY 数评价；提前完成没有额外奖励。

预算依据当前 Python 调度器在 Intel Xeon Platinum 8378C、Python 3.11.14 上的连续十次运行。直线题实测最高值按档位分组：最快一组不超过 70 ms，预算 **250 ms**；`layered_dag_7_6_10` 约 133 ms，预算 **500 ms**；三题约 385–505 ms，预算 **1000 ms**；`layered_dag_7_20_0` 约 1.93 s，预算 **4000 ms**。这些预算约为实测最高值的 2–3.5 倍，给进程内运行波动留余量；不表示理论上需要这些时间。

六道 CFG 题现在使用独立的 `cfg_scheduler.py` 测量。它按基本块保留 IR 计算顺序，并在每条前驱边上插入 COPY，使汇合块所需值落在相同的相对索引；循环的活跃变量通过固定点求解，回边也执行布局对齐。有环题尝试三种全局布局，并对小循环头搜索变量排列，以循环内 COPY 加权计分。`cfg_loop_alignment` 的固定排序产生 29 条静态 COPY，搜索后为 27 条；这说明调度过程确实在优化，但不代表最优。原四题连续运行 30 次的 p95 耗时约为 4、127、67、134 ms（依次对应 `control_flow_alignment`、`cfg_two_layer_alignment`、`cfg_copy_explosion`、`cfg_loop_alignment`）。新增两层汇合循环压力题的当前调度结果：三状态、40 条 IR、49 COPY，12 次测量最高约 179 ms，预算 **500 ms**；四状态、47 条 IR、79 COPY，12 次测量最高约 808 ms，预算 **2000 ms**。这两题的优化搜索尚未改善固定排序的 COPY 数，展示了算法质量和耗时的压力点。预算是当前机器上的工程参考，正式评测仍需固定 CPU 配额、运行时和并发条件并复测。`benchmarks/time_limits.py` 保存逐题清单；运行生成器后可执行 `python3 benchmarks/time_limits.py` 重新写入 JSON。`benchmarks/run_scheduler.py` 报告实测耗时和预算。

| 预算 | Case |
| ---: | --- |
| 250 ms | 其余 15 道直线题；`control_flow_alignment`、`cfg_copy_explosion`、`cfg_loop_alignment` |
| 500 ms | `layered_dag_7_6_10`；`cfg_two_layer_alignment`；`cfg_loop_stress` |
| 1000 ms | `layered_dag_7_20_10`、`layered_dag_7_6_0`、`layered_dag_8_4_13` |
| 2000 ms | `cfg_loop_stress_4` |
| 4000 ms | `layered_dag_7_20_0` |
