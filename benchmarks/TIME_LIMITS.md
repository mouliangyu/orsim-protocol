# Benchmark 调度时间限制（暂定）

每个 `benchmarks/*.json` 有 `time_limit_ms` 正整数，表示单次调度预算，单位为毫秒。该字段是评测元数据，不改变 OR 或 IR 语义。计时从调度器收到已解析的 case 开始，到完整候选结果生成结束；JSON 解析、进程启动、模拟验证和结果评分不计入。先判定结果是否合法，再在限时内按 COPY 数评价；提前完成没有额外奖励。

当前预算按现有 Python 调度器在 Intel Xeon Platinum 8378C、Python 3.11.14 上的五次连续运行确定。多数直线题中位数低于 0.1 秒，给 **1000 ms**；耗时约 0.4–0.5 秒的三题给 **3000 ms**；最慢的 `layered_dag_7_20_0` 约 2.5 秒，给 **10000 ms**。预算刻意留出运行环境和其他实现的余量，不表示求解这些题理论上需要这么久。`benchmarks/time_limits.py` 保存逐题清单；运行生成器后执行 `python3 benchmarks/time_limits.py` 可重新写入 JSON；`benchmarks/run_scheduler.py` 报告实测中位数、p95 和对应预算。

三道 CFG case 的现有调度器尚不支持，统一暂定 **10000 ms**。这是未测量的占位预算；正式使用前需要有 CFG 调度器和固定评测环境的数据。所有预算也应在固定 CPU 配额、语言运行时和并发条件下重新校准。

| 预算 | Case |
| ---: | --- |
| 1000 ms | 其余 16 道直线题 |
| 3000 ms | `layered_dag_7_20_10`、`layered_dag_7_6_0`、`layered_dag_8_4_13` |
| 10000 ms | `layered_dag_7_20_0`；三道 CFG case（暂定） |
