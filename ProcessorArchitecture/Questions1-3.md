1. 请简述精简指令集RISC(Reduced Instruction-Set Computer)和复杂指令集CISC(complex instruction set computer)之间的区别
> 上世纪70年代IBM的John Cocke发现，处理器提供的大量指令集和复杂寻址方式其中只有20%会被经常用到，通常简单指令大都能在一个cycle内完成，基于该思想的指令集叫RISC，以前的叫CISC;
> RISC处理器通过更合理的微架构在性能上打败了CISC处理器，后来Intel设计了Pentium Pro处理器，x86指令集被解码成类似RISC指令的微操作指令(micro-operations,简称uops),
> 以后执行的过程采用RISC内核的方式，导致其性能逐渐超过同期的RISC.

2. 请简述数值0x12345678在大(Big-endian)小(Little-endian)端字节序处理器的存储器中的存储方式
> 计算机系统中是以字节为单位的，每一块内存空间对应着一个字节(8个比特位)，由于寄存器宽度大于一个字节，所以必然存在如何安排多个字节的问题;
> 大小端记忆(小小小，即低字节放在低地址为小端); 如何检测:联合体Union的存放顺序是所有成员都从低地址开始放的
```c
int checkCPU(void)
{
    union w
    {
        int a;
        char b;
    }c;
    c.a = 1;
    return (c.b == 1);
}
```
> 如果返回true，则为小端；否则为大端

3. 请简述在你所熟悉的处理器(比如双核Cortex-A9)中一条存储读写指令的执行全过程
> 经典处理器架构的流水线是五级流水线：取指、译码、发射、执行和写回

> 现代处理器在设计上都采用了超标量结构(Superscalar Architecture)和乱序执行(out-of-order)技术，极大地提高了处理器计算能力
> 超标量技术能在一个时钟周期(CPU主频的倒数)内执行多个指令，实现指令级的并行，有效提高ILP(Instruction Level Parallelism)指令级的并行效率，同时也增加了这个cache和memory层次结构的实现难度

> 一条存储读写指令的执行全过程：

> 对于写指令：
> - 指令首先进入流水线(pipeline)的前端(Front-End)，包括预取(fetch)和译码(decode)，经过分发(dispatch)和调度(scheduler)后进入执行单元，最后提交执行结果。
> - 所有的指令采用顺序方式(In-Order)通过前端，并采用乱序的方式(Out-of-Order, OOO)进行发射，然后乱序执行，最后用顺序方式提交结果，并将最终结果更新到LSQ(Load-Store Queue)部件
>   - 总的来说：**顺序提交指令，乱序执行(非真正数据依赖和地址依赖的指令)，最后顺序提交结果**；如果有两条没有数据依赖的数据指令，后面那条指令的读数据先被返回，它的结果也不能先写回到最终寄存器，而是必须等到前一条指令完成后才可以
> - LSQ部件是指令流水线的一个执行部件，可以理解为存储子系统的最高层，其上接收来自CPU的存储器指令，其下连接着存储子系统。主要功能是将来自CPU的存储器请求发送到存储器子系统，并处理其下存储器子系统的应答数据和消息

> 对于读指令：
> - 当处理器在等待数据从缓存或者内存返回时，对于乱序执行的处理器，可以执行后面的指令；对于顺序执行的处理器，会使流水线停顿，直到读取的数据返回
> - 如下图所示，在x86微处理器经典架构中，存储指令从L1指令cache中读取指令，L1指令cache会做指令加载、指令预取、指令预解码、以及分支预测。然后进入Fetch & Decode单元，会把指令解码成macro-ops微操作指令，然后由Dispatch部件发到Integer Unit或者FloatPoint Unit.
>   - Integer Unit由Integer Scheduler和Execution Unit组成，Execution Unit包含算术逻辑单元(arithmetic-logic unit, ALU)和地址生成单元(address generation unit, AGU)在ALU计算完成之后进入AGU，计算有效地址完毕后，将结果发送到LSQ部件。
>   - LSQ部件首先根据处理器系统要求的内存一致性(memory consistency)模型确定访问时序，另外LSQ还需要处理存储器指令间的依赖关系，最后LSQ需要准备L1 cache使用的地址，包括有效地址的计算和虚实地址转换，将地址发送到L1 Data Cache中。
>     ![x86mpu](https://github.com/RocketKernel/LinuxKernelGo/blob/master/pic/x86mpu.png)
> - 如下图所示，在ARM Cortex-A9处理器中，存储指令首先通过主存储器或者L2 cache加载到L1指令cache中。在指令预取阶段(instruction prefetch stage)，主要是做指令预取和分支预测，然后指令通过Instruction Queue队列被送到解码器进行指令的解码工作
>     ![CortexA9](https://github.com/RocketKernel/LinuxKernelGo/blob/master/pic/Cortex_A9.png)

### 部分术语
> 超标量体系结构(Superscalar Architecture)： 是描述一种微处理器设计理念，它能够在一个时钟周期执行多个指令

> 乱序执行(Out-of-order Execution)：指CPU采用了允许将多条指令不按程序规定的顺序分开发送给各相应电路单元处理的技术，避免处理器在计算对象不可获取时的等待，从而导致流水线停顿

> 寄存器重命名(Register Rename)：用来避免机器指令或者微操作的不必要的顺序化执行，从而提高处理器的指令级并行能力，它在乱序执行的流水线中有两个作用：
>  - 消除指令间的寄存器读后写相关(Write-after-Read, WAR)和写后写相关(Write-after-Write, WAW)
>  - 当指令执行发生例外或者转移指令猜测错误而取消后面的指令时，可用来保证现场的精确。其思路为当一条指令写一个结果寄存器时不直接写这个结果寄存器，而是先写一个中间寄存器过度，当这条指令提交时再写到结果寄存器中
  
> 分支预测(Branch Predictor)：是处理器在程序分支指令执行前预测其结果的一种机制。在ARM中，使用全局分支预测器，该预测器由转移目标缓冲器(Branch Target Buffer, BTB)，全局历史缓冲器(Global History Buffer, GHB)、MicroBTB，以及Return Stack组成

> 指令译码器(Instruction Decode)：指令由操作码和地址码组成。操作码表示要执行的操作性质，即什么操作；地址码是操作码执行时的操作对象的地址。计算机执行一条指定的指令时，必须首先分析这条指令的操作码是什么，以决定操作的性质和方法，然后才能控制计算机其他各部件协同完成指令表达的功能，这个分析工作由指令译码器完成

> 调度单元(Dispatch)：调度器负责把指令或微操做指令派发到相应的执行单元去执行

> ALU算术逻辑单元：ALU是处理器的执行单元，主要是进行算术运算，逻辑运算和关系运算的部件

> LSQ/LSU部件(Load Store Queue/Unit)：LSQ部件是指令流水线的一个执行部件，其主要功能是将来自CPU的存储器请求发送到存储器子系统，并处理其下存储器子系统的应答数据和消息
