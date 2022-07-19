# intro

## Vector Engine

依赖MinorCPU的协处理器，类似RoCC实现。未使用gem5提供的译码、指令调度与执行框架，修改与移植较为复杂

# design

## constraint

+ RVV指令集
  + 可以从寄存器配置vtype
    + 部分向量配置指令在执行时才能得到全部配置信息
    + 向量算术与访存指令需要在译码时确定配置信息
  + LMUL>1时读写寄存器数量太多
    + vmacc最多要读取25个向量寄存器，写回8个向量寄存器
+ gem5框架自身
  + 译码仅有一次，须在译码阶段确定指令执行的全部信息
    + 无法在某指令执行后修改已译码指令的译码信息
  + 时序CPU模型
    + 单次访存数据上限
    + 每条指令对应至多一次访存请求
      + 存储系统反馈后，该指令执行完毕
  + 支持重命名的CPU模型
    + vmu/vtu需要读取原vd
+ 物理实现
  + 每条指令读写寄存器个数
    + 读不超过4
    + 写一般为1

## target

+ 使用gem5译码、指令调度与执行框架
+ 在gem5中SimpleCPU、MinorCPU、O3CPU等模型上运行
+ 少对已有CPU模型做微架构修改
+ 拆分uop，实现RVV指令
+ 较为精确地模拟RVV指令的时序

## difficulty

+ 在乱序CPU模型中高效执行RVV
+ 较难实现的指令
  + vrgather等
    + 输出一个元素最多需要读10个向量寄存器
      + 8 vs2, 1 vs1, 1 v0
  + vslide
+ 平衡寄存器利用率和整体执行效率
  + 生成mask的uop仅写回2-16bit，远小于VLEN
  + 连续访存的单次访存带宽不够VLEN
  + stride，index单次访存数据远小于VLEN
+ 向量访存|算术指令在译码时获得正确的配置信息
+ 兼顾RVV指令集、gem5框架、物理实现的约束
+ 测试以确保正确性
  + VectorEngine中指令实现相关bug有20+处
    + riscv-vectorized-benchmark-suite测试粒度太大且不充分
  + 测试难点
    + LMUL，vl
    + vmu/vma，vtu/vta
    + etc

# implement

## Vector Config

除SimpleCPU模型，MinorCPU与O3CPU都是尽可能多译码指令，几乎填满指令队列来应对乱序调度。

在MinorCPU的fetch2阶段添加stall信号，decoder遇到vconfig指令则暂停译码，并暂停fetch2阶段，直到vconfig指令执行且写回时恢复

在O3CPU的fetch阶段添加stall信号，decoder遇到vconfig指令则暂停译码，并暂停fetch阶段，直到vconfig指令执行且写回时恢复

## Vector Load/Store

### 连续访存

`vle<eew>.v`, `vse<eew>.v`,  `vlm.v`, `vsm.v`, `vl<nf>re<eew>.v`, `vs<nf>r.v`

uop拆分粒度随存储系统的最大位宽而定。连续访存指令较为常用，为保证乱序执行效率，解除uop之间写后读相关性，每次访存都将数据存取到中间临时寄存器中。默认情况下（VLEN=256，访存位宽64），每个向量寄存器对应4条访存指令与1条合并指令。

### 非连续访存

`vlse<eew>.v`, `vsse<eew>.v`, `vluxei.v`, `vsuxei.v`

uop拆分粒度随元素的位宽而定。stride与index访存并不常用，采用写回中间临时寄存器的方式极端情况下拆分uop数量过多（VLEN=256，LMUL=8，SEW=8时，需要256次访存，合并中间结果需要100+uop）。

因此非连续访存uop将读取原vd寄存器，并且将结果直接写回向量寄存器

## Vector Arithmetic

### 基本算术指令

`vadd.v`, `vmadd.v`

拆分uop数量为LMUL

### 拓宽/收缩算术指令

`vwadd.v`, `vwmadd.v`

拆分uop粒度随写回寄存器数量而定，写回一整个向量寄存器对应一条uop。

如`vwadd.v`，LMUL=4时写回8个向量寄存器，则拆分8条uop。LMUL=1时写回2个向量寄存器，则拆分2条uop

### 生成mask指令

`vmseq.v`

拆分uop数量为LMUL+1

uop执行结果写回临时寄存器，最后一条合并指令将结果写回向量寄存器

# summary

