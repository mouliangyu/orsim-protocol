# orsim/v1

块必须包含 `protocol: "orsim/v1"`、字符串 `block_id`、非负整数（或 ASCII 十进制字符串）`bit_width` 和数组 `instructions`。可选 `source` 是逐行源码字符串数组，仅用于说明；可选 `ir` 是具名变量指令数组，用于表达 offset 编码之前的数据流和控制流。两个解释器都会校验块级字段及两层指令的静态格式。`simulate()` 执行 OR 指令，`simulate_ir()` 执行 IR；运行时引用可用性只按实际执行路径检查。两者的计算结果可另行比较。

所有绝对寄存器和临时寄存器固定使用有符号 64 位整数，范围为 `[-2^63, 2^63-1]`（即 `[-9223372036854775808, 9223372036854775807]`）。输入值或算术结果超出范围时报错，不进行静默截断或回绕。`bit_width` 只控制 `t#n` 的偏移编号。块级字段仅允许上述必填字段和可选的 `source`、`ir`，不再使用 `value_width`。

指令字段严格匹配下表，多余字段、缺少字段以及旧的 `inputs` 写法都会报错。 JSON 的推荐排版约定是：`instructions` 数组中每条指令使用一个单行 JSON 对象，只在相邻指令之间换行；不要把同一条指令拆成多行。解析器按 JSON 语义读取，不依赖空白。

| op | 字段 | 语义 | 产生 t |
| --- | --- | --- | --- |
| GET | reg | 读取绝对寄存器 | 是 |
| CONST | value | 产生有符号 64 位整数常量 | 是 |
| SET | reg, value | 将 value 写入绝对寄存器 reg | 否 |
| COPY | src | 复制一个临时值并产生新的临时值 | 是 |
| NEG / NOT | src | 算术取负 / 按位取反 | 是 |
| ADD / SUB / MUL / DIV / MOD | lhs, rhs | 加、减、乘、整数除法、取余 | 是 |
| AND / OR / XOR | lhs, rhs | 按位与、或、异或 | 是 |
| SHL / SHR | lhs, rhs | 左移、算术右移；rhs 是移位位数 | 是 |
| EQ / NE / LT / LE / GT / GE | lhs, rhs | 比较，产生 0 或 1 | 是 |
| SELECT | cond, yes, no | cond 非零取 yes，否则取 no | 是 |
| BR | cond, target | cond 非零时跳转，否则继续 | 否 |
| JMP | target | 无条件跳转 | 否 |
| LABEL | name | 定义标签 | 否 |

所有指令都必须有 `op`。`reg` 必须匹配 `r0`、`r1`、`r2`……的形式；这表示外部绝对索引寄存器。不能使用 `x`、`A00`、`t#1` 或其他名字。寄存器只存整数；没有隐含常量、数组寻址或溢出截断。`GET` 的寄存器必须已定义；`SET` 可以定义新的输出寄存器。只有 `CONST.value` 可以填写整数立即数，不接受布尔值或范围外整数。
除 `CONST.value` 外，作为数据输入的 `src`、`value`、`lhs`、`rhs`、`cond`、`yes`、`no` 必须为 `t#n`，其中 `n` 是无前导零的 ASCII 十进制整数，且 `1 <= n <= 2^bit_width`。它们不能填写绝对寄存器名或立即数。`t#n` 指实际执行路径上倒数第 n 个产生的临时值，运算读取完所有输入后才追加结果。`COPY` 读取 `src` 后追加一个相同数值，这是数据流中的一次显式复制成本。所有临时值属于同一条历史；标签、分支和 SET 不占位置。尚无对应历史值时运行报错。循环每次执行产生新值。

`DIV` 向零截断商；`MOD` 满足 `a = (a DIV b) * b + (a MOD b)`，余数与被除数同号。除数为零报错，`-2^63 DIV -1` 因商越界报错，`-2^63 MOD -1` 则产生 `0`。`SHL` 和 `SHR` 的位数必须在 `[0, 63]`；`SHR` 对负数做符号扩展。`AND/OR/XOR/NOT` 均按有符号 64 位二进制补码逐位运算，结果仍是有符号 64 位数；语言层负责逻辑与、或、非以及短路行为。`ADD/SUB/MUL/NEG/SHL` 的结果越界时报错。比较产生 `0` 或 `1`；需要逻辑非时，先用 `CONST` 产生零，再用 `EQ` 比较待判断值与零。`SELECT` 的三个输入都须已产生，条件非零时取 `yes`。

OR 的 `target` 和 `name` 是非空标签字符串。标签作用域是当前块，必须唯一，所有跳转目标必须存在。执行到指令数组末尾结束；每条实际执行的指令都计入 `max_steps`，包括 `LABEL`、`BR` 和 `JMP`。超过上限时报错。过长的 `bit_width` 或 `t#n` 十进制字符串无法转换时也报协议错误。

```json
{"op": "GET", "reg": "r1"}
```

```json
{"op": "SET", "reg": "r1", "value": "t#1"}
```

这是本项目确定的动态执行历史语义；控制流合并后，相同的 `t#n` 可能对应不同路径上的生产者。

在汇合点使用同一段公共指令时，各前驱路径必须让每个所需逻辑值落在相同的相对位置。例如公共指令把 `t#3/t#2/t#1` 解释为 `x/y/z`，则每条到达路径都必须满足这个映射；路径产生的临时值总数可以不同。必要时可在路径末尾用 `COPY` 重排相对位置。见 [控制流对齐 case](benchmarks/control_flow_alignment.json)。这是指令流编排约束，模拟器按实际路径执行并读取对应位置，不会自动插入对齐指令。

多个基本块可用 `LABEL` 标出入口，以 `BR` 或 `JMP` 连接；`BR` 未跳转时继续执行下一条指令。复杂 CFG 中应分别检查每个汇合块的所有前驱路径，以及汇合后每个后继继续读取的 `t#n`。示例见 [两层四路 CFG](benchmarks/cfg_two_layer_alignment.json)。

## 具名变量 IR

`ir` 和 `instructions` 是同一个块的两种表示：`ir` 用变量名连数据流，`instructions` 用 `t#n` 编码生产者距离。`ir` 不受 `bit_width` 限制，不包含为 offset 窗口刷新而插入的 COPY。变量名及 IR 标签名均匹配 `[A-Za-z_][A-Za-z_0-9]*`；变量必须先定义再读取，允许重新赋值。执行时只有当前路径上的指令会赋值，变量环境会跨分支保留；若某路径没有重新赋值，汇合块读到的是该路径先前的值，若从未定义则报错。`MOV` 用于把路径结果赋给汇合后的统一变量名。`ir` 的算术和寄存器值使用同样的有符号 64 位范围；`BR` 仍以非零为真。IR 每条实际执行的指令同样计入 `max_steps`。

| IR op | 字段 | 语义 |
| --- | --- | --- |
| GET | dst, reg | 读绝对寄存器，赋给变量 |
| CONST | dst, value | 把有符号 64 位立即数赋给变量 |
| SET | reg, value | 将变量写入绝对寄存器 |
| NEG / NOT | dst, src | 读取变量，计算后赋给 dst |
| ADD / SUB / MUL / DIV / MOD | dst, lhs, rhs | 读取两个变量，计算后赋给 dst |
| AND / OR / XOR / SHL / SHR | dst, lhs, rhs | 位运算或移位 |
| EQ / NE / LT / LE / GT / GE | dst, lhs, rhs | 比较并赋 0 或 1 |
| SELECT | dst, cond, yes, no | 按变量 cond 选取一个变量值 |
| MOV | dst, value | 把变量值赋给另一个变量 |
| BR | cond, target | 变量非零时跳转 |
| JMP | target | 无条件跳转 |
| LABEL | name | 定义标签 |

例如 `source` 为 `sum = r1 + r2; r1 = sum * sum;` 时，IR 可写为：

```json
{"op": "GET", "dst": "a", "reg": "r1"}
{"op": "GET", "dst": "b", "reg": "r2"}
{"op": "ADD", "dst": "sum", "lhs": "a", "rhs": "b"}
{"op": "MUL", "dst": "square", "lhs": "sum", "rhs": "sum"}
{"op": "SET", "reg": "r1", "value": "square"}
```

这些对象在 JSON 文件中放进 `ir` 数组，每条指令占一行。独立的 `simulate_ir()` 执行 `ir`，可以与 `simulate()` 的 OR 结果及源码模型比较；目前 [简单示例](example.json) 和 [复杂 CFG case](benchmarks/cfg_copy_explosion.json) 包含这层表示。

## Benchmark 约定

benchmark 放在 `benchmarks/`，输入 JSON 仍遵循同一协议。benchmark runner 可以额外输出统计信息，但统计不属于指令流语义。推荐至少报告 `instructions`、`COPY` 数、最大 `t#n`、输出寄存器和执行耗时；比较不同调度时必须使用相同的 `source` 和初始寄存器，确认结果一致后再比较 COPY 成本。

`source` 本身不是可执行字段。每个正式 case 必须有一个独立 reference model，使用普通源码语义计算结果；`benchmarks/validate_cases.py` 负责把 reference model 与 `simulate()` 的完整寄存器结果逐项比较。只看单个示例输入不足以证明指令流正确，验证应覆盖多个输入和控制流路径。
