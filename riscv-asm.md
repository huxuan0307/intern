# C/C++中的riscv内联汇编

## 参考资料

https://llvm.org/docs/LangRef.html

https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html

## 拓展内联汇编

### 格式

```c++
__asm__ [asm-qualifiers] (
	AssemblerTemplate 
    : OutputOperands 
    : InputOperands
    : Clobbers
    : GotoLabels
);
```

### 汇编修饰词 asm-qualifiers

+ volatile 告知编译器禁止优化
+ inline 编译器会尽可能生成最小长度的二进制代码
  https://gcc.gnu.org/onlinedocs/gcc/Size-of-an-asm.html#Size-of-an-asm
+ goto 用于跳转

### 汇编模板 AssemblerTemplate

类似格式化字符串，使用%0，%1来占位，将C/C++代码中的变量与汇编语句中的寄存器/内存单元绑定。

```c++
__asm__("vadd.vv %0, %1, %2, %3.t;" : "=vr"(vd) : "vr"(vs2), "vr"(vs1), "vm"(vmask)); 
```

### 输出输入操作数 OutputOperands，InputOperands

由逗号分隔的表达式，可以为空

格式：["constraint"(variable)]*

+ constraint 对寄存器或值的约束
+ variable C/C++代码中的变量

### 输出候选寄存器 Clobbers 

由逗号分隔的表达式，可以为空

编译器将在Clobbers中选择寄存器作为与输出操作数绑定的寄存器

### 跳转标签 GotoLabels

用作跳转语句的label

### 约束 constraint 

约束 指明汇编模板中对应占位符的类型和需求，因指令集而异。

#### RISC-V的特性约束

+ A：地址操作数，将使用通用寄存器，不会添加地址偏移

+ I：12bit的带符号整数操作数，构成汇编中的imm12立即数字段

+ J：可能是跳转指令使用的20位无符号立即数（LLVM 文档中的原文是"A zero integer immediate operand.")

+ K：5bit的无符号整数操作数，构成汇编中的uimm字段，用作位移立即数。（可能还有其它用途。RV64的uimm是6位，并不支持6位K约束，此时还是需要使用I）

+ f：浮点寄存器

+ r：通用寄存器

+ vr：向量寄存器

+ vm：用作mask的向量寄存器


#### 通用的约束前缀

+ "=": output操作数的约束前缀，例如"=r"

+ "=&": output操作数的约束前缀，要求目的寄存器与所有源操作数寄存器不同
  rvv中，有些指令要求目的寄存器与源寄存器不能有overlap关系。

  > For any **vrgather** instruction, the destination vector register group cannot overlap with the source vector register groups, otherwise the instruction encoding is reserved.

+ "+": output操作数的约束前缀，指明该操作数同时是输入和输出操作数。
  rvv中，对三操作数指令vmacc.vv，需要对vd使用"+"约束前缀。否则开启优化时，加载vd的指令会被编译器认为是未使用的加载而优化掉。

  ```c++
  __asm__("vle8.v %0, (%1);" : "=vr"(vd) : "vr"(addr)); // O2优化下，会被优化
  __asm__("vmacc.vv %0, %1, %2;" : "=vr"(vd) : "vr"(vs2), "vr"(vs1));
  ```

  

### V拓展使用汇编的例子

代码都来自我正在为V拓展实现的测试集 https://github.com/huxuan0307/riscv-vector-tests

#### 指定寄存器

```c++
__asm__("vsetvli %%t1, zero, e8, m8;" : "=r"(vl)); 
```
"%%"被转义为"%"，于是对应的汇编为

```asm
vsetvli %t1, zero, e8, m8;
```

t1寄存器将保存变量vl。



#### 突破rvv intrinsic的类型限制

当vlen=128，sew=64，vlmul=1时，对于与mask相关的指令，单次运算仅仅使用/产生2bit mask。而内存访问是以字节为单位的，需要对vmask位移。

`rvv intrinsic`没有提供源操作数类型为`vboolx_t`的位移函数，只好用内联汇编来使用位移指令。

```c++
void vadd_vv_u64m1_m_vec(
    uint64_t* d, uint64_t* s2, uint64_t* s1,
    uint8_t* mask, size_t n
) {
	for (i = 0; i < n;) {
        size_t vl = vsetvl_u64m1(n);
        vbool64_t vmask = vlm_v_b64(&mask[i/8], vl);
        size_t offset = i % 8;
        __asm__("vsrl.vx %0, %1, %2;" :"=vm"(vmask) :"vm"(vmask),"r"(offset));
        vuint64_t vs1 = vle_v_u64m1(&s1[i], vl);
        vuint64_t vs2 = vle_v_u64m1(&s2[i], vl);
        vuint64_t vd = vadd(vmask, vundefined_u64m1(), vs2, vs1, vl);
        vse_v_e64m1(vmask, &d[i], vd, vl);
        i += vl;
    }
}
```



#### 合并相同汇编格式但rvv intrinsic函数参数顺序不同的指令

不同的rvv intrinsic，使用rvv intrinsic需要定义两个不同的函数类型

```c++
vint8m1_t vadc_vvm_i8m1 (vint8m1_t op1, vint8m1_t op2, vbool8_t carryin, size_t vl);
vint8m1_t vsbc_vvm_i8m1 (vint8m1_t op1, vint8m1_t op2, vbool8_t carryin, size_t vl);
vint8m1_t vmerge_vvm_i8m1 (vbool8_t mask, vint8m1_t op1, vint8m1_t op2, size_t vl);
```

相同的汇编格式

```asm
vmerge.vvm vd, vs2, vs1, v0
vadc.vvm   vd, vs2, vs1, v0
vsbc.vvm   vd, vs2, vs1, v0
```

使用宏替换指令名称即可

```c++
#define vop(op) \
__asm__(#op ".vvm %0, %1, %2, %3;" : "=vr"(vd) : "vr"(vs2), "vr"(vs1), "vm"(vmask)); 

```



#### 其它的例子

```c++
// 解决vboolx_t 与 vintx_t之间无法类型转换
vbool8_t vmask;
__asm__("vle8.v %0, (%1);" : "=vm"(vmask) : "r"(addr_m));

// 对mask寄存器需要指定.t 后缀，实际上是字符串的拼接
__asm__(#op ".vv %0, %1, %2, %3.t;" : "=vr"(vd) : "vr"(vs2), "vr"(vs1), "vm"(vmask)); 
```

V拓展对一些指令有特殊的要求，目的寄存器不能与源寄存器重合。例如以下报错：

```bash
<inline asm>:1:11: note: instantiated into assembly here
        vwsub.vv v8, v9, v8;
                 ^
```

vrgather.vv vwsub.vv等指令不允许input与output寄存器的overlap

```c++
// 用"&"前缀约束解决input与output的overlap问题
__asm__(#op ".vv %0, %1, %2;" : "=&vr"(vd) : "vr"(vs2), "vr"(vs1));
```

对三操作数指令的vd寄存器加"+"前缀约束，避免加载指令被优化掉

```c++
// 用"+"前缀约束标记vd，来避免编译器对加载vd指令的优化
__asm__("vle8.v %0, (%1);" : "=vr"(vd) : "vr"(addr)); // -O2下，会被优化
__asm__("vmacc.vv %0, %1, %2;" : "=vr"(vd) : "vr"(vs2), "vr"(vs1));
//                                ^改成+
```


