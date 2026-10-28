# 交替步长环形数据流

这组题使用固定的七状态递推，`bit_width=3`，没有随机种子：

```c
state[i] = r[i+1];                    // i = 0..6
for (turn = 0; turn < rounds; turn++) {
    stride = (turn % 2 == 0) ? 1 : 3;
    next[i] = state[i] + state[(i + stride) % 7];
    state = next;                      // 同时更新
}
r[10+i] = state[i];                   // i = 0..6
```

每个旧状态都被下一轮的两条 ADD 使用，一轮的七个结果又全部进入下一轮。因此跨轮存在宽活跃集和共享值，步长在 1 与 3 间切换会改变相邻依赖关系。IR 保留这张具体依赖图；OR 参考解保留每个计算一次，只插入 COPY。没有无用计算或外部重读。

| Case | GET + ADD | 默认调度 COPY | 参考解 COPY | 默认耗时 | 参考搜索耗时 |
| --- | ---: | ---: | ---: | ---: | ---: |
| [8 轮](ring_stride_7_8.json) | 63 | 38 | 23 | 148 ms | 3.59 s |
| [12 轮](ring_stride_7_12.json) | 91 | 59 | 31 | 489 ms | 4.58 s |

耗时是本机五次运行的中位数。默认调度用束宽 16；参考解由束宽 256 生成。两者均成功得到合法解。参考解 COPY 数是可行上界，不是最优性证明。难度证据是**同一数据流在当前两种搜索规模下的差距**，不是原始 OR 指令流中的 COPY 总数。

```sh
python3 benchmarks/generate_ring_stride.py
python3 benchmarks/validate_cases.py
python3 benchmarks/run_scheduler.py
```

验证脚本使用独立 Python 递推模型，对零、正、交替符号和偏移输入，逐项比较源码、具名 IR 和 OR 的完整寄存器结果。
