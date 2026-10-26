# Instruction Set Architecture（指令集架构，ISA）

> [!abstract] 本章要解决的问题：ISA 是软件和硬件之间的合同
> 一条 instruction 可以看成写给处理器的任务单。它必须说清楚三件事：**读哪些数据、做什么操作、结果写到哪里**。学习路径是从 assembly 读到 machine code，再从 machine code 反推 register、memory 和 PC 的动作。ISA 不只是指令名称清单；它是一份让 compiler 和 processor 能够独立设计的可观察行为合同。

> **本章学习结果：**你应该能读懂 `add`、`ld`、`sd`、`beq` 等基本 RISC-V instruction，解释 R/I/S/B/U/J 格式为什么不同，并区分 ISA、ABI 和 microarchitecture。遇到编码题时，先找“操作对象和结果去向”，再找字段位置。

> 本章对应 `CA_4.pdf`，主题是 **Instruction Set Architecture (ISA)**，重点使用 `RISC-V` 作为例子。
>
> **学习定位：**零基础入门。建议先知道 bit（位）、binary（二进制）、register（寄存器）、memory（存储器）大致是什么；如果还不熟，先看 [[2 Representation and Computation of Data]] 中的 bit、two's complement（二进制补码）和浮点数基础。本章不要求你先会电路设计。
>
> **本章目标：**学完后你应能用自己的话说明 ISA 是什么、读懂几条基本 RISC-V instruction，并能描述一条 instruction 如何从 binary 变成处理器动作。
>
> **面向 AI chip design 的意义：**AI accelerator 也必须回答三个问题：计算单元接收什么指令、数据从哪里来、结果写到哪里去。理解 ISA 后，你会更容易理解 accelerator instruction、custom extension、vector instruction、`DMA (Direct Memory Access)`、memory hierarchy 和 compiler/hardware co-design。

### 教材式学习路径与可视化入口

> [!note] 在 Obsidian 中打开交互页
> 先点本表的 HTML 链接。若 Obsidian 没有交给 HTML Reader，而是显示源码或跳到外部浏览器，请从文件列表打开 `assets/3 Instruction Set Architecture/interactive/` 下对应的 `.html` 文件。页面显示但选择器无效时，检查 HTML Reader 的 `Operating Mode`：`Balance Mode` 不运行脚本；互动需要允许 inline script 的模式。较宽松模式会降低安全隔离，只对信任的本地教学文件使用。系统浏览器可作为备用。

把 ISA 想成软件与硬件之间的一份“共同合同”：软件提交 instruction，硬件承诺可观察的行为。阅读本章时，始终沿着 **程序语句 → assembly → encoding → register/memory 操作 → PC 变化** 这条路径走。

### 读取一条指令的五个问题

以 `ld x5, 16(x2)` 为例，按下面顺序分析，比先背它属于 I-type 更容易建立整体理解：

1. **它要做什么？**从 memory 读取一个值。
2. **地址从哪里来？**以 `x2` 为 base，加上 immediate `16`，得到 effective address。
3. **谁负责计算地址？**processor 的 ALU；它此时不是在做普通加法，而是在做地址计算。
4. **结果去哪里？**memory 读出的值写回 destination register `x5`。
5. **为什么需要 I-type？**这条 instruction 需要一个 source register、一个 destination register 和一个 12-bit immediate，因此字段布局要为这些信息留位置。

这五个问题会贯穿本章。**编码格式是实现这份合同的 bit-level 排版；它不是孤立的位表。**当你能从字段推回硬件动作，ISA 才算真正学会。

```mermaid
flowchart LR
    accTitle: ISA 的五步学习路径
    accDescr: 高级程序先变成汇编，再编码成机器码，处理器按 ISA 解码并访问寄存器、内存或下一条 PC。
    source[程序表达式] --> asm[assembly instruction]
    asm --> encoding[32-bit encoding]
    encoding --> decode[opcode、register、immediate]
    decode --> action[register、memory、ALU 或 PC 动作]
    action --> result[可观察结果]
```

如果 Mermaid 没有渲染，可把它压缩成一句话：**ISA 不只是指令名字，而是从 bits 到行为的完整约定。**

| 学习问题 | 本章入口 | 观察重点 | 静态 fallback |
|---|---|---|---|
| 为什么不同指令需要不同字段布局？ | [[assets/3 Instruction Set Architecture/interactive/encoding-explorer.html|instruction encoding 交互探索器]] | 切换 R/I/S/B-type，看字段职责变化 | 各 format 表与 `0x00A98933` 解码 |
| 数据为什么要在 register 与 memory 间搬运？ | Register file、memory hierarchy 与 data reuse | 跟踪 operand 的来源和去向 | `load/store` 伪代码与层次图 |

## 1. 学完本章，你应该能做什么

1. 解释 `ISA`、`microarchitecture`、`instruction`、`instruction set` 的区别。
2. 读懂一条基本的 RISC-V assembly：它使用哪些寄存器、是否访问 memory、结果写到哪里。
3. 区分 `RISC` 与 `CISC` 的设计取向，并理解为什么 RISC-V 常使用规则、固定长度的 instruction。
4. 用 `RV32I` 的三类基本 instruction 表达计算、memory access 和 control flow。
5. 解释 `sign extension`、`zero extension`、`PC-relative addressing` 和 `calling convention`。
6. 看懂 R-type、I-type、S-type、SB-type、U-type、UJ-type 的基本编码思路。
7. 把 ISA 的设计选择连接到 AI chip 的 decoder、register file、data movement、pipeline 和 custom instruction。

## 2. 分层术语 roadmap

```text
Level 0  软件看到的对象（第 3–4 节）
  ISA（指令集架构） -> instruction（指令） -> instruction set（指令集） -> register（寄存器）
      |
Level 1  一条指令如何执行（第 5–8 节）
  register file（寄存器堆） -> ALU（算术逻辑单元） -> immediate（立即数） -> PC（程序计数器）
  -> opcode/function fields（操作码/功能字段） -> load/store（读入/写回存储器）
      |
Level 2  控制流与软件协作（第 9–11 节）
  branch（条件分支）/jump（跳转） -> sign/zero extension（符号/零扩展） -> PC-relative addressing（PC 相对寻址）
  -> ABI（应用二进制接口） -> caller-save/callee-save（调用者/被调用者保存约定）
      |
Level 3  AI accelerator 接口（第 12 节以后）
  extension（指令扩展） -> custom instruction（自定义指令） -> tile（数据块）/MAC（乘加）/partial sum（部分和）
  -> DMA（直接内存访问）/NoC（片上网络）/stride（步长）/layout（数据布局） -> memory consistency（内存一致性）/collective communication（集合通信）
```

| 到达哪一层 | 学习检查 |
|---|---|
| Level 0 | 能说明 ISA 是 software 与 hardware 共同遵守的外部 contract |
| Level 1 | 读一条指令时能指出 operand、operation 和结果位置 |
| Level 2 | 能解释 branch、calling convention 和 extension 各解决什么问题 |
| Level 3 | 能把 AI workload 的 tile、MAC 和 DMA 对应到 ISA-visible 操作 |

### Reference glossary（查阅用，不是先修要求）

| 类别 | Term | 它在本章里的意思与作用 |
|---|---|---|
| 接口 | `ISA (Instruction Set Architecture，指令集架构)` | software 和 hardware 共同遵守的外部行为规范；后面所有 instruction 都要落回这个 contract。 |
| 对象 | `instruction（指令）` / `instruction set（指令集）` | 一条可执行操作 / 全部可用操作的集合；程序就是这些操作的序列。 |
| 存储 | `register（寄存器）` / `register file（寄存器堆）` | CPU 内部少量、快速的 storage / 这些 registers 及其读写端口；ALU 的 operand 通常先来自这里。 |
| 运算 | `ALU (Arithmetic Logic Unit，算术逻辑单元)` / `immediate（立即数）` | 执行 arithmetic、logic、shift 的 datapath / 直接编码在 instruction 里的小常数。 |
| 控制 | `PC (Program Counter，程序计数器)` / `branch（条件分支）` / `jump（跳转）` | 保存下一条 instruction 的地址 / 有条件改变 PC / 无条件改变 PC。 |
| 编码 | `opcode（操作码）`、`funct3/funct7（功能字段）` | decoder 用来识别 instruction 类别和具体 operation 的 bit fields。 |
| 数据搬运 | `load（读入）` / `store（写回存储器）` / `sign extension（符号扩展）` | memory 到 register / register 到 memory / 把较窄的 signed data 扩展到 register 宽度。 |
| 软件协作 | `ABI (Application Binary Interface，应用二进制接口)` / `caller-save/callee-save（调用者/被调用者保存约定）` | function 如何传参、返回和保护 registers 的约定；它们不是硬件自动替你完成的。 |
| 加速 | `extension（指令扩展）` / `custom instruction（自定义指令）` | 在 base ISA 上增加通用或专用功能；对 AI chip 来说是 software-visible accelerator interface。 |
| AI 数据流 | `tile（数据块）` / `MAC (Multiply-Accumulate，乘加)` / `partial sum（部分和）` | 矩阵局部块 / 乘加操作 / 尚未完成的中间累加值；它们决定 local SRAM reuse 和通信量。 |
| 系统搬运 | `DMA (Direct Memory Access，直接内存访问)` / `NoC (Network-on-Chip，片上网络)` / `stride（步长）` / `layout（数据布局）` | 专用内存搬运 / 片上互连 / 地址步长 / tensor 元素排列；它们决定 data movement 是否高效。 |
| 多芯片协作 | `collective communication（集合通信）` / `memory consistency（内存一致性）` | 多 accelerator 共同参与的通信 / 多处理单元观察 memory 更新的规则。 |

读后文时，每遇到一条 instruction 都问三件事：**operand 从哪里读？做什么 operation？结果或下一条 PC 写到哪里？**

## 3. 动机：硬件为什么需要 ISA？

### 从 0 开始：一行程序怎样变成 CPU 的动作？

先看一个不涉及复杂语法的例子：

```c
z = x + y;
```

这行 high-level code（高级代码）只表达“把 `x` 和 `y` 相加”。CPU 不能直接执行变量名，它需要先经过几层翻译：

```text
high-level code
    ↓ compiler
assembly（人类可读的指令名称）
    ↓ assembler
machine code（由 bits 组成的编码）
    ↓ processor decoder
datapath 执行：读 register → ALU 相加 → 写回 register
```

一个可能的 assembly 片段是：

```asm
add x5, x6, x7       # x5 = x6 + x7
```

这里 `x6`、`x7` 和 `x5` 是 register 编号；它们不是变量名，也不是 memory address。若 `x6=10`、`x7=3`，执行后 `x5=13`。之后，machine code 会把 `add、x5、x6、x7` 编码成固定位置的 bit fields；decoder 再把这些 fields 还原成控制信号。

这条链解释了 ISA 的位置：ISA 不规定某个 compiler 内部怎样优化，也不规定 ALU 必须画成什么样，但规定这条 `add` 的可见结果必须是什么。只要结果和异常行为符合 ISA，简单 CPU 和复杂 CPU 都可以实现它。

想象你写了一个 AI model（人工智能模型），其中有矩阵乘法、加法和 data movement（数据搬运）。晶体管只会响应电信号，不会直接理解“做矩阵乘法”。Software（软件）和 hardware（硬件）之间必须有一套共同约定：软件用约定的 instruction（指令）表达工作，硬件按约定执行。这套约定叫 Instruction Set Architecture（指令集架构，ISA）。

对未来设计 AI chip（人工智能芯片）的你来说，ISA 是一个关键的起点：它决定软件能怎样要求硬件工作；硬件结构则决定这些请求执行得多快、耗多少 energy（能量）。要记得，ISA 本身不是某一种具体电路。

### 本章的设计动机地图

后面每个概念都用同一条链来读：**先看问题，再看设计选择，最后看收益和代价。**

| 遇到的问题/约束 | 设计选择 | 带来的收益 | 需要付出的代价或边界 |
|---|---|---|---|
| software 不能直接理解晶体管动作 | 用 ISA 规定 software-visible behavior | compiler 和 hardware 有共同 contract | ISA 不能自动保证性能，仍需 microarchitecture |
| instruction 太复杂会增加 decode 和控制路径 | RISC-V 使用规则格式、固定长度 base instruction | decoder、pipeline 和验证更简单 | 一项高层工作可能要多条 instruction |
| CPU 等待下一条 instruction 会让硬件闲置 | 使用 pipeline 重叠多个 instruction | 提高 throughput | hazard、stall 和最慢 stage 限制性能 |
| memory 远、慢、耗能，反复搬运很贵 | 分离 load/store，并依靠 register、cache、SRAM 和 reuse | 让 data movement 成本显式可管理 | 编译器和程序员要安排 locality 与搬运 |
| 12-bit immediate 或 branch offset 不够表达大值/远地址 | 使用 `lui`、`auipc`、PC-relative addressing | 保持固定长度，同时覆盖更大范围 | 需要多条 instruction 和 assembler 展开 |
| 通用 ISA 不一定能高效表达 tensor operation | 使用 `V` 或 custom extension | 让 accelerator 看到 tile、vector 或 DMA 操作 | 增加 encoding、compiler、验证和同步复杂度 |

这张表只是阅读路线，不替代后文的局部解释。真正学习每个概念时，先问“眼前出了什么问题”，再看它把什么成本交给了 software 或 hardware，并把答案连接到下一节。

### 多年从业者会怎样看同一个设计

下面不是要求你背角色，而是提供读设计文档时的几副“眼镜”。同一项选择，对不同岗位的最优答案可能不同：

| 视角 | 首先关心什么 | 在 ISA 讨论中的典型问题 |
|---|---|---|
| Hardware / implementation | timing、area、ports、power、pipeline control | decoder 是否能在一个 cycle 内完成？register file 是否有足够读写端口？ |
| Compiler / software | 能否表达、调度、优化和维护 | compiler 能否自动生成这些 instruction？寄存器和 memory movement 是否可预测？ |
| Memory / system | bandwidth、latency、reuse、synchronization | 数据从哪里来？一次搬运能复用几次？多个 accelerator 如何保持一致？ |
| Verification / compatibility | corner case、旧程序、可验证性 | signed/unsigned、exception、encoding 和 extension 是否有清楚的行为？ |
| Performance / energy | 有用结果的 throughput、latency、energy | 峰值算力是否被访存或 stall 吞掉？增加复杂度是否真的改善 workload？ |

后文只在相关处调用其中的视角。初学时先理解“这个设计解决什么问题”，再看不同工程师为什么会提出不同的约束。

## 4. 先解决一个问题：程序怎样让硬件工作？

### ISA 是 software 与 hardware 的约定

计算机的“语言”由一个个 `instruction` 组成；所有可用 instruction 的集合叫 `instruction set`。`Instruction Set Architecture (ISA)` 则规定了这些 instruction 的**外部行为**：

- 有哪些 instruction；
- 每条 instruction 的 binary encoding 长什么样；
- 有哪些 registers、每个 register 多宽；
- instruction 怎样读取 operand、怎样产生结果；
- memory address 如何计算；
- branch、function call、exception 等行为如何定义。

ISA 是 interface，不是整个 CPU。软件针对 ISA 编译；硬件实现 ISA。只要最终行为符合 ISA，同一个 ISA 可以由不同 microarchitecture 实现：可以是简单的 in-order processor，也可以是 superscalar、out-of-order processor。

```text
high-level program
        ↓ compiler
assembly / machine code  ←── software targets ISA
        ↓
ISA-defined behavior     ←── hardware must implement this behavior
        ↓
microarchitecture → datapath / control → circuits
```

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 150" role="img" aria-label="从程序到 ISA 再到处理器实现的分层关系" style="width:100%;height:auto;background:#f8fafc;border-radius:12px">
  <defs><marker id="isa-arrow" markerWidth="9" markerHeight="9" refX="8" refY="4.5" orient="auto"><path d="M0,0 L9,4.5 L0,9 Z" fill="#475569"/></marker></defs>
  <g font-family="sans-serif" font-size="17" text-anchor="middle" fill="#0f172a">
    <rect x="20" y="48" width="175" height="54" rx="12" fill="#dbeafe" stroke="#2563eb" stroke-width="2"/><text x="107" y="81">Program</text>
    <rect x="245" y="48" width="175" height="54" rx="12" fill="#dcfce7" stroke="#16a34a" stroke-width="2"/><text x="332" y="81">Compiler</text>
    <rect x="470" y="48" width="175" height="54" rx="12" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="557" y="81">ISA</text>
    <rect x="695" y="48" width="185" height="54" rx="12" fill="#fce7f3" stroke="#db2777" stroke-width="2"/><text x="787" y="81">Microarchitecture</text>
  </g>
  <g stroke="#475569" stroke-width="2.5" marker-end="url(#isa-arrow)"><path d="M195 75 H235"/><path d="M420 75 H460"/><path d="M645 75 H685"/></g>
  <text x="557" y="30" text-anchor="middle" font-family="sans-serif" font-size="14" fill="#475569">software–hardware contract</text>
</svg>

读图时要抓住一句话：**ISA 规定“做什么”，microarchitecture 决定“内部怎样做”。**例如 `add x18, x19, x10` 的 ISA 行为是把两个 register 相加并写入 `x18`；至于使用几个 pipeline stage、是否 out-of-order、ALU 怎样布线，是 microarchitecture 的选择。

![[assets/3 Instruction Set Architecture/CA4-5.png]]

### 一个 instruction 的最小模型

把一条 instruction 看成一个小型函数：

```text
读 source operands → 做 operation → 写 destination / 改变 PC
```

例如：

```asm
add x5, x6, x7
```

含义是：读取 `x6` 和 `x7`，执行加法，把结果写入 `x5`。这里的 `x5`、`x6`、`x7` 不是 memory 地址，而是 register 的编号。

### 快速自测

**问题：**如果两颗处理器执行同一个 RISC-V 程序，它们内部的 pipeline 必须完全一样吗？

**答案：**不必。只要都符合 ISA 规定的行为，内部实现可以不同。

## 5. RISC、CISC 与流水线：为什么 instruction 要规则？

**设计动机：**如果每条 instruction 的长度、operand 位置和执行路径都差异很大，硬件 decoder 就必须处理很多例外，pipeline 也难以稳定切 stage。RISC 用规则、较小的 primitive instructions 解决这个问题，代价是一个高层操作可能需要多条 instruction；CISC 则用更丰富的 instruction 压缩 instruction stream，代价是 decode 和执行路径更复杂。后面讲 RISC-V encoding 和 pipeline 时，你会看到这不是口号，而是具体的硬件取舍。

**工程师视角：**hardware engineer 更在意 decoder、timing 和 pipeline 是否规则；compiler engineer 更在意多条简单 instruction 能否被稳定调度、是否增加 code size；verification engineer 则关心每个 instruction format 是否容易覆盖 corner case。三者共同说明：所谓“简单”，不是只看 assembly 行数，而是看整个实现和工具链的复杂度。

### RISC 与 CISC 的设计取向

`RISC (Reduced Instruction Set Computer)` 从较小、简单的 primitive instruction 集合出发，强调：

- instruction format 规则、容易 decode；
- 常见 operation 由简单 instruction 组合；
- ordinary ALU operation 主要在 registers 之间进行；
- memory access 用独立的 `load` / `store` instruction 表达；
- 规则的 datapath 更容易做 pipelining。

`CISC (Complex Instruction Set Computer)` 倾向于让一条 instruction 完成更多工作，可能使用 variable-length instruction，并允许 instruction 直接带 memory operand。这样可能减少 instruction bytes 和 assembly 行数，但 decode、执行路径和硬件实现更复杂；复杂 instruction 也可以通过 `microcode` 拆成一串内部 microinstruction。

| 对比 | RISC（以 RISC-V、Arm A64 为代表） | CISC（以 x86-64 为代表） |
|---|---|---|
| instruction length | 通常规则；RISC-V base 为 32-bit，`C` extension 可加入 16-bit | variable length，x86-64 常见 1–15 bytes |
| data access | 普通 ALU operation 用 register/immediate；load/store 单独表达 | 很多 instruction 可带 memory operand |
| decoder | 更容易做规则化 decoder | 需要处理更多格式与历史兼容 |
| pipeline | 适合规则化切分，但实际性能仍由实现决定 | 同样可以有 superscalar、out-of-order pipeline |
| 重要提醒 | RISC 不等于一定更快 | CISC 不等于一定更慢 |

真正影响 performance 的是完整系统：compiler、ISA、microarchitecture、memory system 和 workload 都重要。课件的现代观点是：不要把 RISC/CISC 当成“简单一定胜过复杂”的口号。

![[assets/3 Instruction Set Architecture/CA4-10.png]]

### Pipelining 解决的是什么问题？

`Pipelining`（流水线）把 datapath 分成多个 stage，让不同 instruction 同时处在不同 stage。经典五级流水线是：

1. `IF (Instruction Fetch)`：从 instruction memory 取 instruction；
2. `ID (Instruction Decode)`：decode，并读取 register file；
3. `EX (Execute)`：由 ALU 执行计算或计算地址；
4. `MEM (Memory)`：访问 data memory；
5. `WB (Write Back)`：把结果写回 register file。

没有 pipeline 时，一条 instruction 做完后下一条才开始；有 pipeline 后，第一条 instruction 在 `EX` 时，第二条可以在 `ID`，第三条可以在 `IF`。因此 pipeline 主要提高的是 **throughput（单位时间完成多少条）**，不一定降低一条 instruction 的 latency。

![[assets/3 Instruction Set Architecture/CA4-14.png]]

**硬件设计连接：**相邻 stage 之间需要 pipeline register；clock period 受最慢 stage 的 delay 限制。如果某个 stage 特别复杂，整个 pipeline 的 clock 就必须迁就它；stage 不平衡还可能造成 stall。对 AI chip 来说，这对应“把 matrix multiply、activation、memory access 拆成怎样的 stage，并让各 stage 尽量平衡”。

当一条 instruction 依赖前一条尚未产生的结果，或 branch 尚未决定下一条 PC 时，会出现 `pipeline hazard`（流水线冒险）；硬件可能用 `forwarding`（把结果直接旁路给下一 stage）、`stall`（暂时停住流水线）或 `flush`（丢弃错误路径上的 instruction）处理它。这解释了为什么提高 pipeline 深度并不自动等于提高真实 throughput。

## 6. RISC-V：开放、模块化的 ISA

**设计动机：**不同芯片的 workload 不同：小型控制器不需要浮点矩阵单元，AI accelerator 又需要 vector 或 tensor data movement。如果所有功能都塞进一个固定的大 ISA，硬件面积和验证成本会上升；RISC-V 用小型 base ISA 加 extensions，让实现者按需求组合功能。收益是模块化和可扩展，边界是 compiler、ABI、工具链和验证也必须跟着 extension 一起完善。

**工程师视角：**芯片设计者喜欢只实现需要的 extension，以控制 area 和 power；compiler/runtime 工程师要求 extension 有稳定的 programming model，否则硬件“能跑”却无法被自动利用；产品和生态角度则会追问：这个 custom extension 是否值得长期维护，是否会把软件锁在一代芯片上？

### 为什么本章用 RISC-V

RISC-V 是一个 free and open ISA，2010 年起源于 UC Berkeley。它强调：

- `open`：academic 和 industry 可以使用，不把 ISA 本身锁在单一厂商；
- `simple`：基础 instruction 集合规则，便于直接硬件实现；
- `modular`：小的 base ISA 加可组合的 extensions；
- `implementation-neutral`：不强制某种 microarchitecture，也不限定 ASIC 或 FPGA；
- `extensible`：可以加入标准 extension，也可以加入 custom extension。

RISC-V 的写法是：

```text
RV + register width + extensions
```

常见例子：

- `RV32I`：32-bit register + base integer ISA；
- `RV64I`：64-bit register + base integer ISA；
- `RV32IM`：在 `I` 上加入 `M`（integer multiply/divide）；
- `RV32IMAFD`：再加入 atomic 与 single/double precision floating point；
- `RV32G`：常用组合，`G = IMAFD`。

常见 extensions：

| Extension | 作用 | 对 AI chip design 的启发 |
|---|---|---|
| `I` | base integer instructions | 最基本的 control、address、integer datapath |
| `M` | multiply/divide | 乘法器、除法器、较长 latency datapath |
| `A` | atomic operations | 多核同步与共享数据结构 |
| `F`, `D` | single/double floating point | FPU 与 numerical precision |
| `B` | bit manipulation | bit-level data packing、量化与地址处理 |
| `V` | vector operations | 一条 instruction 处理多个 element，接近 data-parallel AI workload |
| custom extension | 专用功能 | 可为 accelerator 暴露矩阵乘、DMA 或特殊 data layout 操作 |

![[assets/3 Instruction Set Architecture/CA4-24.png]]

**关键提醒：**ISA extension 不是自动等于硬件加速。加入 `V` 或 custom instruction 后，仍然需要 compiler、register/data movement、执行单元、memory bandwidth 和验证环境共同支持。

### RV32 与 RV64 的直觉区别

`RV32I` 的 general-purpose register 宽度是 32 bits，`RV64I` 是 64 bits。宽度影响：

- 一次 integer operation 能处理的位数；
- pointer/address 的自然宽度；
- register file、datapath、部分 memory traffic 的面积与能耗；
- compiler 和 ABI 对数据类型的安排。

它们不表示“RV64 一定比 RV32 快”。若 workload 只需 32-bit data，64-bit datapath 可能带来额外硬件成本；如果需要大 address space 或 64-bit integer，RV64 才更合适。

## 7. 三类基本 instruction

**设计动机：**一段程序同时需要计算、搬运 data 和改变执行顺序；如果都交给一种“万能 instruction”，硬件很难提前知道 operand 在 register 还是 memory、是否要等待数据、下一条 PC 从哪里来。RISC-V 把行为拆成 computational、load/store 和 control flow 三条路径，让 decoder、datapath 和 pipeline 更容易安排。代价是 programmer/compiler 需要显式组合多条 instruction，下一步的 load/store 章节会说明为什么这种显式性反而有助于控制访存成本。

**工程师视角：**hardware engineer 通过这三条路径分离 ALU、memory interface 和 PC control；compiler engineer 则利用这种边界做 register allocation、instruction scheduling 和 loop optimization；system engineer 会进一步关心 load/store 是否造成 cache miss，以及 control flow 是否让 pipeline flush。一个看似“简单”的分类，实际上是在切分不同的硬件风险。

RISC-V base instruction 可以先分成三组：

```text
register file → ALU → register file       computational
memory → register file / register file → memory   load/store
PC + condition / target                    control flow
```

### 伪代码如何落到 RISC-V：一个最小例子

先看不带硬件细节的伪代码：

```text
function add_bias(x, bias):
    y = x + bias
    return y
```

如果 `x` 和 `bias` 已经位于 argument registers `a0`、`a1`，返回值放在 `a0`，那么它可以近似写成：

```asm
# 输入: a0 = x, a1 = bias
# 输出: a0 = x + bias
add  a0, a0, a1       # register-register instruction
jalr x0, 0(ra)        # return；x0 丢弃 jump-and-link 的返回地址
```

这里的对应关系是：伪代码中的一次加法变成一条 `add`，函数返回变成一次 `jalr`。如果 bias 是固定的小常数，例如 `3`，可以直接使用 immediate，少一次 register operand：

```asm
addi a0, a0, 3        # a0 = a0 + 3
jalr x0, 0(ra)
```

这就是 RISC 的“用少量、规则的 primitive instructions 组合出更高层操作”：每条 instruction 的工作很小，但 compiler 可以把它们排列成完整程序。

### Computational operation：register 到 register

#### Register-register instruction

一般形式：

```text
op rd, rs1, rs2
```

- `rd`：destination register；
- `rs1`, `rs2`：两个 source register；
- `op`：add、sub、and、or、xor、shift、compare 等 operation。

课件例子把 C expression：

```c
f = (g + h) - (i + j);
```

拆成三条 instruction：

```asm
add t0, g, h
add t1, i, j
sub f, t0, t1
```

这里的学习重点不是记住 `t0` 的名字，而是理解 compiler 如何把一个 expression 拆成：先算子表达式、把中间结果放到 temporary register，再组合最终结果。

![[assets/3 Instruction Set Architecture/CA4-11.png]]

课件把常见 computational operation 放在一张表里：`add/sub` 是 arithmetic，`slt/sltu` 是 comparison，`and/or/xor` 是 logical，`sll/srl/sra` 是 shift。先把它当成“同一个 register-to-register 模板下的不同 ALU operation”，再去区分 signed/unsigned 和 logical/arithmetic shift。

![[assets/3 Instruction Set Architecture/CA4-31.png]]

#### Register-immediate instruction

当一个 operand 是小常数时，不必先把常数放进 register：

```text
op rd, rs1, imm
```

例如：

```asm
addi x1, x2, 10
```

表示 `$x1 \leftarrow x2 + 10$`，其中 `10` 是直接编码在 instruction 里的 `immediate`。在 RV32I 中，普通 I-type immediate 是 12-bit signed value，范围为 $-2048$ 到 $2047$；所以 `addi x5, x6, 2048` 超出范围，不能作为普通 `addi` 直接编码。

![[assets/3 Instruction Set Architecture/CA4-32.png]]

**为什么两种形式都需要？**

- register-register 适合两个变量或两个中间结果参与运算；
- register-immediate 适合加常数、地址偏移、循环计数器更新；
- immediate 直接随 instruction 到达 decoder，少一次读取 register，也节省 instruction 数量。

#### Register 只保存 bit pattern

一个 register 本身没有 `signed` 或 `unsigned` 类型；它只是保存一串 bits。解释方式由 instruction 决定。例如同一组 bits 可以被 signed compare 或 unsigned compare 使用。上一章的 two's complement 知识在这里发挥作用：硬件要区分的是“如何解释 bit pattern”，而不是 register 上贴了什么类型标签。

#### Shift：逻辑移位与算术移位

`shift` 的意思是把一个 bit pattern 中的每一位向左或向右移动若干格。被移出 register 的 bits 会丢失，空出来的位置必须填入新 bits。**移位本身只操作 bits；它代表乘法、除法还是位字段操作，取决于你如何解释结果。**

##### 1. Left shift：左移

左移 $k$ 位时，所有 bits 向更高位移动，右边补 $k$ 个 0：

```text
0010 << 1 = 0100
0010 << 2 = 1000
```

如果把 bit pattern 当作 unsigned 数，左移一位通常相当于乘以 $2$，前提是没有有效位从最高位溢出。在 fixed-width register 中，左边被移出去的 bit 不会被保留。例如 4-bit pattern：

```text
1001 << 1 = 0010    # 最高位的 1 被丢弃
```

因此左移也可能产生 overflow。对 two's complement signed number，还要检查左移后 sign bit 是否仍然合理；`0010`（$+2$）左移两位得到 `1000`，在 4 bits 中会被解释成 $-8$，不能再代表数学上的 $+8$。

在 RISC-V 中，register-register 左移是 `sll rd, rs1, rs2`，带 immediate 的形式是 `slli rd, rs1, shamt`。

##### 2. Logical right shift：逻辑右移

逻辑右移时，bits 向低位移动，左边一律补 0。它适合把 register 当作 **unsigned bit pattern**，或从一个 word 中取出低位/高位字段：

```text
1011 >>logical 1 = 0101
1011 >>logical 2 = 0010
```

若 `1011` 被当作 unsigned 4-bit 数，它是 $11$；右移一位得到 `0101`，即 $5$，接近 $\lfloor 11/2 \rfloor$。但如果同样的 `1011` 被当作 4-bit two's complement，它代表 $-5$；逻辑右移后得到正数 $5$，所以它不保持 signed 数值的意义。

RISC-V 使用 `srl rd, rs1, rs2`（register shift amount）和 `srli rd, rs1, shamt`（immediate shift amount）。

##### 3. Arithmetic right shift：算术右移

算术右移也把 bits 向低位移动，但左边不是补 0，而是重复原来的最高位，也就是 sign bit：

```text
正数：0010 >>arithmetic 1 = 0001   # sign bit 是 0，所以补 0
负数：1110 >>arithmetic 1 = 1111   # sign bit 是 1，所以补 1
```

在 4-bit two's complement 中，`1110` 表示 $-2$。逐位移动后：

```text
原始：       1 1 1 0   = -2
向右移动：   _ 1 1 1
补 sign bit： 1 1 1 1   = -1
```

所以 arithmetic right shift 保持负数的符号。对很多正数，它近似除以 $2^k$；对负数，它更接近向负无穷取整，而不一定等于编程语言中“向 0 截断”的整数除法。例如 4-bit 中 $-3$ 是 `1101`：

```text
1101 >>arithmetic 1 = 1110 = -2
```

而 $-3/2$ 如果向 0 截断会得到 $-1$。因此不能把 `sra` 无条件当成普通 signed division 的替代品。

RISC-V 使用 `sra rd, rs1, rs2` 和 `srai rd, rs1, shamt`。三种常见右移的区别可以压缩成一句话：

| operation | 左侧填入 | 典型用途 |
|---|---|---|
| `srl` logical right | 0 | unsigned 数、bit mask、字段提取 |
| `sra` arithmetic right | 原 sign bit | signed two's complement 数的缩小/符号保持 |
| `sll` left | 右侧填 0 | 位字段移动、乘 $2^k$ 的近似操作 |

课件图展示了同一组 bits 在 logical 与 arithmetic shift 下的不同填充规则。注意：register 没有“signed 标签”；是 `srl` 或 `sra` 这条 instruction 决定如何解释最高位。

![[assets/3 Instruction Set Architecture/CA4-35.png]]

![[assets/3 Instruction Set Architecture/CA4-36.png]]

### Load/Store：register 与 memory 之间

RISC-V 的普通 ALU instruction 不直接把 memory 当 operand，而是分开写：先 `load` 到 register，再计算，最后必要时 `store` 回 memory。

一般形式：

```text
load/store register, offset(base)
```

有效地址（effective address）为：

$$
\text{effective address} = \text{contents of base register} + \text{offset}
$$

例如：

```asm
lw x1, 4(x2)
```

含义是：从地址 `$x2 + 4$` 读出 4 bytes（一个 word），放进 `x1`。这里 `4` 是 instruction 里的小 immediate offset。

| instruction | 数据宽度 | 作用 |
|---|---:|---|
| `lw` / `sw` | 4 bytes | load/store word |
| `lh` / `lhu` / `sh` | 2 bytes | signed/unsigned load half-word，store half-word |
| `lb` / `lbu` / `sb` | 1 byte | signed/unsigned load byte，store byte |

![[assets/3 Instruction Set Architecture/CA4-37.png]]

#### 为什么小于 register 宽度的数据仍然有用？

因为 memory 中的数据不一定天然按 word 存储：字符常用 byte，short 常用 half-word，packed data 可能把多个小数据放在一个大 word 中。读出后必须把较小数据扩展到 register 宽度。

- `sign extension`：复制最高位（sign bit），保持 signed 数的数值意义；
- `zero extension`：高位补 0，保持 unsigned 数的数值意义。

例：从 8 bits 扩展到 16 bits：

```text
+2: 0000 0010 → 0000 0000 0000 0010
-2: 1111 1110 → 1111 1111 1111 1110
```

因此 `lb` 与 `lbu` 不是重复 instruction：它们对相同的 memory bits 选择不同的解释规则。

### 访存很贵：为什么这是 ISA 的核心设计问题

对初学者来说，`add` 和 `lw` 都像“一条 instruction”；对硬件设计者来说，它们的成本完全不同：

- `add` 主要在 register file 和 ALU 之间传递数据；
- `lw` 除了计算地址，还要经过 cache/SRAM/DRAM 的层次，等待数据返回，并消耗 memory bandwidth 和 data-movement energy。

这里的“贵”不只表示时间长，也包括能耗、带宽占用、控制复杂度，以及等待访存造成的 pipeline stall。距离计算单元越远，通常容量越大，但 latency 和搬运成本也越高。因此 ISA 把普通计算和 memory access 分成不同 instruction，是在让软件和硬件都能看见这项成本：`add` 表示“使用已经拿到的数据”，`lw/sw` 明确表示“需要搬运数据”。

| 数据位置 | 典型特点 | 设计者首先要问什么 |
|---|---|---|
| register file / local register | 很快、容量小、端口有限 | 能否让 operand 留在这里复用？ |
| cache / local SRAM | 比 register 大，适合 tile 和 reuse | tile 是否能放下？访问是否连续？ |
| shared SRAM / NoC 另一端 | 需要片上或片间搬运 | 谁发起 DMA？如何同步？ |
| DRAM / host memory | 容量大，但 latency、bandwidth 和 energy 成本高 | 能否批量搬运并减少往返次数？ |

一个极小的对比可以说明问题。假设要计算 `y = a + b + c`，且三个数已经在 registers 中：

```asm
# 数据已经在 x5, x6, x7 中：不需要访存
add t0, x5, x6
add y,  t0, x7
```

如果每次都从 memory 重新读取：

```asm
# 每次计算前都要经过地址计算和 memory hierarchy
lw x5, 0(base)
lw x6, 4(base)
lw x7, 8(base)
add t0, x5, x6
add y,  t0, x7
```

两段代码的加法数量相同，但第二段多了三次 load。若这些数据会被循环重复使用，更好的策略通常是先批量搬到较近的 storage，再在 register/local SRAM 中重复计算：

```text
for each tile:
    DMA/load tile once into local SRAM
    for each reused element in tile:
        compute using registers or MAC array
    store tile result once
```

这就是 `data locality` 的基本直觉：

- `temporal locality`：同一数据很快会再次使用，所以保留在较近的 storage；
- `spatial locality`：相邻地址的数据可能一起使用，所以批量搬运一个连续 block；
- `data reuse`：一次 load 支撑多次 computation，摊薄搬运成本。

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 190" role="img" aria-label="通过一次搬运和多次复用降低访存成本的示意图" style="width:100%;height:auto;background:#f8fafc;border-radius:12px">
  <defs><marker id="reuse-arrow" markerWidth="9" markerHeight="9" refX="8" refY="4.5" orient="auto"><path d="M0,0 L9,4.5 L0,9 Z" fill="#475569"/></marker></defs>
  <g font-family="sans-serif" fill="#0f172a">
    <text x="450" y="25" text-anchor="middle" font-size="18" font-weight="700">一次搬运，多次复用</text>
    <rect x="30" y="65" width="180" height="70" rx="10" fill="#fee2e2" stroke="#dc2626" stroke-width="2"/><text x="120" y="94" text-anchor="middle" font-size="16">DRAM / memory</text><text x="120" y="116" text-anchor="middle" font-size="13">远、容量大</text>
    <rect x="355" y="65" width="190" height="70" rx="10" fill="#dcfce7" stroke="#16a34a" stroke-width="2"/><text x="450" y="94" text-anchor="middle" font-size="16">local SRAM</text><text x="450" y="116" text-anchor="middle" font-size="13">tile buffer</text>
    <rect x="690" y="65" width="180" height="70" rx="10" fill="#dbeafe" stroke="#2563eb" stroke-width="2"/><text x="780" y="94" text-anchor="middle" font-size="16">MAC / ALU</text><text x="780" y="116" text-anchor="middle" font-size="13">多次计算</text>
    <path d="M215 100 H345" stroke="#475569" stroke-width="2.5" marker-end="url(#reuse-arrow)"/><path d="M545 100 H680" stroke="#475569" stroke-width="2.5" marker-end="url(#reuse-arrow)"/>
    <text x="280" y="83" text-anchor="middle" font-size="13" fill="#475569">一次 DMA/load</text><text x="612" y="83" text-anchor="middle" font-size="13" fill="#475569">多次 reuse</text>
  </g>
</svg>

对 AI chip 来说，这个思想会进一步改变 instruction 的设计：一个 `MAC_TILE` 或 vector instruction 不只是“算得更多”，还要让一次 load 搬来的 activation 和 weight 尽可能被多次使用。于是 accelerator ISA 往往需要表达 tile shape、stride、buffer 地址、prefetch、DMA 和同步，而不只是表达一个 scalar `add`。**峰值算力只有在数据及时到达、且能被复用时才会转化为真实性能。**

### Control flow：改变 PC 的下一步

默认情况下，processor 执行下一条 instruction；control-flow instruction 会改变 `PC (Program Counter)`，让程序跳到另一段代码。

#### Conditional branch

```asm
beq x5, x6, label
bne x5, x6, label
```

如果条件成立，就跳到 `label`；否则继续顺序执行。branch 常用于 `if`、`while`、`for`。

#### Unconditional jump and link

```asm
jal ra, function
```

它会跳到 `function`，并把返回地址保存到 `ra`。函数执行完后通常使用 `jalr` 跳回。

课件例子：

```asm
jalr x1, 4(x5)
```

跳转目标地址由 register 加 immediate 计算；`x1`（也叫 `ra`）保存 return address。

![[assets/3 Instruction Set Architecture/CA4-40.png]]

#### PC-relative addressing

> [!note] immediate、地址与 PC offset
> immediate 不是 memory 地址本身，而是 instruction 里随身携带的常数或偏移。`ld x5, 16(x2)` 中，`16` 要先与 `x2` 相加得到 effective address；`beq` 的 immediate 则是相对 PC 的 branch offset。相同的“数字字段”因为所在 instruction 的语义不同，最后参与的硬件动作也不同。

branch/jump 的 offset 位数有限：课件指出 branch 使用 13-bit offset，`jal` 使用 21-bit offset。常见做法是把目标地址写成相对于当前 `PC` 的距离：

$$
\text{target address} = \text{PC} + \text{offset}
$$

这样代码移动到 memory 的另一位置后，附近的 branch 仍然可以工作；目标太远时，assembler 可以组合 `auipc` 等 instruction。

**地址算例：**假设当前 branch 的地址（也就是本条指令的 PC）是 `0x1000`，汇编器算出的目标在 `0x1010`，那么位移是 `0x1010 - 0x1000 = 0x10` bytes，即 16 bytes。基本 RISC-V 指令长 4 bytes，因此这是向前跨过 4 条指令的位置。目标地址为 `PC + offset`，不是“PC 加指令条数”；编码时 branch immediate 的最低位固定为 0（指令对齐），字段还会拆开放置，不能把汇编显示的 `16` 直接当成某一个连续字段。真实偏移范围和边界要以具体 ISA format 为准。

## 8. 从 assembly 到 machine code：instruction encoding

> [!example] 从操作需求反推字段
> 看到一条 RISC-V instruction 时，先问它要几个 register、要不要 immediate、结果是写回 register 还是改变 memory/PC。`add x5, x6, x7` 需要两个 source 和一个 destination，所以 R-type 要提供 `rs1`、`rs2`、`rd`；`addi x5, x6, 12` 只需要一个 source，但还需要 immediate，所以变成 I-type。字段布局是在给“硬件动作”分配固定位置。

**设计动机：**processor（处理器）最终读取的是 bits，而不是人类可读的 mnemonic（助记符）；但一条固定长度 instruction（指令）又要容纳 register（寄存器）、immediate（立即数）、offset（偏移量）和 operation（操作）。RISC-V 把不同需求分到少数 format（格式），并尽量固定 register field（寄存器字段）的位置，换取更简单的 decode wiring（译码连线）。代价是 immediate 会受位数限制，需要 `lui`、`auipc` 或多条 instruction 补足；理解这个取舍后，后面的字段图就不只是背位数。

**工程师视角：**hardware engineer 看的是字段位置能否复用 wiring、减少 mux 和 decode delay；compiler engineer 看的是 immediate 范围、branch 距离和 pseudo-instruction 是否会增加 instruction 数；verification engineer 看的是每种 format 的非法组合、sign extension 和 exception 行为能否穷举验证。

### 交互探索：instruction word 的字段

问题：为什么不同类型的 RISC-V instruction 要把 32 bits 拆成不同字段？在交互页中切换 `R-type`、`I-type`、`S-type` 和 `B-type`，观察寄存器、opcode 与 immediate 的位置如何变化。

[[assets/3 Instruction Set Architecture/interactive/encoding-explorer.html|打开 instruction encoding 交互探索器]]

静态结论：不同 instruction format 是为了同时容纳不同操作所需的信息；`S-type` 和 `B-type` 的 immediate 被拆开，是编码空间与硬件字段位置之间的设计取舍。交互页是字段角色示意，不是完整 assembler。

### Encoding 的全流程：processor 怎样读懂一串 bits

先想象一个很具体的问题：instruction memory 只交给 CPU 一个 32-bit 数，例如 `0x00A98933`。这 32 个 bits 本身不会写着“这是加法”或“结果放 x18”。CPU 必须遵循 ISA 规定的 encoding，把 bits 切成字段，再由 decoder 解释字段的组合。

因此 `instruction encoding（指令编码）` 就是一份紧凑的约定：**哪些 bits 表示操作种类，哪些 bits 表示寄存器编号，哪些 bits 表示常数或跳转距离。**Assembly（汇编语言）是给人看的名字；machine code（机器码）是同一条指令按这份约定写成的 bits。

初学时按固定顺序解码最稳：①把十六进制展开成 32 个 bits；②按 format 切出 opcode、寄存器号和功能字段；③先用 opcode 判定大类；④再用 `funct3/funct7` 确定具体 ALU 操作；⑤最后把 `rs1/rs2/rd` 映射回寄存器名。寄存器字段里的 `18` 是寄存器编号 `x18`，不是数值 18，也不是 memory 地址。用这套顺序读 `0x00A98933`，先认出 R-type 与 `add`，再得到目的寄存器 `x18`、源寄存器 `x19` 和 `x10`，最终汇编为 `add x18, x19, x10`。先分类再解释，避免看到一段 bits 就猜它的意思。

```text
人写 assembly                  processor 读取
add x18, x19, x10  --assembler--> 00000000101010011000100100110011
                                      │
                                      └─ decoder 按格式切字段，再控制 datapath
```

#### 第一步：从十进制寄存器编号理解字段

RV32I 有 32 个 general-purpose registers，编号为 `x0` 到 `x31`。要区分 32 个对象，至少需要 5 bits，因为 $2^5=32$。所以 instruction 中的 `rd`、`rs1`、`rs2` 字段通常各占 5 bits；字段存的是寄存器**编号**，不是寄存器里的数据。

以课件指令为例：

```asm
add x18, x19, x10
```

它表达的是：从 `x19` 与 `x10` 读取值，做加法，再把结果写到 `x18`。RISC-V 的 R-type 32-bit 排列如下，位编号从右往左，`bit 0` 是最低位：

```text
 31       25 24     20 19     15 14   12 11      7 6       0
+----------+---------+---------+-------+----------+---------+
| funct7   | rs2     | rs1     |funct3 | rd       | opcode  |
+----------+---------+---------+-------+----------+---------+
|  7 bits  | 5 bits  | 5 bits  |3 bits | 5 bits   | 7 bits  |
+----------+---------+---------+-------+----------+---------+
```

总数是 $7+5+5+3+5+7=32$ bits。位置不是随机的：`rs1`、`rs2`、`rd` 在多种格式中尽量待在相同位置，decoder 可以从固定位置取寄存器编号，简化连线与控制逻辑。

![[assets/3 Instruction Set Architecture/CA4-46.png]]

#### 第二步：把课件的十六进制机器码真的拆开

课件给出的机器码是 `0x00A98933`。每个 hexadecimal digit（十六进制数字）代表 4 bits，所以 8 个 hex digits 正好是 32 bits：

```text
0    0    A    9    8    9    3    3
0000 0000 1010 1001 1000 1001 0011 0011
```

按照上面的 bit ranges 切割：

| 字段 | 从机器码取出的 bits | 二进制值 | 十进制/解释 |
|---|---|---:|---|
| `opcode` `[6:0]` | `0110011` | `0x33` | R-type integer register operation |
| `rd` `[11:7]` | `10010` | 18 | 写入 `x18` |
| `funct3` `[14:12]` | `000` | 0 | 与其他功能字段一起识别具体 operation |
| `rs1` `[19:15]` | `10011` | 19 | 第一个输入 `x19` |
| `rs2` `[24:20]` | `01010` | 10 | 第二个输入 `x10` |
| `funct7` `[31:25]` | `0000000` | 0 | 区分 `add` 与某些相近操作，例如 `sub` |

对 `add` 来说，字段组合 `opcode=0x33, funct3=000, funct7=0000000` 表示加法。因此完整解码结果是 `add x18, x19, x10`。**解码顺序**可以记成：先看 opcode 找到大类，再结合 `funct3/funct7` 确认具体操作，最后用 register fields 找 operand 和目的地。

#### 第三步：为什么需要多种 instruction format？

不是每条指令都需要两个输入 register。`add` 要两个 register；`addi` 要一个 register 加一个常数；`sw` 要两个 register 和一个地址偏移；`beq` 要比较两个 register 并带一个分支目标。32 bits 空间有限，格式必须随指令的工作内容变化。

| Format | 例子 | 人类要表达的内容 | 为什么需要这种形状 |
|---|---|---|---|
| R-type | `add x5,x6,x7` | 两个 register 输入，一个 register 输出 | 两个输入寄存器都要编码 |
| I-type | `addi x5,x6,-3`、`lw x5,8(x6)` | 一个 register 加常数/地址偏移 | 用 12-bit immediate 代替 `rs2` |
| S-type | `sw x7,8(x6)` | 将 `x7` 写入 `x6+8` 的 memory | store 不产生 `rd`，要编码数据源和地址 |
| B-type（课件写 SB-type） | `beq x5,x6,label` | 比较两个 register，并表示 branch offset | 没有普通 `rd`，位数用来编码目标偏移 |
| U-type | `lui x5,imm20` | 一大段高位 immediate 与目的 register | 用于构造较大的常数或地址 |
| J-type（课件写 UJ-type） | `jal x1,label` | jump offset 与 return-address register | 需要相对较远的跳转范围 |

常见误区是把这些名字当作不同的“指令种类”。它们首先是**字段布局格式**；具体执行什么仍由 opcode 和功能字段决定。

#### I-type：12-bit immediate 怎样变成有正有负的整数？

`immediate`（立即数）是直接写在 instruction 里的小常数，不需要先从第二个 register 读取。I-type 把 bits `[31:20]` 用作 12-bit signed immediate：

```text
31                 20 19     15 14   12 11      7 6       0
+-------------------+---------+-------+----------+---------+
| imm[11:0]         | rs1     |funct3 | rd       | opcode  |
+-------------------+---------+-------+----------+---------+
```

12-bit two's complement 的范围是 $-2^{11}$ 到 $2^{11}-1$，也就是 $-2048$ 到 $2047$。例如 `addi x5, x6, -3` 中的 $-3$ 要用 12-bit 补码编码：

```text
+3  = 0000 0000 0011
取反 = 1111 1111 1100
+1  = 1111 1111 1101
-3  = 1111 1111 1101  (12-bit immediate)
```

decoder 取出 immediate 后要做 `sign extension`，把最高位复制到 register 的高位；因此在 RV32 中这个 `-3` 会扩展为 `0xFFFFFFFD`，而不是正数 4093。课件特别提醒：`addi x5, x6, 2048` 超出范围，不能直接用一条 I-type 编码。

![[assets/3 Instruction Set Architecture/CA4-50.png]]

**一个小特例：shift immediate。**RV32 register 有 32 bits，移位量只可能是 0 到 31，因此 `slli`/`srli`/`srai` 只需要 5-bit `shamt`。它们仍使用 I-type 外观，但 immediate 的高位用于区分 shift 操作（例如 logical right shift 与 arithmetic right shift），decoder 因而能复用 I-type 的字段位置。

![[assets/3 Instruction Set Architecture/CA4-51.png]]

#### S-type：为什么 store 的 immediate 被切成两段？

`sw x7, 8(x6)` 的意思是“把 `x7` 的低 32 bits 写到 memory 地址 `x6+8`”。store 有地址基址 `rs1` 和要写出的数据 `rs2`，但没有目的 register `rd`。若把 offset 放在原本 R-type 的 `rd` 和 `funct7` 位置，就能保留 `rs1`、`rs2` 在固定的位置。代价是 immediate 被拆为两段：

```text
31       25 24     20 19     15 14   12 11      7 6       0
+----------+---------+---------+-------+----------+---------+
| imm[11:5]| rs2     | rs1     |funct3 | imm[4:0] | opcode  |
+----------+---------+---------+-------+----------+---------+
```

`imm[11:5]` 和 `imm[4:0]` 在 decoder 内重新拼成一个 12-bit signed offset，再加到 `x6`。图上看起来不连续，是为了让 source register 的位置与其他格式保持一致；不是 CPU 把地址偏移理解成两个独立数字。

#### B-type：分支目标为什么也要拆位，并且最低位隐含为 0？

branch instruction 的目标通常按 instruction 对齐，课件中的 RV32I branch offset 至少以 2 bytes 为单位。于是 offset 最低位必定为 0，不必浪费一个 instruction bit 来存它；编码把其余 offset bits 分散在 instruction 的立即数字段位置。处理器解码时再重组、补上最低的 0、符号扩展，最后计算：

$$
\text{branch target} = \text{current PC} + \text{sign-extended offset}
$$

例如若分支 offset 是 $+12$ bytes，编码时最低位 `0` 隐含，剩余值对应 6 个 2-byte units；执行时恢复成 12，再加到当前 `PC`。正负 offset 都可表达，所以 branch 可以跳向前方，也可以跳回前方的 loop 起点。

![[assets/3 Instruction Set Architecture/CA4-52.png]]

#### U-type 和 J-type：把有限的立即数字段用于更远的目标

`U-type`（`lui`、`auipc`）把 immediate 放在 instruction 高 20 bits，低 12 bits 视为 0。直觉上，它一次先搭出一个“高位底座”：

```text
lui x5, 0x12345
x5 = 0x12345 << 12 = 0x12345000
```

之后可用 `addi` 补低位。`auipc` 相似，但底座加在当前 `PC` 上，适合构造 PC-relative 地址。

`J-type` 的 `jal rd,label` 要同时保存返回地址 `PC+4` 并跳转。它使用比 branch 更宽的 offset（课件按 byte 计为 21 bits），所以能跳得更远；目标仍以当前 `PC` 为基准。跳得很远、超出单条 instruction 范围时，需要 assembler 展开成 `auipc` 等多条真实 instruction。

![[assets/3 Instruction Set Architecture/CA4-53.png]]

#### 一个地址例子：PC-relative 到底怎么算？

假设当前 instruction 的 `PC = 0x1000`，branch 的目标在 `0x1018`，那么 assembler 计算 offset：

$$
\text{offset} = 0x1018 - 0x1000 = 0x18 = 24\text{ bytes}
$$

CPU 取得 branch instruction 后重建这个 offset；若比较条件成立，就令 `PC = 0x1000 + 24 = 0x1018`，否则走顺序路径 `PC+4`。因此机器码存的不是绝对目标 `0x1018`，而是相对当前位置的距离。代码整体搬到另一个位置时，只要 branch 和目标一起移动，两者之间距离不变，机器码仍可工作。

![[assets/3 Instruction Set Architecture/CA4-54.png]]

#### Assembler、pseudo-instruction 与真实 machine instruction

人写 `li x5, 0x12345678` 很方便，但 RV32I 没有一条能装下任意 32-bit 常数的普通 instruction。`li` 是 `pseudo-instruction`（伪指令）：它是 assembler 提供的人类友好写法，不一定对应独立 opcode。Assembler 会按常数大小展开成 `lui`、`addi` 等真实指令；具体展开形式可能依 assembler、relocation 和目标地址而不同。

这条转换链要分清三层：

```text
source code:       x = 3 + 4
assembly:          li x5, 7       # 可能是伪指令
real instruction:  addi x5, x0, 7
machine code:      32-bit encoding consumed by decoder
```

从 assembly 到 machine code 的工具叫 `assembler`；`compiler` 通常先把高级语言翻译为 assembly 或 machine code，还会做 register allocation 等工作。读懂这一点后，看到 `mv`、`li`、`ret` 等写法时就不会误以为 processor 一定有同名硬件指令。

![[assets/3 Instruction Set Architecture/CA4-58.png]]

#### 把 encoding 的设计选择连回硬件

为什么不把每种格式都设计得完全不一样，让每条指令最大限度塞进有用信息？因为 decoder 会更复杂。RISC-V 尽量让 `rs1`、`rs2`、`rd` 固定在相似的位置，opcode 也固定在低位；于是 register file 的地址线可以从 instruction 固定位置接入，decode 不必先经过很多动态选择。

代价是 S/B/J immediate 有时需要拆位、重排和 sign extension，某些大常数也要多条 instruction。这是编码设计的取舍：**多花一点 immediate 重组逻辑，换更规律的 decode 和 datapath 控制。**课件的 design principle 是 “Simplicity favors regularity”；这不是为了让 instruction 看起来整齐，而是让硬件少做格式猜测。

![[assets/3 Instruction Set Architecture/CA4-56.png]]

![[assets/3 Instruction Set Architecture/CA4-55.png]]

**读一条 RISC-V instruction 的完整心法：**先根据 opcode 判断大类；再找 format；按 bit 编号切出 register、function 与 immediate；对 immediate 按规则重组并 sign-extend；最后说清 operand 从哪里来、做什么、结果或 PC 去哪里。这样 encoding 就不再是一张需要死背的字段表，而是一份 CPU 和 assembler 共同执行的解码协议。

![[assets/3 Instruction Set Architecture/CA4-28.png]]

![[assets/3 Instruction Set Architecture/CA4-44.png]]

### 大立即数与长距离地址：`lui`、`auipc`

上面的 encoding 例子已经解释了为什么单条 I-type immediate 放不下任意大的常数。RISC-V 把构造工作分成几步：

- `lui rd, imm20`（`Load Upper Immediate`）：将 20-bit immediate 放到 register 高位，等价于 `rd = imm20 << 12`；
- `auipc rd, imm20`（`Add Upper Immediate to PC`）：将高位 immediate 左移 12 bits 后加到当前 `PC`，等价于 `rd = PC + (imm20 << 12)`。

Assembler 根据目标常数或地址产生合适的 instruction 序列。大常数例子和 PC-relative address 例子分别体现“编码长度有限”与“代码可重定位”的设计目标。

![[assets/3 Instruction Set Architecture/CA4-42.png]]

## 9. 设计原则：规则性是为了让硬件更容易实现

课件最后把 RISC-V 的编码选择归纳成三条工程原则：

1. **Simplicity favors regularity**：固定的字段位置、固定的 operand 数量，让 decoder 和 wiring 可以复用；
2. **Smaller is faster**：register file、decoder 或 datapath 变得很大时，信号传播和访问时间也会增加；
3. **Good design demands good compromises**：统一 32-bit instruction 简化 decode，但也意味着要用不同 format 容纳 immediate、store offset 和 jump offset。

这不是“越简单越好”，而是在 instruction memory、decoder complexity、compiler 便利性与可表达范围之间做取舍。对 AI chip 也一样：给矩阵乘法加一条 custom instruction 可能减少 software overhead，但会增加 encoding、验证、pipeline hazard 和 compiler lowering 的成本。

![[assets/3 Instruction Set Architecture/CA4-55.png]]

## 10. Register file、memory hierarchy 与数据搬运

**设计动机：**ALU 想马上得到 operand，但大容量 memory 不可能同时做到近、快、低能耗；如果每次加法都去远端 memory 取数据，计算单元会等待，带宽和能量也会被搬运消耗。系统因此使用 register、cache、local SRAM、DRAM 等层次，并让 ISA 明确区分 register operation 与 load/store。收益是可以显式安排 locality 和 reuse，代价是 software/hardware 必须共同管理地址、prefetch、DMA 和容量边界；这会直接连接到后面的 AI accelerator dataflow。

**工程师视角：**memory/system engineer 首先看 bandwidth、queue 和一致性；accelerator architect 看一个 tile 能否在 local SRAM 中复用；performance/energy engineer 则会问增加一个 buffer 是否真的减少了远端访问，还是只增加了控制和搬运开销。对初学者来说，先记住一句经验：很多系统不是算不动，而是等数据。

ISA 不是只记 instruction 名字。对硬件设计者而言，更重要的是每条 instruction 如何使用 storage。先把 storage 层次的轮廓放在眼前，再区分 register、register file、SRAM、cache 和 DRAM：

可以把 memory hierarchy 读成一个取舍图：越靠近 CPU，容量通常越小、速度越快、每 bit 成本越高；越远离 CPU，容量更大，但访问 latency 和搬运能耗更高。

![[assets/3 Instruction Set Architecture/CA4-47.png]]

`load/store` 把 memory hierarchy 与 register file 接起来；如果同一数据可以留在 register file 中复用，就能减少 memory traffic。对 AI chip 来说，这个原则更明显：很多 workload 不是算力不足，而是 activation、weight 和 partial sum 来回搬运造成 bandwidth 与 energy bottleneck。

![[assets/3 Instruction Set Architecture/CA4-45.png]]

这张课件图的重点是：R-type instruction 的 operand 先从 register file 读出，再进入 ALU，最后写回 register file。它是后面理解 AI accelerator 的一个小模型——把 register file 换成 local SRAM，把 ALU 换成 vector/MAC array，仍然要解决“数据从哪里来、在哪里计算、结果写到哪里”。

### Register 到底是什么？

先从一个最小问题开始：CPU 要做 `a + b` 时，两个数放在哪里，才能让加法器马上拿到它们？CPU 内部有一小组可以直接读写的存储位置，叫 `register`（寄存器）。可以把每个 register 想成 processor 手边的一个小格子：格子有编号，里面保存固定宽度的 bits；指令通过编号指定要读哪个格子、把结果写回哪个格子。

例如 `add x5, x6, x7`：`x6` 与 `x7` 是两个 register 的编号，register file 读出它们当前保存的 bit values，ALU 对这些值求和，再把结果写进 `x5`。`x6` 不是值本身，`x6` 里面的 bits 才是值；类似“抽屉编号”和“抽屉里的东西”的区别。

```text
instruction 中的编号       register 里保存的值
rs1 = x6  ───────────────→ 读取 x6 的内容 ──┐
                                             ├─ ALU 做加法 ─→ 写入 x5
rs2 = x7  ───────────────→ 读取 x7 的内容 ──┘
```

`register file`（寄存器堆）是这组 registers 连同读写电路的整体名字。它不等于“一个 register”：RV32I 有 32 个 architectural integer registers，每个宽 32 bits；RV64I 有 32 个，每个宽 64 bits。光存储位数分别是 $32\times32=1024$ bits 和 $32\times64=2048$ bits，实际电路还需要 decoder、read mux、写入选择和端口线路。

![[assets/3 Instruction Set Architecture/CA4-48.png]]

看课件图左半边：写地址经过 decoder 选中某个 register，右边两个 read-address 输入各自选择一份 register 内容，输出 `OutA`、`OutB` 供 ALU 使用。这张图说明 register file 通常要支持“同时读两个 operand，再写回一个结果”，因为 `add rd, rs1, rs2` 需要两个输入和一个输出。多一个读写 port 会增加电路面积、功耗和布线，所以 register file 不能无限扩张。

右半边的 SRAM cell 图与 register file 不是同一个层次：SRAM cell 是一种保存 bit 的电路单元；register file 则是把很多存储单元组织起来，并加上地址译码、读写端口后得到的结构。现代处理器内部具体用什么电路实现 register file，依 microarchitecture 而定。课件用 6T/10T SRAM cell 说明读写端口需要真实 transistor 和连线，因此“多开几个 port”有硬件代价。

### Register、cache、SRAM 与 DRAM 的区别

这些词经常一起出现，初学时容易误以为它们只是大小不同的同一种东西。可以先这样分层理解：

| 名称 | 它是什么 | 谁通常决定读写哪个位置？ | 对执行的作用 |
|---|---|---|---|
| `register` | ISA 可直接命名的少量工作格子，例如 `x5` | instruction 明确写出 register 编号 | ALU 的直接 operand；访问近、快，但数量有限 |
| `register file` | 一组 registers 加上读写选择电路 | decoder 从 instruction 取 `rs1/rs2/rd` | 支持指令读 operand、写结果 |
| `SRAM` | 一种静态随机存取存储电路技术 | 由地址和控制电路选择 | 常用于 cache、scratchpad/local SRAM，也可实现某些 register file |
| `cache` | 自动保存近期 memory 内容的层次/机制 | 主要由硬件自动管理 | 命中时减少去较远 memory 取数，但程序通常不直接点名某个 cache line 来做 `add` |
| `DRAM` | 主存常用的动态存储技术 | 由 memory controller 和地址选择 | 容量大，用来放程序和数据；访问路径更远 |

所以 `register` 是 processor ISA/执行模型中能直接点名的工作存储；`SRAM` 是一种电路实现技术；`cache` 是管理 memory contents 的结构；`DRAM` 是主存常见的实现技术。这些概念有关联，但不是同义词。

### 一条指令怎样用 register：读、算、写

回到课件机器码例子 `add x18, x19, x10`。它不表示“到 memory 的地址 18 取数”，而表示访问 register file 中编号为 19、10 和 18 的三个寄存器：

1. `rs1=x19` 和 `rs2=x10` 指定两个 read address；
2. register file 输出当前 `x19` 与 `x10` 的值；
3. ALU 执行 add；
4. `rd=x18` 指定 write address，结果写入 `x18`。

若当前 `x19=7`、`x10=5`，执行后 `x18=12`；原来的 `x19`、`x10` 不会因此改变。对应的 R-type instruction 在编码中的三个 5-bit 字段只保存编号，这就是前一节拆解机器码时看到的 `rs1`、`rs2` 和 `rd`。

`load` 和 `store` 则把 register file 与 memory 接起来：

```asm
lw x5, 8(x6)     # 从 memory 地址 x6+8 读一个 word，放入 x5
sw x5, 8(x6)     # 把 x5 的 word 写到 memory 地址 x6+8
```

`lw` 的数据方向是 memory → register；`sw` 是 register → memory。普通 `add` 不会自动去 memory 找 operand，因此若数据还在 DRAM，必须先通过 load/cache 路径把它带到处理器能使用的位置。

### `x0`、`x1`、`a0`、`t0` 这些名字有什么不同？

`x0` 到 `x31` 是 RISC-V integer registers 的硬件编号。ABI 为一些编号提供更好记的 symbolic names（符号名）：

| 硬件编号 | 常见别名 | 初学时先记住的用途 |
|---|---|---|
| `x0` | `zero` | 永远读作 0；写入它的结果会被丢弃 |
| `x1` | `ra` | return address，函数返回地址 |
| `x2` | `sp` | stack pointer，栈指针 |
| `x5`–`x7` 等 | `t0`–`t6` | temporary，临时值 |
| `x10`–`x17` | `a0`–`a7` | arguments，函数参数；也用于返回值 |
| `x8`–`x9`、`x18`–`x27` | `s0`–`s11` | saved registers，按调用约定需要保留的值 |

别名不会增加新的存储格子：`a0` 就是 `x10`，`ra` 就是 `x1`。硬件看的是 register number；compiler/programmer 使用别名帮助理解用途。`zero` 则是特殊硬件行为，不是普通的可写存储：不管某条指令把结果写到 `x0`，读 `x0` 仍然得到 0。

**特别区分：硬件编号与软件约定。**`x0` 固定为 0 是 ISA 规定的硬件行为；`a0` 用来传参数、`t0` 是 temporary、`s0` 要由 callee 保存，是 ABI（software calling convention）的约定。后者让不同 compiler 编译的 function 能合作，不代表 register 电路天生知道 `a0` 是参数。

### 为什么 register 少而快，而不是做得很多？

register 离 ALU 很近，processor 可从少量已知位置快速取 operand；但 register file 有物理面积、线路长度和读写端口成本。register 越多，指令编码通常需要更多 bits 来表示编号，read mux 也要在更多来源中选择，布局布线距离可能变长。因此工程上不会把整个 memory 都做成 register，而是让少量 register 放眼前正在计算的数据，让 cache/SRAM/DRAM 分层提供更大容量。

对 compiler 来说，有限 register 也意味着必须安排哪些值暂时留在寄存器。若同时活跃的值太多，compiler 可能把一些 register value `spill`（溢出保存）到 stack memory，需要时再 load 回来。这个词不是算术 overflow：`spill` 是资源不足时把临时值存到 memory；`overflow` 是数值超出 bit width 的表示范围。

从硬件设计视角，register 数量和端口数共同影响 timing、area 与 power；从 compiler 视角，register allocation 决定哪些数据能留在近端、哪些要反复 spill/load。理解这些限制后，`Why Registers Matter` 的重点就更清楚：register 不是因为“快”就有价值，而是因为它让 processor 能在有限的近端 storage 中保留当前需要重复使用的 operand 与 intermediate result。

![[assets/3 Instruction Set Architecture/CA4-49.png]]

### Why Registers Matter：关键不是“寄存器小而快”，而是 data reuse

课件第 49 页把重点说得很直接：register 是 CPU 中最快的 data storage，理想情况下应尽量复用已经进入 register file 的数据。真正值得记住的因果链是：

```text
数据从远端 memory 搬到 processor
        ↓
放进 register / local SRAM
        ↓
同一份数据被多次计算（data reuse）
        ↓
少做 load、少占 bandwidth、少等 memory、少消耗搬运能量
```

`Data reuse` 指同一个数据被多次 computation 使用，而不必每次都重新从更远的 memory hierarchy 读取。它和两个相关概念不同：`temporal locality` 是“同一数据很快再次被用”，`spatial locality` 是“相邻地址的数据可能一起被用”；二者都可以帮助实现 reuse，但 reuse 关注的是**一次搬运究竟支撑了多少次有用计算**。

![[assets/3 Instruction Set Architecture/CA4-47.png]]

上图的 memory hierarchy 从上到下通常越来越大、越来越远；越靠近 processor 的 storage 越适合放高频复用的数据。register file 的容量很小、读写端口也有限，所以设计目标不是“把所有数据放进 register”，而是把当前 computation 最需要的 operand 和 intermediate result 留在最近的位置。

#### 例子 1：向量加法中，register reuse 减少了什么？

假设要计算长度为 4 的向量：

$$
\mathbf{y}=\mathbf{a}+\mathbf{b}
$$

一种没有 reuse 的想法是每做一次加法，就重新从 memory 读取 `a[i]`、`b[i]`，并把 `y[i]` 写回：

```text
for i = 0..3:
    load a[i] from memory
    load b[i] from memory
    y[i] = a[i] + b[i]
    store y[i] to memory
```

这需要 8 次 load（读取两个输入向量）和 4 次 store（写回结果），合计 12 次 memory transfer，以及 4 次加法。若处理器先把一小段数据放入 registers，代码可以变成：

```asm
lw   t0, 0(a0)       # a[0]
lw   t1, 0(a1)       # b[0]
add  t2, t0, t1       # y[0]
sw   t2, 0(a2)

lw   t0, 4(a0)       # a[1]
lw   t1, 4(a1)       # b[1]
add  t2, t0, t1
sw   t2, 4(a2)
```

这里每个元素只被使用一次，所以 register reuse 很有限；它主要展示了 `load → register → ALU → store` 的路径。更明显的收益出现在一个数据被多个 operation 重用时。比如计算

$$
y_i = a_i \times w + b_i \quad (i=0,1,2,3)
$$

标量 `w` 和 `b` 可以只 load 一次，随后在 4 次迭代中一直留在 registers：

```asm
lw   t4, 0(w_addr)   # w：只搬一次
lw   t5, 0(b_addr)   # b：只搬一次

lw   t0, 0(a0)
mul  t1, t0, t4      # 复用 t4
add  t2, t1, t5      # 复用 t5
sw   t2, 0(y_addr)

lw   t0, 4(a0)
mul  t1, t0, t4      # w 没有再次 load
add  t2, t1, t5      # b 没有再次 load
sw   t2, 4(y_addr)
```

这就是 register 的直接价值：`w`、`b` 的一次 load 支撑了多次 ALU operation。若 `w` 原本位于更远的 cache 或 DRAM，省下的不只是两条 assembly，而是两次潜在的 cache lookup、总线传输和等待时间。

#### 例子 2：矩阵乘法为什么更依赖 reuse？

AI chip 常见的矩阵乘法可以写成：

$$
C_{ij}=\sum_{k=0}^{K-1}A_{ik}B_{kj}
$$

对一个固定的 $C_{ij}$，每个 $A_{ik}$ 和 $B_{kj}$ 都会参与一次乘法；但当我们同时计算一小块 $C$ 时，同一个 `A` 元素会被多个 `B` 元素复用，同一个 `B` 元素也会被多个 `A` 元素复用。计算 $2\times2$ 的 tile 时：

```text
A tile:                 B tile:
[ a00 a01 ]             [ b00 b01 ]
[ a10 a11 ]             [ b10 b11 ]

C00 = a00*b00 + a01*b10
C01 = a00*b01 + a01*b11
C10 = a10*b00 + a11*b10
C11 = a10*b01 + a11*b11
```

如果按最笨的方式逐个计算 $C_{ij}$，相同的 `a00`、`a01`、`b00` 等数据会被反复从 memory 读取。若先把这个 tile 搬到 local SRAM，再把当前行/列放进 registers 或 MAC array 附近的 buffer：

```text
DRAM / cache
      │  一次搬入 A、B tile
      ▼
local SRAM / scratchpad
      │  分块读取，保持 partial sum 在 register
      ▼
MAC array
      │  A、B 多次复用；Cij 累加多次
      ▼
store C tile
```

同一批数据由一次或少数几次搬运支撑许多次 multiply-accumulate，这就是 `tiling` 或 `blocking` 背后的基本思想。对于 AI accelerator，register、local SRAM 和 `partial sum` 的位置会共同决定 `MAC utilization`：如果 partial sum 每次都写回远端 memory，计算阵列就可能在等待数据，而不是持续计算。

数一数这个 $2\times2$ 例子会更直观：直接逐个算 4 个输出，每个输出需要 2 次乘法，因此共有 8 次乘法，并读取 16 次乘法输入（每次乘法各读一个 $A$ 和一个 $B$）。若把两个 $2\times2$ tile 各搬入近端一次，则只需搬入 4 个 $A$ 元素和 4 个 $B$ 元素，共 8 个输入元素；每个输入会在近端被使用两次。实际硬件还会读取/写回 partial sum、受 tile 容量和端口数限制，但这个最小例子清楚展示了 reuse 如何把 input traffic 减半。

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 980 330" role="img" aria-label="processor 通过 register 和 local SRAM 复用数据的路径：memory 只搬入一次，数据在近端被多次使用" style="width:100%;height:auto;background:#f8fafc;border-radius:12px">
  <defs><marker id="reuse-flow-arrow" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto"><path d="M0,0 L10,5 L0,10 Z" fill="#334155"/></marker></defs>
  <g font-family="sans-serif" fill="#0f172a">
    <text x="490" y="28" text-anchor="middle" font-size="20" font-weight="700">Data reuse：一次搬运，近端多次计算</text>
    <rect x="35" y="88" width="190" height="96" rx="12" fill="#fee2e2" stroke="#dc2626" stroke-width="2"/>
    <text x="130" y="123" text-anchor="middle" font-size="17">DRAM / cache</text><text x="130" y="150" text-anchor="middle" font-size="14">容量大，距离远</text><text x="130" y="170" text-anchor="middle" font-size="14">latency / energy 高</text>
    <rect x="295" y="88" width="205" height="96" rx="12" fill="#dcfce7" stroke="#16a34a" stroke-width="2"/>
    <text x="397" y="123" text-anchor="middle" font-size="17">local SRAM</text><text x="397" y="150" text-anchor="middle" font-size="14">保存 tile / buffer</text><text x="397" y="170" text-anchor="middle" font-size="14">批量搬入、反复读取</text>
    <rect x="570" y="88" width="170" height="96" rx="12" fill="#dbeafe" stroke="#2563eb" stroke-width="2"/>
    <text x="655" y="123" text-anchor="middle" font-size="17">register file</text><text x="655" y="150" text-anchor="middle" font-size="14">当前 operand</text><text x="655" y="170" text-anchor="middle" font-size="14">短距离、低延迟</text>
    <rect x="810" y="88" width="135" height="96" rx="12" fill="#fef3c7" stroke="#d97706" stroke-width="2"/>
    <text x="877" y="123" text-anchor="middle" font-size="17">ALU / MAC</text><text x="877" y="150" text-anchor="middle" font-size="14">多次计算</text><text x="877" y="170" text-anchor="middle" font-size="14">partial sum</text>
    <path d="M225 136 H285" stroke="#334155" stroke-width="3" marker-end="url(#reuse-flow-arrow)"/><path d="M500 136 H560" stroke="#334155" stroke-width="3" marker-end="url(#reuse-flow-arrow)"/><path d="M740 136 H800" stroke="#334155" stroke-width="3" marker-end="url(#reuse-flow-arrow)"/>
    <text x="255" y="116" text-anchor="middle" font-size="13" fill="#334155">DMA / load</text><text x="530" y="116" text-anchor="middle" font-size="13" fill="#334155">tile reuse</text><text x="770" y="116" text-anchor="middle" font-size="13" fill="#334155">operand reuse</text>
    <path d="M877 205 C877 285 397 285 397 205" fill="none" stroke="#7c3aed" stroke-width="3" stroke-dasharray="8 6" marker-end="url(#reuse-flow-arrow)"/>
    <text x="635" y="305" text-anchor="middle" font-size="15" fill="#6d28d9">partial sum / tile 数据尽量留在近端，避免每次写回远端 memory</text>
  </g>
</svg>

读图时，实线是数据第一次向 processor 靠近的路径；紫色虚线表示计算结果在近端继续累加和复用。register 并不是“存得越多越好”：它的面积、端口和布线也有成本，因此真正的设计问题是“哪些数据值得留在 register，哪些数据放到 local SRAM，什么时候批量搬运”。

#### Processor 与 memory 的一个数量级直觉

**Background（外部扩展）：**不同处理器、cache 命中状态、工艺和访问模式会改变具体数字，因此不要把下面的数量级当成固定常数。更稳妥的结论是：register 通常比 cache/DRAM 更近，且数据搬运常常比一次简单 ALU operation 更昂贵。NVIDIA 的 CUDA Best Practices Guide 将 `global memory` 访问效率和减少不必要的 global memory traffic 列为性能优化重点；Eyeriss 论文则从 AI accelerator 的 dataflow 角度分析 local storage、数据复用和 data movement energy 的关系。

| 数据位置 | processor 看到的典型角色 | 访问后最值得问的问题 |
|---|---|---|
| register | 当前 cycle/短循环的 operand、partial sum | 这个值会不会马上再次使用？能否避免 spill 到 memory？ |
| local SRAM / scratchpad | 一个 tile 的 activation、weight、partial sum | 一次搬入能支撑多少次 MAC？tile 是否放得下？ |
| cache | 由硬件自动管理的近期数据 | access pattern 是否有 locality？是否频繁 miss？ |
| DRAM / host memory | 大容量输入、权重和最终结果 | 能否批量搬运、预取和减少往返？ |

可以把一次计算的粗略成本写成：

$$
\text{总成本}\approx N_{\text{compute}}\cdot E_{\text{compute}} + N_{\text{data movement}}\cdot E_{\text{movement}}
$$

当同一个 operand 被复用 $R$ 次时，一次搬运成本可以摊到多个 operation 上；若没有 reuse，$N_{\text{data movement}}$ 会随计算次数一起增长。对 AI chip 设计而言，这也是为什么论文经常报告 `operations per byte`、`arithmetic intensity` 或 `MAC utilization`，而不仅仅报告 peak TOPS。

**外部资料：**

- NVIDIA, *CUDA C++ Best Practices Guide*, “Memory Optimizations”：https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/index.html（访问日期：2026-10-21）。用于补充 global memory traffic、coalescing 与数据复用的工程背景。
- Chen et al., *Eyeriss: An Energy-Efficient Reconfigurable Accelerator for Deep Convolutional Neural Networks*, ISCA 2016, DOI: https://doi.org/10.1145/3007787.3001177。用于补充 AI accelerator 中 dataflow、local storage 与 data movement energy 的关系。

把这个主题和 ISA 连起来：`add`、`mul`、vector/MAC instruction 描述 processor 要做什么；`load/store`、DMA、stride 和 buffer 指令描述数据怎样到达计算单元。只有 instruction 产生的计算和 storage 层次中的 data reuse 配合起来，processor 的峰值算力才不会长期等待 memory。

## 11. Assembly programming：function call、stack 与 ABI

**设计动机：**register（寄存器）数量有限，多个 function（函数）又必须共享它们；没有统一规则时，一个 function 覆盖 register 就会悄悄破坏 caller（调用者）的数据。Application Binary Interface（应用二进制接口，ABI）用 caller-save/callee-save（调用者/被调用者保存约定）分配保护责任，让独立编译的 function 可以互相调用；收益是协作规则清楚，代价是保存/恢复可能产生 stack memory traffic（栈存储器流量）。理解这个代价后，再看 AI kernel 的 function boundary 和 runtime call，就能判断哪些数据搬运来自 ABI，哪些来自 accelerator。

**工程师视角：**compiler engineer 关心调用约定是否便于寄存器分配和跨模块链接；runtime engineer 关心 function call、interrupt 和 accelerator queue 是否能共享一致的保存规则；performance engineer 则会衡量 stack spill/fill 的访存代价。ABI 因此不是“寄存器名字表”，而是软件生态可以长期协作的性能契约。

### Register convention 是软件约定

RISC-V 的 register convention 通常由 ABI（Application Binary Interface）规定。它不是硬件强制的“物理定律”，而是 compiler、library 和 programmer 共同遵守的协议。这样不同 function 才能互相调用。

需要熟悉的名字：

| symbolic name | register | 常见用途 | saver |
|---|---:|---|---|
| `zero` | `x0` | 永远为 0，写入也不会改变 | hardware fixed |
| `ra` | `x1` | return address | caller-save |
| `sp` | `x2` | stack pointer | callee-save |
| `a0`–`a7` | argument registers | function arguments / return values | caller-save |
| `t0`–`t6` | temporary registers | 临时计算 | caller-save |
| `s0`–`s11` | saved registers | 跨 function call 保持值 | callee-save |

![[assets/3 Instruction Set Architecture/CA4-59.png]]

`caller` 是发起 function call 的函数，`callee` 是被调用的函数：

- `caller-save`：callee 可以覆盖；如果 caller 在 call 之后还需要旧值，caller 必须先保存；
- `callee-save`：callee 如果要使用，就必须先保存旧值，返回前恢复。

这种分工不是为了让某一方“多做工作”，而是为了让有限的 registers 可以在很多 function 之间可靠复用。

### Stack 与 heap

`stack` 是一块用于 function call 临时数据的 memory 区域，通常按 LIFO（last-in, first-out）使用。`sp` 指向 stack top；在课件约定中，stack space 通过减少 `sp` 分配，stack 向较低地址增长。

![[assets/3 Instruction Set Architecture/CA4-63.png]]

| 对比 | stack | heap |
|---|---|---|
| 管理者 | function-call mechanism / compiler | programmer，例如 `malloc`、`free` |
| 内容 | local variables、保存的 registers、parameters | dynamic data |
| 生命周期 | 自动随 function call 管理 | 由程序显式管理 |
| 特点 | 通常更快、更规整 | 更灵活，但管理成本更高 |

![[assets/3 Instruction Set Architecture/CA4-64.png]]

### 课件例子：callee-save registers

这次我们真的“手撕”课件里的函数。C 代码是：

```c
int f(int x, int y) {
    return (x + 3) | (y + 123456);
}
```

注意 `|` 是 **bitwise OR（按位或）**：把两个数的二进制位逐位比较，有任一位为 1，结果该位就是 1。它不是普通加法。

按 RISC-V ABI，前两个参数 `x`、`y` 分别通过 `a0`、`a1` 传入，整数返回值通过 `a0` 返回。课件汇编如下：

```asm
f:
    addi sp, sp, -8       # 1. 在 stack 上留出 8 bytes
    sw   s0, 4(sp)         # 2. 先保存调用者原来的 s0
    sw   s1, 0(sp)         #    再保存调用者原来的 s1
    addi s0, a0, 3         # 3. s0 = x + 3
    li   s1, 123456        # 4. s1 = 123456 (li 是 pseudo-instruction)
    add  s1, a1, s1         #    s1 = y + 123456
    or   a0, s0, s1         # 5. a0 = (x+3) | (y+123456)，准备返回
    lw   s1, 0(sp)          # 6. 恢复调用者原来的 s1
    lw   s0, 4(sp)          #    恢复调用者原来的 s0
    addi sp, sp, 8          # 7. 释放刚才的 8 bytes，sp 回到进入时的位置
    ret                     # 返回 caller；等价于 jalr x0, 0(ra)
```

#### 先把 stack 地址画出来

假设进入 `f` 时 `sp = 0x1000`。stack 往低地址增长，所以 `addi sp, sp, -8` 后 `sp = 0x0FF8`。现在函数拥有从 `0x0FF8` 到 `0x0FFF` 的 8 bytes：

```text
高地址
0x1000  ← 进入 f 时的旧 sp；函数结束时必须恢复到这里
0x0FFC  ┌────────────────────┐
0x0FFB  │ 调用者旧 s0，4 bytes │  地址 0x0FFC = 新 sp + 4
0x0FFA  │                    │
0x0FF9  │                    │
0x0FF8  │ 调用者旧 s1，4 bytes │  地址 0x0FF8 = 新 sp + 0
        └────────────────────┘ ← 当前 sp = 0x0FF8
低地址
```

在 RV32 中一个 word 是 4 bytes，所以两个 register values 共需要 8 bytes。`sw s0, 4(sp)` 把旧 `s0` 写到地址 `sp+4`；`sw s1, 0(sp)` 把旧 `s1` 写到地址 `sp+0`。这里 stack 保存的是 register 里的旧 bit values，不是 register 编号。

#### 用具体数字跟踪每一步

取一个简单例子：调用 `f(4, 10)`。进入函数时 `a0=4`、`a1=10`；假设 caller 原来的 `s0=111`、`s1=222`。这两个旧值属于 caller 的状态，callee 虽然要借用 `s0/s1`，但必须保证返回后它们仍是 111 和 222。

| 执行位置 | `sp` | `s0` | `s1` | `a0` | 发生了什么 |
|---|---:|---:|---:|---:|---|
| 刚进入 `f` | `0x1000` | 111 | 222 | 4 | 参数在 `a0/a1`，旧 `s0/s1` 属于 caller |
| `addi sp,sp,-8` 后 | `0x0FF8` | 111 | 222 | 4 | stack frame 开出 8 bytes |
| 两条 `sw` 后 | `0x0FF8` | 111 | 222 | 4 | 111 写入 `0x0FFC`，222 写入 `0x0FF8` |
| `addi s0,a0,3` 后 | `0x0FF8` | 7 | 222 | 4 | `s0` 暂时被函数拿来存 $x+3=4+3$ |
| `li` 与 `add` 后 | `0x0FF8` | 7 | 123466 | 4 | `s1=10+123456=123466` |
| `or a0,s0,s1` 后 | `0x0FF8` | 7 | 123466 | 123471 | 结果准备放进返回寄存器 `a0` |
| 两条 `lw` 后 | `0x0FF8` | 111 | 222 | 123471 | 函数工作结束，旧 `s0/s1` 已恢复 |
| `addi sp,sp,8` 后 | `0x1000` | 111 | 222 | 123471 | stack frame 释放，`sp` 复原 |
| `ret` 后 | caller 的 stack | 111 | 222 | 123471 | 控制流返回；caller 从 `a0` 取函数结果 |

最后的按位或也手算一下。十六进制可以更容易对齐位：

```text
x + 3          = 7      = 0x00000007
y + 123456     = 123466 = 0x0001E24A
                           0x00000007
                        OR 0x0001E24A
                           ----------
result                   = 0x0001E24F = 123471
```

低位 `0xA` 是 `1010`，与 `0x7` 的 `0111` 按位 OR 得 `1111`，所以低 nibble 变成 `F`；更高位保持 `0x1E24`。函数把 123471 留在 `a0`，但把 `s0/s1` 还原为 caller 原来的值。

#### 为什么要先存、最后恢复？

`s0` 和 `s1` 的 `s` 表示 saved register。ABI 的承诺是：**如果 callee 要使用这些寄存器，它必须在返回前把旧值恢复好。**如果不保存，`f` 执行完会留下 `s0=7`、`s1=123466`，caller 原来放在这两个寄存器中的值就被悄悄覆盖，后续 caller 指令可能算错。

可以把两条边界条件当成读这段代码的检查表：

```text
进入 f：caller 的 s0/s1 = 某些旧值
离开 f：caller 的 s0/s1 必须仍是那些旧值
进入 f：sp = 旧地址
离开 f：sp 必须回到同一个旧地址
返回 f：结果放在 a0
```

保存和恢复可以按镜像过程看：`sw` 把寄存器值存到 stack，之后 `lw` 从相同地址取回；先开 stack frame，再释放它。只要函数要覆盖 callee-saved register，就必须确保所有正常返回路径都执行恢复。

![[assets/3 Instruction Set Architecture/CA4-65.png]]

**小心课件例子的边界：**这里的 8-byte stack frame 是为了让你看清两个 32-bit register 的保存过程。真实 RISC-V ABI 对 stack alignment 有更严格的要求（常见标准 ABI 要求 16-byte 对齐），完整函数还要处理可能的 function calls、更多保存值和不同的 return paths；所以把这段当作教学用的核心机制，不要直接当作所有环境下可复制的完整函数序言。

### 课件例子：caller-save registers

```c
int sum(int a, int b) { return a + b; }
int x = 1, y = 2;
int z = sum(x, y);
int w = sum(z, y);
```

caller-side assembly 的关键片段：

```asm
li a0, 1
li a1, 2
addi sp, sp, -8
sw ra, 0(sp)
sw a1, 4(sp)
jal ra, sum
lw a1, 4(sp)
jal ra, sum
lw ra, 0(sp)
addi sp, sp, 8
```

为什么保存 `ra` 和 `a1`，但不保存 `a0`？

- `ra`：第一次 call 之后 caller 还需要自己的 return address；
- `a1`：`y` 在第二次 call 仍然需要；
- `a0`：第一次 call 前它只是 `x`，第一次 call 后变成 `z`，而 `z` 正好就是下一次 call 的第一个 argument，所以不需要保留旧值。

这个例子体现一个重要原则：**caller-save register 不是每次都要保存，只有 call 之后仍然需要旧值时才保存。**

![[assets/3 Instruction Set Architecture/CA4-67.png]]

## 12. 从 ISA 走向 AI chip design

**设计动机：**通用 scalar instruction 能表达程序控制，却不能直接告诉硬件一个 tensor `tile`（矩阵/张量的局部块）的 shape、reuse 方式和跨 chip data movement；如果只用 scalar load/add，instruction 数量和搬运开销会淹没 `MAC (Multiply-Accumulate)` 阵列的计算能力。AI accelerator 因此引入 vector、matrix、`DMA (Direct Memory Access)`、同步或 custom instruction，让一次 command 覆盖更大的数据块。收益是数据并行和 reuse 更强，代价是 compiler、memory system、verification 和 programming model 都更复杂；超节点例子会把这条链扩展到 chip 间通信。

**工程师视角：**accelerator architect 追求 dataflow、MAC utilization 和 local buffer reuse；compiler engineer 追求 `layout`（元素排列）和 `stride`（地址步长）能否自动生成；system engineer 追求 DMA、`NoC (Network-on-Chip)` 和 collective communication 不要成为瓶颈；verification engineer 则必须覆盖不同 shape、precision、同步顺序和异常路径。真正成熟的 ISA 设计，必须让这几种视角在同一个 software-visible contract 中对齐。

### ISA 与 accelerator 的接口

如果你设计一个 AI chip，不能只问“MAC 单元每秒能算多少次”；还要定义软件怎样使用它。一个 accelerator ISA 或 custom extension 需要回答：

1. 输入 activation、weight、partial sum 位于哪个 register、SRAM 或 memory 地址？
2. instruction 如何描述 tensor shape、stride、data type 和 layout？
3. 一条 instruction 是做一个 scalar operation，还是启动一段 vector/matrix operation？
4. 结果写回哪里，何时可被下一条 instruction 读取？
5. 如何处理 quantization、saturation、rounding、overflow？
6. CPU、DMA、accelerator 之间如何同步，如何避免 cache/memory inconsistency？

因此，AI chip 常见的优化不只是在 ALU 上加乘法器，还包括：

```text
instruction / command
        ↓
decode + control
        ↓
register file / local SRAM
        ↓
MAC / vector / matrix datapath
        ↓
accumulator + activation / quantization
        ↓
写回 local memory 或 shared memory
```

### 背景扩展：从单个 accelerator 到超节点（Supernode）

`Supernode` 不是本 PDF 的原始术语；这里把它作为理解 AI 系统扩展性的背景概念。可以先把它理解成：**多个 AI accelerator、较大的共享 memory 和高速互连，被组织成一个对软件来说“像一台更大的机器”的 scale-up 域**。它比单颗 chip 大，但通常比跨机房的普通 cluster 更紧密，目标是让同一个模型的 tensor 能在这些计算单元之间快速交换。

单芯片时，instruction 主要描述“本地 register/SRAM 里的数据怎样计算”；进入超节点后，还必须描述或触发“数据怎样跨 chip 移动”。这里的 `NoC (Network-on-Chip)` 是 chip 内部的互连网络；`collective communication` 是多个 accelerator 共同参与的通信操作，例如把各 chip 的 `partial sum` 汇聚起来；`memory consistency` 则规定不同处理单元何时能可靠地看到彼此的 memory 更新。因此系统的可见接口通常分成三层：

| 层次 | 主要问题 | 对应本章概念 |
|---|---|---|
| chip 内 | 哪个 datapath 执行 operation？ | `opcode`、register file、ALU/MAC、pipeline |
| chip 间 | 哪个 tensor tile 发往哪个 accelerator？ | DMA、NoC/高速 link、address/queue、同步 |
| 超节点间 | 多个 scale-up 域如何协作？ | collective communication、软件 runtime、memory consistency |

<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 900 250" role="img" aria-label="超节点中多个 AI accelerator 通过高速互连共享数据的示意图" style="width:100%;height:auto;background:#f8fafc;border-radius:12px">
  <defs>
    <marker id="supernode-arrow" markerWidth="9" markerHeight="9" refX="8" refY="4.5" orient="auto"><path d="M0,0 L9,4.5 L0,9 Z" fill="#475569"/></marker>
  </defs>
  <g font-family="sans-serif" fill="#0f172a">
    <text x="450" y="27" text-anchor="middle" font-size="19" font-weight="700">Supernode：一个紧密互连的 scale-up 域</text>
    <rect x="28" y="52" width="844" height="164" rx="14" fill="#ffffff" stroke="#94a3b8" stroke-width="2"/>
    <g font-size="15" text-anchor="middle">
      <rect x="60" y="92" width="145" height="62" rx="9" fill="#dbeafe" stroke="#2563eb" stroke-width="2"/><text x="132" y="119">AI chip 0</text><text x="132" y="140" font-size="13">MAC + SRAM</text>
      <rect x="250" y="92" width="145" height="62" rx="9" fill="#dbeafe" stroke="#2563eb" stroke-width="2"/><text x="322" y="119">AI chip 1</text><text x="322" y="140" font-size="13">MAC + SRAM</text>
      <rect x="505" y="92" width="145" height="62" rx="9" fill="#dcfce7" stroke="#16a34a" stroke-width="2"/><text x="577" y="119">Shared memory</text><text x="577" y="140" font-size="13">weights / KV cache</text>
      <rect x="695" y="92" width="145" height="62" rx="9" fill="#fef3c7" stroke="#d97706" stroke-width="2"/><text x="767" y="119">Runtime / DMA</text><text x="767" y="140" font-size="13">queues + sync</text>
    </g>
    <g stroke="#475569" stroke-width="2.5" marker-end="url(#supernode-arrow)"><path d="M205 123 H240"/><path d="M395 123 H495"/><path d="M650 123 H685"/></g>
    <text x="450" y="190" text-anchor="middle" font-size="14" fill="#475569">高速 link / NoC：搬运 activation、weight、partial sum</text>
  </g>
</svg>

读图时要注意：超节点不是“把 instruction 变长”这么简单。它把本章的几个问题放大了：`load/store` 的地址空间如何分层、DMA 什么时候搬数据、不同 chip 之间怎样同步、partial sum 是否需要跨 chip 汇聚，以及通信 latency 是否会让 MAC 阵列等待数据。

对 AI workload，常见的系统级操作可以抽象成：

```text
tile 到达 local SRAM
  → accelerator 执行 matrix/vector instruction
  → partial sum 保留或发送到另一个 chip
  → collective communication 汇聚结果
  → quantization / activation
  → 写回 shared memory 或下一个计算阶段
```

因此评价超节点不能只看峰值 `TOPS`。还要看 memory bandwidth、link bandwidth、通信 latency、同步开销、数据复用率，以及 compiler/runtime 能否把这些搬运和计算安排好。这个连接正是从 ISA 走向 AI chip system design 的下一步。

#### 一个最小伪代码例子：超节点执行分块矩阵乘法

下面是教学用伪代码，不是某个真实厂商的 API。目标是计算 $C = A \times B$，并把 $C$ 切成多个 tile，交给超节点中的不同 accelerator。假设每个 accelerator 有自己的 local SRAM 和 MAC array，`all_reduce` 负责把不同 chip 算出的 partial sum 汇聚起来。

先把一个 accelerator 看成只能理解四条“超精简指令”的小机器：

```text
SET_TILE  dst, A_range, B_range, shape   # 设置地址、tile 形状和数据类型
DMA_IN    local_buffer, shared_memory    # 把数据搬到本地 SRAM
MAC_TILE  A_buffer, B_buffer, accumulator # 启动矩阵乘加
REDUCE    accumulator, link_group        # 通过互连汇聚 partial sum
```

这不是标准 RISC-V encoding，而是帮助理解的 accelerator command model。它把复杂的矩阵运算拆成“配置 → 搬运 → 计算 → 汇聚”四个软件可见动作。真实芯片可以把它们实现成 custom instruction、memory-mapped command 或 runtime API。

对应的教学用调用顺序是：

```text
SET_TILE  C[0:16, 0:16], A[0:16, 0:64], B[0:64, 0:16], INT8
DMA_IN    SRAM.A, A[0:16, 0:64]
DMA_IN    SRAM.B, B[0:64, 0:16]
MAC_TILE  SRAM.A, SRAM.B, ACC.INT32
REDUCE    ACC.INT32, group_0
```

你可以把它和前面的 RISC-V `add` 对比：`add` 直接描述一次 scalar register operation；`MAC_TILE` 描述的是一次更大的 accelerator operation，可能在内部启动很多 MAC、buffer 和 pipeline stage。前者适合通用 CPU，后者适合 AI chip，但二者都必须定义清楚 input、output、状态和完成时机。

```text
function supernode_matmul(A, B, C, accelerators):
    for each output_tile (m, n) in C:
        partials = []

        for each accelerator p in accelerators:
            k_range = partition_K_dimension(p, A, B)

            # 1. Runtime/ISA command: 把本次 tile 的地址和形状交给 p
            command(p, "SET_TILE",
                    A_tile=(m, k_range),
                    B_tile=(k_range, n),
                    C_tile=(m, n))

            # 2. DMA: 从 shared memory 搬到 p 的 local SRAM
            dma_load(p.local_sram.A, A[m, k_range])
            dma_load(p.local_sram.B, B[k_range, n])

            # 3. Custom/vector instruction: local MAC array 计算 partial sum
            partial = matrix_mac(
                p.local_sram.A,
                p.local_sram.B,
                precision="INT8",
                accumulate="INT32")
            partials.append(partial)

        # 4. Communication: 跨 accelerator 汇聚同一个 output tile
        C[m, n] = all_reduce_sum(partials)

        # 5. 写回前做量化或 activation
        C[m, n] = requantize(C[m, n], output_type="INT8")

    return C
```

把这段伪代码映射回本章的硬件语言：`command` 类似 accelerator-visible instruction；`dma_load` 是数据搬运而不是 ALU 运算；`matrix_mac` 可能由一个 custom extension 启动；`all_reduce_sum` 暴露了 chip 间 link 和同步成本；最后的 `requantize` 对应 rounding、saturation 和 data type conversion。

最容易漏掉的一点是 `partial` 的生命周期：它不能在 local SRAM 被覆盖前丢失，也不能让 MAC array 长时间等待通信。真实设计会进一步决定 tile 大小、double buffering、通信拓扑，以及下一次 `dma_load` 是否可以和当前 `matrix_mac` 重叠执行。

### 为什么 RISC-V 对 custom AI instruction 有吸引力

RISC-V 的 modularity 允许在保留 base ISA 的同时加入 extension。一个典型思路是：

- `I` 处理 control、address 和普通 integer operation；
- `V` 处理通用 vector data parallelism；
- custom extension 处理特定 matrix/tensor operation；
- compiler 把高层 kernel 映射为这些 instruction；
- microarchitecture 决定 instruction 是直接驱动 datapath，还是启动更长的内部 sequence。

但 custom instruction 也有代价：要设计 encoding、decoder、register interface、pipeline hazard、verification、compiler intrinsic/算子 lowering 和性能模型。**能执行不等于能被软件高效使用。**

### 用本章知识阅读 AI accelerator paper

以后读一篇 AI chip paper，可以按下面顺序提问：

1. 它的 software-visible interface 是什么？是普通 instruction、custom instruction、memory-mapped command，还是 kernel API？
2. 它的 data movement 由谁发起？CPU、DMA、load/store unit 还是专用 network-on-chip？
3. 它使用什么 data type？INT8、FP16、BF16、FP8？扩展和 rounding 规则在哪里定义？
4. 它的 compute tile 如何从 local SRAM 取得数据？是否有 reuse？
5. instruction latency 与 throughput 如何定义？是否被 memory 或 pipeline stall 限制？
6. compiler 是否能自动生成高效 instruction，还是依赖手写 assembly/kernel？

这套问题把“ISA 是一组指令”提升为“ISA 是整个 software/hardware contract”。

## 13. 本章最容易混淆的概念

| 容易混淆 | 正确区分 |
|---|---|
| ISA vs microarchitecture | ISA 是软件可见规范；microarchitecture 是内部实现 |
| instruction vs instruction set | 一条操作 vs 所有可用操作的集合 |
| register vs memory | register 在 CPU 内部、快而少；memory 容量更大、访问更慢 |
| `lw` vs `lb` | 都是 load，但读出的数据宽度不同 |
| `lb` vs `lbu` | 都读 byte；前者 sign-extend，后者 zero-extend |
| logical right shift vs arithmetic right shift | 前者补 0；后者复制 sign bit |
| branch vs jump | branch 有条件；jump 无条件，`jal` 还保存 return address |
| real instruction vs pseudo-instruction | 前者有真实 encoding；后者由 assembler 展开 |
| caller-save vs callee-save | 谁在 function call 前/后负责保护 register |
| latency vs throughput | 单条/单个任务的完成时间 vs 单位时间完成量 |

## 14. 学习检查

先不看答案，尝试口头解释：

1. 为什么 ISA 不是 CPU 的全部？
2. `add x5, x6, x7` 中每个 register 分别做什么？
3. 为什么 RISC-V 要把 load/store 与普通 ALU operation 分开？
4. `lb` 与 `lbu` 什么时候会给出不同的高位 bits？
5. `beq x5, x6, label` 如何决定下一条 PC？
6. 为什么 RISC-V 同时需要 R-type、I-type、S-type 等格式？
7. 在第二次 `sum(z, y)` 前，为什么课件例子要恢复 `a1`？
8. 如果 AI accelerator 有很强的 MAC，但没有足够 memory bandwidth，会发生什么？

### 老师带你做一个最小练习

把下面的 C 表达式翻译成 RISC-V 风格的 assembly。假设 `a`、`b`、`c` 已经在 registers 中，结果放入 `out`：

```c
out = (a + b) - c;
```

**第 1 步：拆出中间结果。**先计算 `a + b`，需要一个 temporary register。

```asm
add t0, a, b
```

此时 `t0 = a + b`。

**第 2 步：完成第二个 operation。**把 `t0` 减去 `c`，结果写入 `out`。

```asm
sub out, t0, c
```

**第 3 步：用硬件视角复述。**这两条 instruction 都是 register-register operation：register file 读出 operand，ALU 做加法或减法，结果写回 register file。没有任何 memory access。

**第 4 步：连接 AI chip。**如果 `a`、`b`、`c` 是向量，scalar `add/sub` 就可能扩展成 vector instruction；如果它们是 tensor tile，则需要更大的 local buffer、更多并行 ALU/MAC 和相应的 custom instruction。抽象流程不变：读 operand → 计算 → 写回。

### 自测答案如何判断对错

- 是否先保存了 `a+b` 这个中间结果？
- `sub` 的两个 source 是否是 `t0` 和 `c`？
- 最终结果是否写入 `out`？
- 是否能说出这两条 instruction 为什么不需要 load/store？

### 参考答案

1. ISA 只规定 software 可见的行为；pipeline、cache、out-of-order、ALU 数量等属于 microarchitecture。
2. `x6`、`x7` 是 source，`x5` 是 destination；结果为 `x6 + x7`。
3. 规则化的 register-to-register datapath 更容易 decode、做 pipeline，也让 memory access 的代价更清楚。
4. 当 byte 的最高位为 1 时，signed interpretation 需要复制 1，unsigned interpretation 需要补 0；例如 `1111 1110`。
5. 比较 `x5` 与 `x6`；相等则 `PC ← PC + offset` 跳到 label，不等则按顺序执行下一条 instruction。
6. 不同 instruction 需要不同 operands：有的需要两个 register，有的需要 immediate，有的需要 store/branch offset；不同格式能在固定长度中容纳这些字段。
7. `a1` 保存的是 `y`，而 caller-save register 可能被第一次 call 覆盖；第二次 call 仍要使用 `y`。
8. compute unit 会等待数据，pipeline 出现 stall；峰值 TOPS/FLOPS 不等于真实 application performance。

## 15. 继续学习的顺序

建议按下面的路线继续：

```text
ISA 基本行为
  → single-cycle datapath：instruction 如何被 decode 和执行
  → pipeline hazards：data / control hazard 与 forwarding
  → memory hierarchy：cache、SRAM、DRAM、bandwidth
  → vector / SIMD / GPU：一条 instruction 处理多个 data
  → AI accelerator：tensor dataflow、local buffer、DMA、NoC
  → compiler + hardware co-design：把模型算子映射到 ISA 和 datapath
```

现在不必马上背完所有 encoding 位段。优先掌握四条主线：

1. **行为线：**instruction 读什么、算什么、写什么；
2. **数据线：**register、memory、offset、extension 如何搬运 data；
3. **控制线：**PC、branch、jump、function call 如何改变执行顺序；
4. **实现线：**规则的 ISA 如何帮助 decoder、pipeline 和 AI datapath。

---

## 16. Reference glossary（查阅用，不是先修要求）

| Term | 初学者解释 |
|---|---|
| `ISA` | software 与 hardware 之间的 instruction-level contract |
| `instruction` | 一条由 processor 执行的操作描述 |
| `register` | CPU 内部很小、很快的 storage |
| `register file` | 一组 registers 及其读写端口 |
| `immediate` | 直接编码在 instruction 中的小常数 |
| `opcode` | 标识 instruction 类别的编码字段 |
| `funct3/funct7` | 进一步选择具体 operation 的字段 |
| `PC` | Program Counter，保存当前/下一条 instruction 的地址 |
| `RISC` | 以规则、较简单 instruction 为核心的设计取向 |
| `CISC` | 倾向让一条 instruction 完成更多工作的设计取向 |
| `pipeline` | 把 datapath 分 stage，让多条 instruction 重叠执行 |
| `load/store` | 在 memory 与 register 之间读写 data 的 instruction |
| `data movement` | 在 memory hierarchy、register、local SRAM 和 accelerator 之间搬运 data；很多 AI workload 的瓶颈来源。 |
| `data locality` | 让近期会复用的数据停留在较近 storage，减少昂贵的远端访存。 |
| `data reuse` | 一次 load 支撑多次 computation，摊薄 bandwidth、latency 和 energy 成本。 |
| `sign extension` | 复制 sign bit 扩展 signed data |
| `zero extension` | 高位补 0 扩展 unsigned data |
| `branch` | 根据条件决定是否修改 PC |
| `jump` | 无条件修改 PC；`jal` 还保存返回地址 |
| `ABI` | 规定 function、register、stack 如何协作的软件接口 |
| `caller` | 发起 function call 的函数 |
| `callee` | 被调用的函数 |
| `custom extension` | 为特定 workload 增加的 ISA 功能 |
| `Supernode` | 多个 AI accelerator、共享 memory 与高速互连组成的紧密 scale-up 域；重点观察 chip 间 data movement 和同步。 |

## 17. 专家层：ISA、ABI、microarchitecture 与 memory model

学到这里，最容易出现的高级混淆是：**只要都叫“接口”，就把 ISA、ABI 和 microarchitecture 混成一层。**它们的可见对象不同，改变它们的代价也不同。

| 层次 | 规定什么 | 直接影响谁 | 典型改变 |
|---|---|---|---|
| ISA | instruction、register、encoding、exception、memory-visible behavior | compiler、OS、hardware implementation | 增加 `M/F/D/V` 或 custom extension |
| ABI | function 参数、返回值、stack、register 保存、object/binary 连接 | compiler、linker、library、OS | 改变 calling convention 会影响二进制兼容 |
| microarchitecture | datapath、pipeline、cache、OoO、prediction 的内部实现 | chip implementation、power/performance/area | 同一 ISA 做不同 core |

**判断方法：**如果一个 C 程序重新编译后仍能运行，可能只是 compiler 选择不同；如果旧的 machine code 也必须保持行为，必须看 ISA 与 ABI 是否兼容；如果软件完全不变但速度/功耗不同，通常是 microarchitecture 或 implementation 改变。

### 17.1 读懂 RISC-V 的 extension 字符串

`RV32I` 表示 32-bit 基础整数 ISA；`RV64I` 表示 64-bit 基础整数 ISA。后面的字母表示扩展，例如 `M` 是 integer multiply/divide，`A` 是 atomic，`F/D` 是 single/double precision floating-point，`C` 是 compressed instruction。常见的 `G` 是一组常用扩展的 shorthand；实际项目应以对应版本的规范和实现支持列表为准。

扩展的设计问题不是“功能越多越好”：

- compiler 是否能稳定生成它；
- decoder、register file、execution unit 是否需要新路径；
- context switch、debugger、ABI 和 OS 是否理解新增状态；
- 新指令减少了多少 instruction count，是否值得增加硬件和验证成本。

### 17.2 为什么 memory model 属于 ISA，而 cache coherence 通常属于实现层

多核程序会问：“一个 core 写入的数据，另一个 core 何时、以什么顺序能看到？”`Memory model（内存模型）`规定软件可依赖的可见顺序与同步规则；cache coherence 则描述多个 cache 如何保持同一地址的副本一致。前者是程序语义的一部分，后者可以通过不同硬件协议实现，只要对外满足 ISA 规定的行为。

因此看到 `fence`、atomic instruction 或 synchronization primitive 时，不要只把它当作“额外的 memory instruction”：它们是在告诉处理器和 compiler 哪些重排可以做、哪些观察顺序必须保留。

### 17.3 程序怎样进入操作系统？Privilege、trap、exception 与 interrupt

用户程序不应任意读写设备或改写系统级状态，因此 ISA 需要定义权限边界。Privilege mode（特权级）表示当前执行环境可以使用哪些指令和资源；RISC-V 常见模式包括 User、Supervisor 与 Machine，具体实现可以只支持其中一部分。

Trap（陷入）是控制流转交给更高权限软件处理程序的架构事件。Exception（异常）由当前 instruction 的执行引起，例如非法 instruction 或访问错误；interrupt（中断）由外部或计时器等异步事件触发，和当前 instruction 的具体运算无关。

教学级路径是：硬件保存必要的 PC 与原因到 control/status register（CSR，控制与状态寄存器），跳到 trap handler（陷入处理程序）；操作系统检查原因、处理后决定恢复或终止程序。具体 CSR、返回语义和 delegation 规则见 privileged specification。普通应用程序通常不直接管理这些细节。

这和 ABI 不同：ABI 约定普通函数如何传参和返回；trap 是硬件与操作系统之间的受控入口。两者都是接口，但参与者和状态边界不同。

### 17.4 Atomic instruction：为什么普通 load/add/store 不能安全更新共享计数器

假设两个 hart（硬件线程）同时执行 counter = counter + 1。普通指令通常展开为 load、add、store。若两个 hart 都先读到 7，各自算出 8，最后都写 8，两次加一只让计数增加一次，这叫 lost update（更新丢失）。

Atomic read-modify-write（原子读-改-写）把读取、修改、写回作为不可被其他处理器插入的整体操作。RISC-V A extension 包含 LR/SC（load-reserved/store-conditional）与 AMO（Atomic Memory Operation）等机制，支持锁、计数器和同步算法。Atomicity 解决“更新不可拆开”；memory ordering 解决其他读写以什么顺序被观察，二者相关但不是同一性质。

例如 AMO 加法可原子地给一个 word 加上数值并返回旧值。但竞争仍会导致等待，过度使用可能成为共享瓶颈；正确同步还要选择合适的 ordering 语义。这是操作系统和并行程序的背景扩展，不是单线程 assembly 入门的先修要求。

### 17.5 Endianness 与 alignment：同一个 word 怎样放进 memory？

Memory 通常按 byte address（字节地址）编号，而一个 word 可能包含多个 bytes。Endianness（字节序）规定一个多字节数的最低有效 byte 放在哪个较低地址。以 32-bit 数值 0x11223344 为例：

| 地址从低到高 | Little-endian | Big-endian |
|---|---|---|
| A | 44 | 11 |
| A+1 | 33 | 22 |
| A+2 | 22 | 33 |
| A+3 | 11 | 44 |

这不是“数字变了”，而是同一个数在 memory 中的 byte 排列不同。读取完整 32-bit word 的硬件会按当前系统规则重新组合；但用 lb 逐 byte 读取、解析文件格式或和另一台机器交换 binary data 时，byte order 就会直接影响结果。网络协议和文件格式通常会明确规定自己的 byte order，不能默认使用 CPU 的本地排列。

Alignment（对齐）问的是一个对象的起始地址是否满足它的宽度或 ABI 要求。例如 4-byte word 常放在 4 的倍数地址，8-byte value 常放在 8 的倍数地址。对齐有助于 memory interface 一次完成访问；未对齐访问在某些实现上可能被拆成多次、产生额外成本，或触发 exception，具体行为由 ISA 与实现共同决定。

因此看到 lw x5, 0(x6) 时，除了计算有效地址，还要问：地址是否满足该 access 的 alignment？读取的 byte order 如何解释？这就是第 2 章的数据表示、第 3 章的 load/store 和第 4 章的 memory pipeline 之间的接口。

### 17.6 Compressed instruction 的真实取舍

`C extension` 用较短编码表达常见操作，可能减少 program code size，进而减少 instruction memory traffic 和 instruction-cache pressure；代价是 decoder 需要处理不同长度的 instruction，取指边界和对齐逻辑会更复杂。它体现了 ISA 的典型权衡：**减少 instruction bytes 不等于每条 instruction 都更简单**，最终收益要用 workload 的 cache miss、fetch bandwidth 和 decode cost 测量。

### 17.7 在线拓展

- RISC-V 非特权 ISA 与 extensions：<https://docs.riscv.org/reference/isa/v20240411/>（RISC-V International，官方规范；访问日期：2026-10-21）。
- RISC-V privileged architecture：<https://riscv.org/technical/specifications/>（从官方 ratified specifications 目录进入 Privileged Architecture；访问日期：2026-10-21）。
- RISC-V ELF psABI：<https://github.com/riscv-non-isa/riscv-elf-psabi-doc>（RISC-V Non-ISA specification，官方维护仓库；访问日期：2026-10-21）。
- RISC-V 压缩指令、向量和其他扩展的目录：<https://riscv.org/technical/specifications/>（RISC-V International；访问日期：2026-10-21）。
- RISC-V Privileged Architecture（特权级、CSR、trap/interrupt）：<https://docs.riscv.org/reference/isa/v20240411/priv/priv-preface.html>（RISC-V International；访问日期：2026-10-22）。
- RISC-V A extension（LR/SC、AMO 与 ordering）：<https://docs.riscv.org/reference/isa/v20260120/unpriv/a-st-ext.html>（RISC-V International Ratified Specifications；访问日期：2026-10-22）。
- RISC-V 基础 ISA 关于字节寻址、endianness 与 instruction parcel 的说明：<https://docs.riscv.org/reference/isa/v20240411/unpriv/intro.html>（RISC-V International；访问日期：2026-10-22）。

## 18. Source and verification note

### 主要课件案例与截图索引

| Case ID | 来源页 | 课件案例 | 正文落点 | 截图（状态/可读性） |
|---|---:|---|---|---|
| CA4-01 | 5--14 | ISA、RISC/CISC 与 pipeline 动机 | ISA contract、RISC/CISC | `CA4-5.png`、`CA4-10.png`、`CA4-14.png`（正文已嵌入；正常宽度可读） |
| CA4-02 | 11 | `f=(g+h)-(i+j)` 的中间寄存器 | computational operation | `CA4-11.png`（正文原页；正常宽度可读） |
| CA4-03 | 28、31--32 | `0x00A98933 → add x18,x19,x10` | instruction encoding | `CA4-28.png`、`CA4-31.png`、`CA4-32.png`（正文已嵌入；正常宽度可读） |
| CA4-04 | 37、40、44--46 | load/store、branch 与 PC-relative address | 三类 instruction 与 encoding | `CA4-37.png`、`CA4-40.png`、`CA4-44.png`、`CA4-45.png`、`CA4-46.png`（正文已嵌入） |
| CA4-05 | 47--49 | register file、memory hierarchy 与 reuse | data movement | `CA4-47.png`、`CA4-48.png`、`CA4-49.png`（正文已嵌入；正常宽度可读） |
| CA4-06 | 59、63--67 | stack、callee-save/caller-save | ABI 与 function call | `CA4-59.png`、`CA4-63.png`、`CA4-64.png`、`CA4-65.png`、`CA4-67.png`（正文已嵌入） |
| CA4-07 | 不适用 | Supernode 与 AI accelerator 背景图 | AI chip design | 正文内教学 SVG（外部背景；不是课件截图，已提供文字 fallback） |

正文中的每个案例都先给人类可读的动作路径，再给 machine-level 形式；截图用于检查课件字段、结构和原始案例。所有源图位于 `assets/3 Instruction Set Architecture/`。

- 主要来源：`/Users/dongyangchen/Desktop/lecture note/CA/CA_4.pdf`，课件标注为 Chapter 3 - ISA，日期为 September 30, 2026。
- 已保留课件中的核心例子：`f = (g+h)-(i+j)`、`0x00A98933 → add x18,x19,x10`、`lw x1,4(x2)`、branch/jump、callee-save 和 caller-save function-call 例子。
- 原课件共 70 页；正文只嵌入直接支撑概念或保留完整 worked case 的页面，不附逐页图库。选取的截图覆盖 ISA 层次、RISC/CISC 对比、五级 pipeline、RISC-V extensions、register/register-immediate operation、移位、六种 encoding format、`lui/auipc`、load/store、memory hierarchy、stack/heap 和 caller/callee-save cases。
- 截图文件统一保存在 `assets/3 Instruction Set Architecture/`，正文使用 Obsidian 相对链接。
- “超节点（Supernode）”部分是面向 AI chip design 的背景扩展，不是 `CA_4.pdf` 中的原始讲义内容；其中的 SVG 为帮助理解 chip 间计算、memory、DMA 与通信关系的教学图示。
- 本轮按更新后的通用 skill 重新组织学习路径：全章只做简短动机导航，随后在各主题的首次需要处按“可感知问题 → 局部约束 → 设计选择 → 机制 → 收益/代价 → 下一概念”逐步展开，并在关键取舍处加入 hardware、compiler、memory/system、verification、performance/energy 等从业者视角；PDF 原始案例与后加的 AI chip 背景例子保持区分。
- 外部背景核对：RISC-V 官方 ISA 手册用于确认 base ISA、instruction encoding 与 extensions 的标准术语：[RISC-V ISA Manual](https://docs.riscv.org/reference/isa/v20240411/)。该链接用于补充专业背景，不替代本课件中的定义和案例。
- 外部背景拓展：RISC-V 官方 RVWMO memory model 说明多 hart（硬件线程）观察 load/store 顺序的规则，以及 `FENCE` 如何建立排序约束：[RVWMO](https://docs.riscv.org/reference/isa/v20240411/unpriv/rvwmo.html)（访问日期：2026-10-19）。这是 ISA/ABI 之后的进阶主题；单核顺序执行入门不需要先掌握它。
