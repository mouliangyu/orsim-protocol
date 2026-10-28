# Benchmark 调度时间限制（暂定）

每个 `benchmarks/*.json` 的 `time_limit_ms` 是单次调度预算，单位为毫秒。它是评测元数据，不改变 OR 或 IR 语义。计时从调度器收到已解析的 case 开始，到完整候选结果生成结束；JSON 解析、进程启动、模拟验证和结果评分不计入。先判定结果是否合法，再在限时内按 COPY 数评价；提前完成没有额外奖励。

预算依据当前 Python 调度器在 Intel Xeon Platinum 8378C、Python 3.11.14 上的连续十次运行。直线题实测最高值按档位分组：最快一组不超过 70 ms，预算 **250 ms**；`layered_dag_7_6_10` 约 133 ms，预算 **500 ms**；三题约 385–505 ms，预算 **1000 ms**；`layered_dag_7_20_0` 约 1.93 s，预算 **4000 ms**。这些预算约为实测最高值的 2–3.5 倍，给进程内运行波动留余量；不表示理论上需要这些时间。

三道 CFG case 尚无可用的 CFG 调度器。`control_flow_alignment` 和 `cfg_two_layer_alignment` 暂定 **1000 ms**，较大的 `cfg_copy_explosion` 暂定 **3000 ms**；这三项没有对应的调度器实测，不应作为正式限时。正式评测还需固定 CPU 配额、运行时、并发条件并复测。`benchmarks/time_limits.py` 保存逐题清单；运行生成器后可执行 `python3 benchmarks/time_limits.py` 重新写入 JSON。`benchmarks/run_scheduler.py` 报告实测耗时和预算。

| 预算 | Case |
| ---: | --- |
| 250 ms | 其余 15 道直线题 |
| 500 ms | `layered_dag_7_6_10` |
| 1000 ms | `layered_dag_7_20_10`、`layered_dag_7_6_0`、`layered_dag_8_4_13`；两道较小 CFG（暂定） |
| 3000 ms | `cfg_copy_explosion`（暂定） |
| 4000 ms | `layered_dag_7_20_0` |
