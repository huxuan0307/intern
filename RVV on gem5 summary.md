# RVV on gem5总结

## 基本参数

| 参数           | 支持情况                              |
| -------------- | ------------------------------------- |
| 最大元素位宽   | 64位，即Zve64拓展                     |
| 向量寄存器长度 | 256位，即Zvl256b。暂不支持配置VLEN    |
| SEW            | 8，16，32，64                         |
| LMUL           | mf8，mf4，mf2，m1，m2，m4，m8         |
| 数据格式       | INT8，INT16，INT32，INT64，FP32，FP64 |



## CSR寄存器

| CSR    | Privilege | 支持情况                                                     |
| ------ | --------- | ------------------------------------------------------------ |
| vstart | URW       | 支持读写，但不支持中断/异常时保存现场                        |
| vxsat  | URW       | 支持读写，隐式写入vxsat的指令正在debug(vsadd等)              |
| vxrm   | URW       | 支持读写，暂未实现隐式读取vxrm的指令(vaadd, vssrl, vnclip等) |
| vcsr   | URW       | 支持读写，且与vxsat，vxrm同步更新                            |
| vl     | URO       | 支持读取，隐式读取vl的指令都可以正确执行                     |
| vtype  | URO       | 支持读取，已经实现向量非法指令异常，有待优化实现方式         |
| vlenb  | URO       | 支持读取                                                     |



## vmu/vtu

| 已支持                                                       | 待测试             |
| ------------------------------------------------------------ | ------------------ |
| `vle<eew>.v`,` vlse<eew>.v`, `vsse<eew>.v`, `vred*`, `vfred*` | 其它已经实现的指令 |



## 进度

### 指令实现情况

<p align="middle">已实现指令列表 </p>

| 类别              | 指令                                                         |
| ----------------- | ------------------------------------------------------------ |
| Vector Config     | vsetvl，vsetvli，vsetivli                                    |
| OPIVV/OPIVX/OPIVI | vadd, vsub, vrsub, vmin, vminu, vmaxu, vmax, vand, vor, vxor, <br />vadc, vmadc, vsbc, vmsbc, <br />vmerge, vmv,<br />vmseq, vmsne, vmsltu, vmslt, vmsleu, vmsle, vmsgtu, vmsgt<br />vsrl, vsra |
| OPFVV/OPFVF       | vfadd, vfsub, vfrub, vfmin, vfmax, vfdiv, vfdivu, vfmul, <br />vfsgnj, vfsgnjn, vfsgnjx,<br />vfredusum, vfredosum, vfredmin, vfredmax, <br />vmfeq, vmfle, vmflt, vmfne, vmfgt, vmfge, <br />vfmadd, vfnmadd, vfmsub, vfnmsub, vfmacc, vfnmacc, vfmsac, vfnmsac<br />vfwadd, vfwsub, vfwadd.w, vfwsub.w, vfwmul,<br />vfwmacc, vfwnmacc, vfwmsac, vfwnmsac |
| OPMVV/OPMVX       | vredsum, vredand, vredor, vredxor, vredminu, vredmin, vredmaxu, vredmax<br />vdivu, vdiv, vremu, vrem, vmulhu, vmul, vmulhsu, vmulh, <br />vmadd, vnmsub, vmacc, vnmsac<br />vwaddu, vwadd, vwsubu, vwsub, vwaddu.w, vwaddu, vwsubu.w, vwsub.w<br />vwmulu, vwmulsu, vwmul, vwmaccu, vwmaccus, vwmaccsu |
| Vector Load/Store | vle\<eew\>.v, vse\<eew\>.v, <br />vlm.v, vsm.v, <br />vl\<nf\>re\<eew\>.v, vs\<nf\>r.v, <br />vlse\<eew\>.v, vsse\<eew\>.v |



<p align="middle"> 未实现指令列表 </p>

|       类别        |                             指令                             |
| :---------------: | :----------------------------------------------------------: |
| OPIVV/OPIVX/OPIVI | vrgather, vrgatherei16, vslideup, vslidedown,<br />vsaddu, vsadd, vssubu, vssub, vsmul, vssrl, vssra,<br />vnsrl, vnsra, vnclipu, vnclip,<br />vwredsumu, vwredsum |
|    OPFVV/OPFVF    | vfslide1up, vfslide1down, vfmv.f.s, vfmv.s.f,<br />vfcvt, vfwcvt, vfncvt,<br />vfsqrt, vfrsqrt7, vfrec7, vfclass, vfmerge, vfmv<br /> |
|    OPMVV/OPMVX    | vaaddu, vaadd, vasub, vslide1up, vslide1down, <br />vmv.x.s, vpopc, vfirst, vmv.s.x, vzext, vsext, <br />vmsbf, vmsof, vmsif, viota, vid, vcompress, <br />vmandnot, vmand, vmor, vmxor, vmornot, vmnand, vmnor, vmxnor, |
| Vector Load/Store |                        segment, index                        |

已经实现的V拓展指令约250条，约占全部指令的60%。

目前未实现指令中，定点包括饱和算术指令(vsaddu等)，收缩位移指令(vnsrl等)，拓位规约指令(vwredsumu等)，拓位指令(vzext等)，平均算术运算指令(vaaddu等)，掩码位运算指令(vmand等)，单操作数指令(vfirst等)；浮点包括类型转换指令(vfcvt等)，单操作数指令(vfsqrt等)；访存还有index未完成debug，segment未实现。



### CPU Model支持情况

已实现的指令均支持`AtomicSimpleCPU`，`TimingSimpleCPU`，`MinorCPU`，`O3CPU`模型。其中`O3CPU`模型中`vset*`指令对译码阶段的阻塞方案有待改进。



## 未来一月计划（到8.15）

|   日期   |                           进度安排                           |
| :------: | :----------------------------------------------------------: |
|  ~7.25   | index访存，饱和算术指令(vsaddu等)，拓位规约指令(vwredsumu等)，掩码位运算指令(vmand等)，收缩位移指令(vnsrl等) |
| 7.26~8.1 | 拓位指令(vzext等)，平均算术运算指令(vaaddu等)，定点/浮点单操作数指令 |
| 8.2~8.8  |                 浮点类型转换指令(vfcvt等)，                  |
| 8.9~8.15 |             vrgather，vslide，vfslide等零碎指令              |

segment load/store随缘吧，假如灵感突现&有时间做。