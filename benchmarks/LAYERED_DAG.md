# 分层交叉依赖调度题

这三道题固定 `bit_width=3`，每条输入只在开头 `GET` 一次，每条输出只在末尾 `SET`。每层生成与 lane 数相同的 `ADD`；旧层的每个 lane 至少作为一次左操作数，右操作数由固定种子选取，更新顺序也被打乱。下一层只能使用上一层结果。每个 JSON 的 `source` 描述计算规则，`ir` 列出确切依赖边，`instructions` 是合法参考调度。

| Case | GET + ADD | 默认束宽 16 | 束宽 96 | 束宽 512 的参考解 |
| --- | ---: | ---: | ---: | ---: |
| [六路十二层](layered_dag_6_12_17.json) | 78 | 22 COPY | 4 COPY | 1 COPY |
| [七路六层 0](layered_dag_7_6_0.json) | 49 | 未找到可行解 | 未找到可行解 | 7 COPY |
| [七路六层 10](layered_dag_7_6_10.json) | 49 | 未找到可行解 | 11 COPY | 5 COPY |

这里的失败是当前束搜索剪枝造成的，参考解证明实例可以编码。1、5、7 COPY 均只是**可行上界**，没有最优性证明。宽搜索也不是单调改进的保证：束宽变大可能保留不同布局，最终结果可能变差。题目包含多层共享值和相互竞争的后继，不依赖重复无用计算或预先放置的 COPY。

生成和复核：

```sh
python3 benchmarks/generate_layered_dag.py
python3 benchmarks/validate_cases.py
python3 benchmarks/run_scheduler.py
```

生成器使用固定种子，独立的 Python 源码模型重新生成依赖关系。验证脚本对零值、正值、交替符号和偏移输入，分别比较源码模型、具名 IR 和 OR 结果。测量脚本额外测试当前默认调度器；`search_failed` 表示调度器没有找到可行结果，不表示 IR 不合法。
