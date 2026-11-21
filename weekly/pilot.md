# 处理器架构与矢量设计周刊

**第 001 期 · 试刊号 · 2026 年 11 月 17 日**

覆盖窗口为北京时间 2026-11-10 21:57 至 2026-11-17 21:57，含起点。检索于 11 月 17 日晚启动。

## 矢量与矩阵

### Zen 6：扩充整数点积的同时，让加载端供应更多独立操作数

11 月 11 日，Chips and Cheese 发布对 AMD Zen 初代架构师 Mike Clark 的访谈。[访谈原文](https://chipsandcheese.com/p/talking-zen-6-with-mike-clark-at)

**问题**　量化推理在循环里反复加载成组数据，再用 VNNI 做整数点积并累加。如果下一组操作数没就绪，或者所有迭代都依赖同一个累加器，增加点积流水线以后仍可能有空拍。

**方法**　Zen 6 相对 Zen 5 增加两条 VNNI 流水线，加载端也能同时服务四个独立的 256 位向量加载。Clark 解释，Zen 5 已有服务两个 512 位加载所需的布线；把这组布线分开交给更多加载，控制会更复杂。

**披露的效果**　两个 512 位加载与四个 256 位加载的总宽度相同，都是 1,024 位。差别在于同一时段能服务更多条独立加载。这个比较来自接口宽度和并行度，实际点积循环的吞吐仍需测量。

**可借鉴**　用同一条点积指令先做纯寄存器循环，把独立累加链加到吞吐饱和。然后加入独立加载，先让工作集命中 L1，再扩大到主要命中 L2，比较点积指令/周期、加载字节/周期和缓存事件。这样可以先找出扩大执行器还是改进供数更值得做。

<details class="article-detail" markdown="1">
<summary>展开三组微基准与 L2 需求读、预取竞争</summary>

| 测试 | 改变的条件 | 观察什么 |
|---|---|---|
| 纯寄存器点积 | 扫独立累加链数量，固定指令宽度 | 点积指令/周期何时停止增长 |
| L1 供数 | 让操作数经加载到达，固定累加链数量 | 与纯寄存器的差值，以及加载使用延迟和读吞吐 |
| L2 供数 | 增大工作集，保持访存步长和指令序列 | 缓存命中位置改变时，点积吞吐怎样变化 |

这是建议的测试序列。跨平台对照时采用同一指令编码，保持对齐、绑核和测量方式一致，记录频率和重复结果。Clark 也提到，Zen 5 上命中 L2 的预取会占用需求读带宽，Zen 6 减少了这类竞争，设计目标是维持 **64 B/周期** 的 L2 读取能力。若寄存器和 L1 两组跟得上，进入 L2 后才变慢，可在可控的情况下比较不同预取设置，或用固定工作集对照规则与不规则步长。[AMD 架构师访谈](https://chipsandcheese.com/p/talking-zen-6-with-mike-clark-at)

</details>

### SIMD 还是 CGRA？面向 AI 非线性函数的 RVV 与 CGRA SoC 对照

*To SIMD or Not to SIMD: Comparing RVV vs. CGRA SoCs for Non-Linear Functions in AI*

Sameer Deshmukh、Naoya Mizuki、Takumi Honda（富士通）。arXiv:2611.12435 v1 在窗口前提交，纳入本期的是 SC26 H2RC 工作坊安排在北京时间 11 月 16 日 23:30 的报告。[会议议程](https://sc26.conference-program.com/presentation/?id=ws_h2rc103&sess=sess139) · [预印本及 PDF](https://arxiv.org/abs/2611.12435) · [全文](https://arxiv.org/html/2611.12435v1)

**问题**　同一套 Gemmini 已经负责矩阵乘。剩下的 SiLU 和 Softmax，交给 RVV 还是可重构阵列（CGRA）？

**方法**　作者固定 Rocket 核、Gemmini 和内存层级，只比较两种非线性算子路径。SiLU 的每个元素彼此独立，CGRA 填满流水线后能连续处理。Softmax 先归约一整行，再归一化。这套 CGRA 还要在 CPU 与阵列之间交接中间值并按块重启，开销随之上来。

**结果**　RTL 周期仿真中，CGRA 的 SiLU 内核最高快 **7.12×**，RVV 的 Softmax 内核最高快 **9.8×**。以 N=128 的 Softmax 为例，Saturn RVV 为 **9.1 周期/元素**，CGRA 为 **89.3**。两项最大倍率对应各自的测试规模。[论文结果](https://arxiv.org/html/2611.12435v1#S5)

**可借鉴**　评估一条算子链时，先分别计数**执行单元占用、归约等待、跨单元搬运**，再回到包含矩阵乘的整块周期。若归约路径慢，先量化少一次中间值往返能省几拍，再考虑增加算术单元。

<details class="article-detail" markdown="1">
<summary>看两套架构原图、Softmax 周期分解和 SiLU 消融</summary>

#### 硬件路径怎么读

先在两张图上各找一次 **Rocket** 和 **Gemmini**。Rocket 发出指令或控制命令，Gemmini 负责矩阵乘。再找处理非线性算子的那块硬件。图 1 的 Saturn 执行 RISC-V 向量指令，跟着 Rocket 的指令流水线工作。图 2 的 CGRA（粗粒度可重构阵列）通过控制器接收命令，由 DMA 搬运数据，先放入片上暂存区再送进阵列。

<div class="figure-pair" markdown="1">

<figure class="source-figure" markdown="1">

![富士通原图 1：Rocket、Saturn RVV 与 Gemmini 的连接](https://arxiv.org/html/2611.12435v1/saturn_gemmini_arch_2.svg)

<figcaption markdown="1">

**原图 1 · Saturn RVV 路径。** 顺着 Rocket 找 Saturn，二者位于同一个 tile。图中的 Gemmini 仍是单独的矩阵乘单元。Deshmukh、Mizuki、Honda（富士通），© 2026 IEEE。[看原图 1](https://arxiv.org/html/2611.12435v1#S3.F1) · [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

</figcaption>
</figure>

<figure class="source-figure" markdown="1">

![富士通原图 2：Rocket、CGRA 控制器、暂存区与 Gemmini 的连接](https://arxiv.org/html/2611.12435v1/cgra_gemmini_arch.svg)

<figcaption markdown="1">

**原图 2 · CGRA 路径。** 先看 Rocket 到 CGRA 控制器的命令路径，再看 DMA 到暂存区、暂存区到阵列的数据路径。Deshmukh、Mizuki、Honda（富士通），© 2026 IEEE。[看原图 2](https://arxiv.org/html/2611.12435v1#S4.F2) · [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

</figcaption>
</figure>

</div>

#### 两种算子在硬件上怎样执行

**SiLU** 在 CGRA 上映射成一条空间流水线。输入仍需等待流水填满，但之后可以持续向前推进，算子内部无需为每组元素重复发射一串向量指令。**Softmax** 的 CGRA 映射把求最大值和倒数放在 Rocket 上，数据按块经过暂存区，阵列先为归约计算，再为归一化重新启动。Saturn 则用向量加载和存储处理成组元素。它配置为 VLEN 128 位、数据通路 128 位，LMUL=4 时一条 `vle32.v` 可覆盖 16 个 FP32 元素，经这条通路分四拍传送。[实现细节与性能分析](https://arxiv.org/html/2611.12435v1)

#### 从内核到整块

保留矩阵乘和其他开销后，整块收益比单独测内核小。下表按作者的周期仿真结果列出各自最高值。[论文结果](https://arxiv.org/html/2611.12435v1)

| 负载 | 周期数较少的一侧 | 非线性内核最高加速比 | 相应整块最高加速比 |
|---|---|---:|---:|
| SiLU | CGRA | 相对 RVV 约 7.12× | SwiGLU 约 3.16× |
| Softmax | RVV | 相对 CGRA 约 9.8× | Attention 约 4.24× |

#### Softmax 的时间花在哪儿？

原图 8 最右一组对应 N=128。CGRA 的右柱中，约 45.1% 花在阵列上，约 30.2% 花在 CPU 与阵列之间的拷贝。这个分解能指出先改哪段路径，再回测内核与整块。

<figure class="source-figure chart-figure" markdown="1">

![富士通原图 8：不同输入规模下 RVV 与 CGRA 的 Softmax 每元素周期数，以及 CGRA 的各阶段开销](https://arxiv.org/html/2611.12435v1/softmax_cycles_per_element_breakdown.svg)

<figcaption markdown="1">

**原图 8 · Softmax 周期分解。** ME-VE 是 Saturn RVV 方案，ME-RE 是 CGRA 方案。每组左柱为 RVV，右侧堆叠柱为 CGRA；右柱各色块显示阵列、CPU、控制与搬运的贡献。Deshmukh、Mizuki、Honda（富士通），© 2026 IEEE。[看原图 8](https://arxiv.org/html/2611.12435v1#S5.F8) · [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

</figcaption>
</figure>

#### 控制变量后，SiLU 的优势还在吗？

一种解释是 RVV 要用软件算指数，而 CGRA 有流水化指数单元。作者在图 6 单独去掉 RVV 内核的指数计算，其余计算路径用于对照。在 N=16、32、64、128 四组中，省掉指数让原 RVV 激活阶段少用约 **12.7%–13.2%** 的周期，RVV 的每元素周期仍约为 CGRA 的 **5.4–6.2 倍**。这组实验能削弱「指数软件代价足以解释全部差距」的说法，支持继续查空间流水和逐组发射成本。作者用去掉指数的路径估计周期份额。完整 SiLU 的输出误差另见图 5。[作者消融实验、图 6](https://arxiv.org/html/2611.12435v1#S5.F6) · [实验条件](https://arxiv.org/html/2611.12435v1#S5.SS1.SSS1)

</details>

### FP32 输入下 SIMD 数学库的函数级精度与性能评估

*Evaluating Function-Level Accuracy and Performance of SIMD Math Libraries with FP32 Inputs*

Xingjian Qi、Robert Strzodka（海德堡大学），SC26 Correctness 工作坊，北京时间 11 月 17 日 00:45。

**问题**　同一个 AVX2 数学函数，输入越过边界时可能进入慢路径。平均吞吐和库说明里的精度上限，分别容易漏掉尾延迟与最坏误差。

**方法**　作者比较四套库的单参数 FP32 接口。精度穷举有限 FP32 编码，用 256 位 MPFR 作参照；性能按便宜和昂贵的执行路径分组，再与标量接口对照。[会议摘要](https://sc26.conference-program.com/presentation/?id=ws_corr108&sess=sess134)

**结果**　部分函数的慢路径比标量还慢。海德堡大学列出的一个精度反例是 AOCL 5.3.0 的 AVX2 余弦，最坏观察值 **12.778 ULP**，该表所列上限为 **4 ULP**。[作者的逐函数误差表](https://asc.ziti.uni-heidelberg.de/projects/2026-PNRFC/#fp32-avx2-accuracy)

**可借鉴**　测向量数学库时，把全常见值、全边界值和**只混入一个难算 lane** 分成三组。同时看每元素周期和最坏 ULP，才能判断一个边界值到底拖慢了哪部分。

<details class="article-detail" markdown="1">
<summary>查看库版本、误差口径和复核入口</summary>

被测库包括 AOCL-LibM、GNU libmvec、SLEEF 和 Intel SVML。余弦数据来自作者项目页列出的 AMD EPYC、256 位 FP32 AVX2 接口。ULP 可近似理解为这个数值量级上跨了几格可表示值。表里的 **12.778 ULP** 描述被测输入域的极值；要知某类业务输入多常触发它，要另看输入分布。项目页后加的 SVML CR 数据与原研究中的库版本分别记录。[版本、适用区域和测试代码](https://asc.ziti.uni-heidelberg.de/projects/2026-PNRFC/)

</details>

## 通用处理器架构

### Apple M 系列处理器是否满足 Arm 与 TSO 的复合内存模型？

*Do Apple M-Series Processors Enforce the Compound Memory Model? Sequential and Parallel Composition of Arm and TSO*

Vijay Nagarajan（犹他大学），arXiv:2611.21406 v1，北京时间 11 月 17 日首次提交。

**问题**　Apple M 系列上的 Arm 与 TSO 线程可以共享内存。真正要查的是一个模式的写入，能否按预期顺序被另一个模式读到。

**方法**　下图的 (c) 检查跨模式通信。Arm 线程先写数据 `x`，再用释放写入 `STLR` 发布标志 `y`。TSO 线程先读标志，再读数据。两边各约束一段顺序，合起来就禁止「看到新标志，却读到旧数据」。

下图的 (d) 和 (e) 则检查同一线程内的写后读顺序。两条 TSO 线程各写一个地址，再用 `LDAR` 读对方写的地址。在 (d) 中，`STLR` 到 `LDAR` 要保持顺序，两边同时读到旧值被禁止。在 (e) 中，普通写入到 `LDAR` 没有这条约束，两边可以同时读到旧值。

<figure id="apple-figure-1" class="source-figure litmus-figure" markdown="1">

<table class="litmus-panel">
<caption>(c) Arm 发消息，TSO 接收</caption>
<thead><tr><th scope="col" class="arm">线程 0 · Arm</th><th scope="col" class="tso">线程 1 · TSO</th></tr></thead>
<tbody>
<tr><td><code>STR x = 1</code></td><td><code>LDR y → r1</code></td></tr>
<tr><td><code>STLR y = 1</code></td><td><code>LDR x → r2</code></td></tr>
<tr class="litmus-outcome"><td colspan="2"><strong>禁止</strong>　r1 = 1、r2 = 0（看到新标志却读到旧数据）</td></tr>
</tbody>
</table>

<div class="litmus-pair" markdown="1">

<table class="litmus-panel">
<caption>(d) 释放写入 + 获取加载</caption>
<thead><tr><th scope="col" class="tso">线程 0 · TSO</th><th scope="col" class="tso">线程 1 · TSO</th></tr></thead>
<tbody>
<tr><td><code>STLR x = 1</code></td><td><code>STLR y = 1</code></td></tr>
<tr><td><code>LDAR y → r1</code></td><td><code>LDAR x → r2</code></td></tr>
<tr class="litmus-outcome"><td colspan="2"><strong>禁止</strong>　r1 = r2 = 0</td></tr>
</tbody>
</table>

<table class="litmus-panel">
<caption>(e) 普通写入 + 获取加载</caption>
<thead><tr><th scope="col" class="tso">线程 0 · TSO</th><th scope="col" class="tso">线程 1 · TSO</th></tr></thead>
<tbody>
<tr><td><code>STR x = 1</code></td><td><code>STR y = 1</code></td></tr>
<tr><td><code>LDAR y → r1</code></td><td><code>LDAR x → r2</code></td></tr>
<tr class="litmus-outcome"><td colspan="2"><strong>允许</strong>　r1 = r2 = 0</td></tr>
</tbody>
</table>

</div>

<figcaption markdown="1">

**原文图 1(c)–(e) · 初值 x = y = 0。** 每列从上往下读是同一线程的指令顺序。对比 (d) 与 (e)，两条线程的读取都是 `LDAR`，唯一的指令差别是把释放写入 `STLR` 换成了普通写入 `STR`。图中列出的是作者提出的组合模型对目标结果的判定。改编自 Vijay Nagarajan 的图 1，重排并扩写了指令和判定。[原图与图注](https://arxiv.org/html/2611.21406v1#S1.F1) · [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

</figcaption>
</figure>

**结果**　作者在 Linux 下的 M1 和 M2 上各运行 936 项 litmus test，每项重复 6 亿次。529 项混合测试中，模型禁止的 348 个目标结果均未观察到。(d) 和 (e) 的实测在一条线程中用 `DMB` 替换图示的排序标注。两台机器都观察到了 (e) 的允许结果，均未观察到 (d) 的禁止结果。[图 1 的测试说明](https://arxiv.org/html/2611.21406v1#S1.F1)

**可借鉴**　做兼容模式回归时，为同一测试配两组结果：一组应被禁止，另一组通过调整一条排序条件后应当允许。**先确认允许结果确实能出现**，再解释禁止结果计数为零。

<details class="article-detail" markdown="1">
<summary>展开组合模型的两种排序约束</summary>

线程之间，各线程保留自身模式规定的程序顺序，再按共用的内存模型判断写入何时对另一条线程可见。线程内部，要检查 release、acquire 和屏障是否给 TSO 原本允许重排的写后读加上额外约束。实验支持 M1、M2 上的被测模式和指令组合。[论文的模型与实验方法](https://arxiv.org/html/2611.21406v1)

</details>

### 显微镜下的 SPEC CPU2026：程序、阶段与采样

*SPEC CPU2026 Under the Microscope: Programs, Phases, and Samples*

Peijun Li、Tianming Cui、Caiwen Ding、Yang Katie Zhao（明尼苏达大学双城分校），Pei-Hung Lin、Maya Gokhale（劳伦斯利弗莫尔国家实验室）。arXiv:2611.16376 v1，北京时间 11 月 12 日提交。

**问题**　SimPoint 找到走相似代码的区间，再取一个代表区间估计整段 CPI。指令路径相似时，如果缓存压力不同，代表点的速度仍可能失真。

**方法**　作者在 Xeon 8480+ 上单核运行 SPECrate 的 90 个 ref 输入，用硬件计数器测全程 CPI。先对照 SimPoint 的默认估计，再固定原聚类，用已量到的区间 CPI 重选代表点，单独检查选点的影响。[实验平台与测量方法](https://arxiv.org/html/2611.16376v1#S3)

**结果**　以实机全程 CPI 为基线，非平稳输入的个别采样误差达 **43%**。保留原聚类、用实机区间 CPI 重选代表点后，90 个测试输入的全程 CPI 估计误差均在 **2.6%** 内。这个重选实验依赖预先测到的区间 CPI，用来判断误差有多少来自选点。[论文采样实验](https://arxiv.org/html/2611.16376v1#S6.SS4)

**可借鉴**　评估预取器、缓存或 ROB 扩容的小幅收益时，固定输入与权重，先看结果在多个种子、选点和暖机长度下是否稳定。若选点差异与收益同量级，先补测短时慢阶段。

<details class="article-detail" markdown="1">
<summary>复核采样口径、随机种子和 CPython 例子</summary>

作者用每 10 毫秒的硬件计数器观察运行阶段，另将每一亿条指令区间的实测周期与基本块向量对齐。改变随机种子可让一个输入的误差范围跨 58 个百分点。[实验方法与选点敏感性](https://arxiv.org/html/2611.16376v1#S6.SS3)

> “choosing each representative by measured CPI brings every input within 2.6%”  
> —— Peijun Li 等，[论文摘要](https://arxiv.org/abs/2611.16376)

一个具体例子是 `714.cpython_r/2`。占 **36% 权重** 的簇，默认选出的代表区间 IPC 为 **3.02**，整个簇平均为 **3.57**。默认估计的全程 CPI 因此出现 **+7.0%** 的误差。基本块向量能描述执行了哪些代码；遇到执行相似代码却跑得快慢不同的区间，还得另用硬件计数或性能指标交叉检查。这一例说明「代码相似」和「性能有代表性」是两个需要分别验证的条件。[论文第 VI-D 节与图 9](https://arxiv.org/html/2611.16376v1#S6.SS4)

</details>

## 开源项目与规范

### 香山：等 `vstart` 可见后再恢复译码

11 月 17 日合入昆明湖 v2。

**问题**　`vsetvl` 刷新后，`vtype` 和提交后的 `vstart` 到达译码的拍数不同。第一条新向量指令如果早一拍进入，可能用到旧的起始元素。

**方法**　ROB 把现有恢复标志再打一拍。标志有效期间，译码暂停接受新指令，出端 `valid` 也保持关闭。[合入差分](https://github.com/OpenXiangShan/XiangShan/commit/c95fe3f48312410138f5a48bfec55cb9cf26da8a)

**结果**　这条刷新恢复路径多等一拍，给提交后的 CSR 留出可见时间。公开证据支持这一拍的控制变化，全程序 IPC 尚未量化。

**可借鉴**　波形先放三根信号：**刷新事件、正确 `vstart` 可见、首条新指令被接受**。用不同的旧值和新值，加连续重定向或背压。断言消费的是正确版本，具体等几拍由各实现决定。

<details class="article-detail" markdown="1">
<summary>从 ROB 跟到 DecodeStage 的两个门控</summary>

顺着信号读补丁，可以从 ROB 的 `isVsetFlushPipeReg2` 追到送给译码的 `isResumeVType`。在译码端，这个标志置位时，入口 `canAccept` 和出端指令的 `valid` 都被挡住。因此这次改变的是新指令可以被接收、发出的时刻，CSR 的存储结构没有改动。[ROB 产生恢复标志](https://github.com/OpenXiangShan/XiangShan/blob/c95fe3f48312410138f5a48bfec55cb9cf26da8a/src/main/scala/xiangshan/backend/rob/Rob.scala#L625-L638) · [译码入口门控](https://github.com/OpenXiangShan/XiangShan/blob/c95fe3f48312410138f5a48bfec55cb9cf26da8a/src/main/scala/xiangshan/backend/decode/DecodeStage.scala#L192-L213) · [译码出端有效位](https://github.com/OpenXiangShan/XiangShan/blob/c95fe3f48312410138f5a48bfec55cb9cf26da8a/src/main/scala/xiangshan/backend/decode/DecodeStage.scala#L230-L236)

</details>

### 香山：将 ROB 入队的数据选择移到 bank

11 月 16 日合入昆明湖 v3。

**问题**　ROB 每个表项都从同一批入队请求中挑头 µop，宽数据选择逻辑随表项数量重复。

**方法**　指针低位先选 bank，每个 bank 只生成一份头 µop 数据。各表项仍用完整指针生成自己的写使能，并保留重定向门控。实现要求 **bank 数为 2 的幂，重命名宽度不超过 bank 数**。[ROB 合入差分](https://github.com/OpenXiangShan/XiangShan/commit/3614486179322ff060a532608b1e8341e9a929b0)

**结果**　宽数据选择器从每项一份收敛到每 bank 一份；面积、时钟和功耗仍要实际综合、布线才能判断。

**可借鉴**　开始前用满宽入队和指针绕回校验独热选择；比较物理结果时，把选择器面积与 bank 内数据扇出、最慢路径放在同一张表里。这样能判断逻辑节省是否被布线吃掉。

<details class="article-detail" markdown="1">
<summary>查看 enqOH、bankHeadUop 和写入约束</summary>

把数据和控制两条路径分开看。改动前，表项用自己的 `enqOH` 从所有请求里选数据。改动后，低位匹配先生成每个 bank 的 `bankHeadUop`，表项用 `i % bankNum` 取共享数据；写不写表项仍由原来的 `enqOH` 和重定向门控决定。源码的独热断言检查逐项选择，配置约束则在生成硬件时检查。补丁修改了入队 RTL，可从这些位置核对，差分没有附带新的测试文件。[选择器、写使能和配置约束](https://github.com/OpenXiangShan/XiangShan/blob/3614486179322ff060a532608b1e8341e9a929b0/src/main/scala/xiangshan/backend/rob/Rob.scala#L161-L188) · [前后差分](https://github.com/OpenXiangShan/XiangShan/commit/3614486179322ff060a532608b1e8341e9a929b0)

</details>

### LLVM：让 Arm SVE 结构化加载处理固定长向量去交错

北京时间 11 月 12 日合入 LLVM 主干。

**问题**　内存里排的是 `A0, B0, A1, B1`，消费者要分别拿 A、B。旧后端在总宽度 64 位的固定长结果上漏了加载与去交错的合并。

**方法**　当目标选择用 SVE 实现固定长向量时，后端用可伸缩向量容器承载结果，再用谓词只激活需要的 lane。一次 SVE 结构化加载分出各通道，结果裁回原长度。这次合入的路径跳过已有掩码的加载。

**结果**　汇编回归在指定的 streaming-compatible 配置下检查 8×i8 用例应生成 `ptrue … vl8` 和相应的 `ld2b`、`ld3b` 或 `ld4b`。这里核对的是指令选择，芯片吞吐需另测。[LLVM 回归用例](https://github.com/llvm/llvm-project/blob/5b0dc777e90c28aeea139a2326a4c8bdc101102d/llvm/test/CodeGen/AArch64/sve-fixed-vector-interleave.ll#L1-L48)

**可借鉴**　验证先给每个通道填可辨认标签，再扫向量长度和边界。评估收益则用同一批数据对比合并前后的每元素周期与访存流量。

<details class="article-detail" markdown="1">
<summary>查看后端匹配和 masked load 的退出条件</summary>

入口是 `performVectorDeinterleaveCombine`。新增条件先检查目标是否选择用 SVE 实现固定长向量，再为加载选容器、生成适配固定元素数的谓词。源节点若已是 masked load，新增的固定长路径会提前退出。新增回归则从一段 LLVM IR 开始，用 FileCheck 核对预期汇编。这样可以区分代码写明的适用边界，以及这次补丁真正覆盖的路径。[后端匹配与谓词](https://github.com/llvm/llvm-project/blob/5b0dc777e90c28aeea139a2326a4c8bdc101102d/llvm/lib/Target/AArch64/AArch64ISelLowering.cpp#L32041-L32087) · [固定长去交错回归](https://github.com/llvm/llvm-project/blob/5b0dc777e90c28aeea139a2326a4c8bdc101102d/llvm/test/CodeGen/AArch64/sve-fixed-vector-interleave.ll#L1-L48)

</details>

### RISC-V Zama16b：澄清非对齐原子性的适用范围

11 月 14 日合入 ISA 手册主干。

**问题**　非对齐访问的原子性受地址区域和访问类型共同约束。旧措辞容易被读成所有主存都要满足。

**方法**　特权手册对齐既有 Zama16b 定义，把 MAG16 支持要求写明为同时**可缓存、参与一致性**的主存区域。

**结果**　这是规范适用范围的澄清。向量访存即使落在同一个粒度内，也没有由这项属性得到原子性保证。[规范差分](https://github.com/riscv/riscv-isa-manual/commit/21ee747cb7c0815a5f60daee3d13b83c9a468517)

**可借鉴**　把原子性回归分成三个维度：区域属性、指令类别、访问字节是否跨粒度边界。特别是标量路径向量化时，检查调用方是否依赖原有原子性。

<details class="article-detail" markdown="1">
<summary>核对手册里的适用条件和向量例外</summary>

读差分时，先找 MAG16 这条规范性要求，再看前面关于向量访存的说明。改动发生在 `machine.adoc`，工程上要更新的是适用条件和验证预期，不能据此推断某个处理器已实现了这项属性。区域属性、指令类型和粒度内的字节覆盖仍需要一起检查。[手册的原子性条件与向量例外](https://github.com/riscv/riscv-isa-manual/blob/21ee747cb7c0815a5f60daee3d13b83c9a468517/src/priv/machine.adoc#L3109-L3117)

</details>

### GCC RVV：阻止宽、窄源操作数的架构寄存器重叠

11 月 11 日进入 GCC 主干，提交 `r17-6133`。

**问题**　`vwadd[u].wv` 的 `vs2` 以两倍 SEW 读取，`vs1` 以 SEW 读取。窄源即便起始编号不同，也可能落在宽源寄存器组里。

**方法**　GCC 在宽源 `vs2` 的指令模板上采用 `Wn4` 约束，让分配器识别两段占用范围的冲突。[GCC 主干差分](https://gnu.googlesource.com/gcc/+/2e046596d270a5c61b8a02e18e69211a04453d16)

**结果**　修复针对的是同一架构寄存器被按两种有效元素宽度作为源读取的保留编码，指令语义仍按 RVV 规范。

**可借鉴**　扫合法 LMUL 和 SEW，再用较高寄存器压力触发分配。核对的是宽、窄**源寄存器组的交集**；目的操作数的重叠按相应规则单独检查。

<details class="article-detail" markdown="1">
<summary>用 vwadd.wv 反例核对寄存器组</summary>

提交举的例子是 `vwadd.wv v4,v4,v6`。在宽源从 `v4` 开始、实际占用范围包含 `v6` 的配置下，窄源 `v6` 就与宽源重叠；单看起始寄存器，`v4` 和 `v6` 看起来仍然不同。差分改的是 `gcc/config/riscv/vector.md` 中操作数 3 的约束。这次提交未增加测试文件，能够直接核对的是编译器模板的约束变化和作者给的反例。[触发实例及单文件差分](https://gnu.googlesource.com/gcc/+/2e046596d270a5c61b8a02e18e69211a04453d16) · [修改后的指令模板](https://gnu.googlesource.com/gcc/+/2e046596d270a5c61b8a02e18e69211a04453d16/gcc/config/riscv/vector.md)

</details>

## 专利

### AMD：重用阵列做矩阵与向量乘积

US20260359386A1，*Matrix Engine with Matrix and Vector Products*。申请人为 Advanced Micro Devices, Inc.，主张 2025-03-13 的美国临时申请优先权，2026-11-12 公开。

**问题**　连续两段矩阵或向量运算复用一个中间值时，经过阵列外写出、读回会占用额外通路。

**方法**　专利披露了一条局部反馈路径：结果寄存器留住中间值，下一指令用选择器把它接回阵列输入，其余操作数照常进来。[公开文本，权利要求 4](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/20260359386#page=16)

**披露的效果**　可直接消费该中间值的序列，理论上能省一组外部访问。申请文本给出结构关系，未报告芯片时序或能耗。

**可借鉴**　先在一个算子链上标记原有的写出、读回和首次可用拍。加入反馈后，比较省下的搬运能否抵过选择器延迟；同时检查有第二个消费者时，这个值还能否按原时点被看见。

<details class="article-detail" markdown="1">
<summary>查看独立项、反馈项和矩阵／向量操作分支</summary>

- **独立项 1** 写的是多个算术单元和一个按指令选择向量输入序列的选择器。沿着这一项可先找到「指令 → 选择器 → 运算单元」。
- **从属项 2、3** 加上第二个选择器及行列方向。**从属项 4** 回引第 2 项，另加结果寄存器输出接回第一选择器。这就是正文中“中间值回送”的出处。
- **从属项 6–8** 沿另一条链增加可选输入序列，并分别对应矩阵乘和向量乘。**从属项 9** 直接回引第 1 项来限定转置。按回引编号阅读时，这两支各自的限定就能分清。[USPTO 公开文本第 16–17 页](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/20260359386#page=16)

</details>

### Qualcomm：矩阵单元直接消费线性寄存器布局

US20260359928A1，*Techniques for Supporting Different Register Layouts in Graphics Processing Units (GPUs)*。申请人为 Qualcomm Incorporated，美国申请日为 2025-05-12，公开首页未列另行优先权，2026-11-12 公开。

**问题**　矩阵单元和后续向量指令要求不同的寄存器排布时，结果写回后常要先重排才能用。

**方法**　让矩阵单元直接读取线性行或列布局，并保留线性写回的选择。以 GEMM 紧接逐行激活为例，结果可按行写回，交给向量指令直接读取。[公开文本，权利要求 1、7、8](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/20260359928#page=21)

**披露的效果**　直接交接依赖生产者和消费者对布局达成一致。申请文本没有给出实际节省的指令数或端口代价。

**可借鉴**　用同一个 GEMM 分别接逐行激活和逐列归约，比较**重排指令数、寄存器端口占用、bank 冲突**。数据能不能直接交给下一条指令，比只比较矩阵单元的峰值更接近设计取舍。

<details class="article-detail" markdown="1">
<summary>核对线性读入、编译选择和写回各自的条款</summary>

- **独立项 1** 从内存装入数据，让不同的矩阵线落到不同 GPR，由矩阵乘单元使用，每个执行线程（fiber）处理相应的一条线。**从属项 2、3** 各自把矩阵线指定为行或列，两项分别回引第 1 项。
- **从属项 4、6** 将编译器选择与向量运算及指令交错关联。**从属项 7** 单独从第 1 项引出线性写回，**第 8 项**再将写回选择和后续向量运算关联。
- 用「一次读入 → 矩阵乘 → 写回 → 激活」做图解时，前半段有独立项 1 的对应动作，后半段使用第 7、8 项提供的额外条件。方案能否少做重排还要看后继算子的布局，逐行激活是本文选的说明例子。[USPTO 公开文本第 21–22 页](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/20260359928#page=21)

</details>

## AI 工程实践

### 办公：把两版公开材料整理成有出处的差异表

**问题**　两版规格的参数分散在图、脚注和表格里，单位不同也容易被误判为设计改动。

**方法**　让 AI 逐项写出原值、单位、来源位置和归一化值，工程师据此回查图和表。[MKA 公开评估](https://blog.google/innovation-and-ai/models-and-research/gemini-models/multimodal-knowledge-agent-benchmark/)

**示例结果**　自构造的总线宽度 64 bit 和 8 byte 可标为等值，队列深度从 8 到 16 则应列为变更，两处位置都保留在差异表中。

**可借鉴**　让 AI 整理可点回原文的差异行，由工程师逐行确认。找不到图号或单元格的记录留待复核。

### RTL：把时序违例变成单一、可验证的改动

**问题**　RTL 功能通过后，最慢路径仍可能赶不上时钟。一次改多处，很难归因于哪项改动，也难发现接口契约被改变。

**方法**　AccelClosure 让 AI 根据路径与面积报告提出局部修改，再经过 Verilator、Yosys 和 OpenROAD 重查功能、综合与布局布线。[东京 RISC-V Day 官方摘要](https://riscv.or.jp/en/risc-v-day-tokyo-2026-autumn-2/)

**结果**　作者的 Sky130HD、8×8 INT8 阵列以 200 MHz 为目标，初始 setup slack 为 −2.19 ns，报告的布线后估算最高频率为 212 MHz。数字来自作者的实现估计。

**可借鉴**　一次只改一个可回退的地方。若拆了流水拍，重跑反压、清空和复位测试，并把最慢路径、面积、接口延迟放进同一张表。

<details class="article-detail" markdown="1">
<summary>拆拍前要守住的握手条件</summary>

除复位或清空等协议定义的例外外，若 `valid=1, ready=0`，下拍仍应保持 `valid` 与 `payload`，直到完成握手。固定工艺、时钟预算和回归，才能把变化归因到这一次修改。作者另比较了不同阵列尺寸在 TinyBERT 与 TinyLlama 上的表现，本刊未重跑 EDA 流程。[作者公开说明](https://riscv.or.jp/en/risc-v-day-tokyo-2026-autumn-2/)

</details>

### 处理器架构：比较两种向量写回组织

**问题**　共享写回队列只从头取。如果第二个结果与第一个抢同一 bank，后面原本能写另一 bank 的结果也会被挡住。

**方法**　两个 bank 各有一条写端口，预装 `B0:A, B0:B, B1:C, B1:D`。比较只从队首连续取的共享 FIFO，以及每 bank 各取一个的队列。

**结果**　手算和小队列模型分别得到 3 拍、2 拍。下面逐拍列出写入项。模型使用固定的请求顺序和预装请求，省略总线、仲裁和投机清除的代价。

| 写回规则 | 第 1 拍 | 第 2 拍 | 第 3 拍 |
|---|---|---|---|
| 共享 FIFO、队首连续取 | A | B、C | D |
| 每个 bank 独立取队首 | A、C | B、D | 完成 |

**可借鉴**　给微基准分别喂均匀流量和热点 bank 流量，固定入口带宽、VLEN 和深度。除了延迟分布，再查被分支清除的请求是否还能唤醒 scoreboard。[相关 bank 复用讨论](https://github.com/OpenXiangShan/XiangShan/pull/6930)

## 线索附录

<details markdown="1">
<summary>展开补充线索</summary>

- **RVV 数学函数接入 NumPy。** *Accelerating NumPy’s Transcendental Functions on the RISC-V Vector Extension with SLEEF*。Masato Urushibara（N High School）与合作者 Ryotaro Ohara（署名单位待核实），11 月 11 日东京会场摘要。RVV 1.0 下将 FP64 三角和指数函数接入 SLEEF，报告相对传统 libm 的跨函数与平台几何均值为 1.68×，最大误差 1.0 ULP。SpacemiT A100 上几何均值 2.11×，cos 最高 4.19×。材料为公开日程摘要。[主办方摘要](https://riscv.or.jp/en/risc-v-day-tokyo-2026-autumn-2/)
- **LLVM 五路交错访存测试。** 11 月 16 日合入新增的 LLVM IR 回归覆盖，适合检查非二次幂交错宽度；此变更只反映测试覆盖。[LLVM PR #239573](https://github.com/llvm/llvm-project/pull/239573)
- **Qualcomm 转置加载。** US20260359927A1，Qualcomm Incorporated，2025-05-08 美国申请，首页未列另行优先权，2026-11-12 公开。转置加载在读入时把同一源向量的元素分配到不同执行线程的同号寄存器。它与前述线性布局申请是不同的披露，尚未核验同族或实现成本。[USPTO 原件](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/20260359927)
- **QEMU 矢量点积 RFC。** 初版在实验窗口前；11 月 10 日 22:12 的邮件提供浮点转换交叉验证反馈，未给出测试规模，仍为 RFC 讨论。[公开邮件原文镜像](https://lore-kernel.gnuweeb.org/qemu-devel/20261110141239.46244-1-chao.liu@processmission.com/T/#u)

</details>

## 厂商公开动态

### NVIDIA：用 Vera CPU 机架承接数据密集型科学计算

NVIDIA 于 11 月 16 日在 SC26 公开方案。

**问题**　科学计算扩到多节点后，要同时搬本地内存的数据并等待跨节点交换。内存与通信负担可能分别限制进度。

**方法**　Vera 液冷机架把 192 颗 CPU、高带宽内存和 Quantum-X800 InfiniBand 集成起来，部分集合通信可交给网络处理。[NVIDIA SC26 公告](https://nvidianews.nvidia.com/news/nvidia-expands-vera-cpu-platform-to-accelerate-scientific-discovery)

**公开结果**　192 颗 CPU 的机架聚合峰值内存带宽超过 **200 TB/s**。NVIDIA 的 OpenFOAM v2606 测试用 motorBike Large 网格，对照双路 AMD EPYC 9755，报告 Vera 达到 **1.9× 性能**和 **2.4× 每瓦性能**。两组数字分别来自机架配置与双路系统测试。

**可借鉴**　同一网格与求解条件下，按单插槽、多插槽、多节点逐级量，依次看本地带宽、NUMA 和集合通信各耗时多少。

<details class="article-detail" markdown="1">
<summary>展开单颗 CPU 参数与比较口径</summary>

OpenFOAM 对照两边均为双路系统，均使用 GCC 16.2。Vera 的每颗 CPU 有 88 核，配 LPDDR5X-9600；EPYC 9755 每颗 128 核，配 DDR5-6400。比较时核心数和内存系统都不同，这组结果反映整机配置在指定网格上的差异。测试简报第 10–11 页为负载及结果，第 23 页为平台与软件版本。[OpenFOAM 结果](https://dam-cdn.nvd.orangelogic.com/AssetLink/b8ip35u047q24s1600wns7rtilf0qcru.pdf#page=10) · [测试配置](https://dam-cdn.nvd.orangelogic.com/AssetLink/b8ip35u047q24s1600wns7rtilf0qcru.pdf#page=23)

</details>

<details class="article-detail" markdown="1">
<summary>AMD 补充线索：原生与模拟 FP64 的负载评估</summary>

AMD 在 11 月 16 日的 SC26 公告中提到，比较覆盖 11 个应用或基准的 47 种配置，部分科学负载仍需原生 FP64。当前公开公告提供的是测试范围和厂商判断。若要评估低精度模拟路径，可固定收敛准则，再分别量算子时间、残差、迭代次数、回退比例和端到端时间，找出拆分与重组在哪些负载会抵消核内的收益。[AMD SC26 公告](https://newsroom.amd.com/news/sc26-amd-expands-frontier-scientific-computing/)

</details>
