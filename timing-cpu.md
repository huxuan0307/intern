# 构建

```bash
scons build/RISCV/gem5.opt -j$(nproc)
```



# 使用

默认将使用不带cache的AtomicSimpleCPU模型

```bash
build/RISCV/gem5.opt configs/example/se.py --cmd binary_to_run [--options OPTION]
```

使用TimingSimpleCPU模型

```bash
build/RISCV/gem5.opt configs/example/se.py --cpu-type=TimingSimpleCPU ...
```

使用MinorCPU模型

```bash
build/RISCV/gem5.opt configs/example/se.py --cpu-type=MinorCPU ...
```

使用带cache的MinorCPU模型

```bash
build/RISCV/gem5.opt configs/example/se.py --cpu-type=MinorCPU --caches ...
```

使用O3CPU模型（因vector config指令的译码相关问题，暂时未做支持）

```bash
build/RISCV/gem5.opt configs/example/se.py --cpu-type=O3CPU ...
```



# 指令支持情况

![MEM](img/MEM.svg)

![OPIVV](img/OPIVV.svg)

![OPMVV](img/OPMVV.svg)

![OPFVV](img/OPFVV.svg)