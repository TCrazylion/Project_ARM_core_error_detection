# week7_summary

### 1.BIST ATE

- ATE

  Automatic test equipment. 是用来给测试芯片提供测试模式，分析芯片对测试模式
  的响应来检测芯片是好还是坏的测试系统。测试系统有一个或多个测试头（test headers), 包含了到被测芯片的电路。

  加激励（测试向量），测量芯片响应输出(response), 与事先预测的结果比较，若符合，芯片是好的。

- BIST

  在设计时在电路中植入相关功能电路用于提供自我测试功能的技术

  - MemBIST

    芯片内部有一个BISTController，用于产生存储器测试的各种模式和预期的结果，并比较存储器的读出结果和预期结果。

  - LogicBIST

    采用多输入寄存器（MISR）作为获得输出信号产生器。

    LogicBIST方法是利用内部的向量产生器逐个地产生测试向量，将它们施加到被测电路上，然后经过数字压缩和鉴别产生一个鉴别码，将这个鉴别码同预期值进行比较。

    缺点：只能检测是否出错，但是不能定位出错的位置点。

- ATE vs BIST

   BIST检测速度快，成本比ATE低，测试覆盖率大

### Intel 手册

处理器内置自测硬件在开机时可能执行执行BIST。

如果处理器通过BIST，则EAX 寄存器 (0H)被清除。BIST之后EAX 寄存器中的非零值表示检测到处理器故障。如果没有请求BIST，则在硬件重置之后EAX寄存器的内容为0H。



## Future Plan

lock-step 和 BIST的底层原理和逻辑先理顺清楚。