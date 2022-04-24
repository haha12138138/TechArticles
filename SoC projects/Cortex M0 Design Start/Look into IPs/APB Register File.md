#APB #IP介绍 
### APB 寄存器堆介绍

在上一篇[[APB System]]中提到，APB接在APB系统里的几乎都是些低速外设。 这篇要讲的APB寄存器堆往往都是这些外设的其中一部分。这部分电路承担了配置外设，显示外设状态，进行数据交换等功能。事实上这些寄存器也可以看成是一个Adapter，将APB协议转化为外设控制逻辑的电路。

这一章主要会提到
	1. APB的解码与Response 的产生。
	2. 寄存器类型。
	3. 如何设计集成不同的寄存器到这个寄存器堆中。

### APB解码
[[APB System|上一篇]]中提到的APB总线可以被视为由一个valid-ready对包裹起来的总线。APB解码的内容就是对这个总线的信号进行解析生成向寄存器输出：读写使能以及地址。它同时还要检测这次传输是否合法，以向主机返回error信号。

``` systemverilog
  input PSEL, PREADY,PWRITE;
  logic RegWriteEnable, RegReadEnable;
  assign RegWriteEnable= PSEL&!PREADY&PWRITE;
  assign RegReadEnable= PSEL&PREADY&!PWRITE;
```
注意，对于读操作这里的PREADY最好是PREADY=1的时候才使能，因为对于寄存器来说读一个周期就能完成，没有必要提前一周期将数据输入，而且如果被访问的寄存器是FIFO，提前置ReadEnable 会导致数据读取错误。至于写使能，我们希望在PREADY\==1 的时候就能知道写操作成功与否，自然需要在第一周期将使能置一。

``` systemverilog
input error_from_reg;
assign PSLVERR= PSEL&PREADY&error_from_reg;
```
而错误信号的返回则比较直接了，在第二周期时输出来自寄存器的信号。这里的error_from_reg 可以由一个复杂的逻辑驱动：比如当是写操作的时候可以返回FIFO_full而读操作的时候可以返回FIFO_empty。

### 寄存器类型
寄存器通常是指有触发器组成的元件。但是在这个地方它还有一个更广泛的意义：作为程序可以访问的逻辑上的区域。比如那些记录产品编号的常值"寄存器"，往往不是用触发器堆起来的，而是直接通过TIEHI TIELO 拉到电源线上的线网。

寄存器类型其实就是指的这个寄存器能以什么方式被软/硬件访问。这里面主要有：
+ RW                     可读可写
+ RO/WO              只读/只写
+ RC/W1C/W0C   读清0/写1清0/写0清0
+ WP                     写则输出脉冲
。。。。。

这里要注意的是，一个寄存器往往是软件硬件都能进行访问的，要划分好软件能做什么硬件能做什么。比如一个寄存器在软件看来是只读，在硬件看来时只写。有没有可能一个寄存器在软硬件双方看来都是RW的呢？有可能，但是这样的寄存器往往是buffer，而且也不常用，因为这会限制住带宽。

### 设计APB寄存器堆
具体一个寄存器堆的设计往往与软件上的API函数相关，通常我们会将有联系的寄存器（"字段"）紧密的排在一起，变成一个大的寄存器方便程序进行访问。
+ 这些大寄存器的宽度不应当超过数据总线的宽度。

除此之外APB协议本身还对寄存器的设置有限制。在designstart 给出的设计之中使用了APB3协议，这意味着APB总线所能写的最小宽度就为32bit（默认数据总线宽度）。

+ 这也就意味着这些大寄存器即使宽度不为32bit，也必须占一个Word 的地址空间，也就是APB寄存器堆按word进行对齐。
+ 即使使用了APB4，APB5，最多也只是允许了能按Byte写。对于读操作，依然是一次访问32 bit。

当然，我们本身也不必为了省那一点点空间而故意去凑32bit从而设计出奇怪的寄存器。能使用的地址空间相当的多，在designstart 里面ARM为一个APB子系统预留了64KB的地址，绰绰有余了。

%% #todo ARM Memory Layout%%

### 具体寄存器堆rtl设计
这一章主要展示实现一个寄存器的各组成的代码片段，以供参考
选择器部分：
```systemverilog
logic select；
logic read_data;
assign select= PADDR==`REG1ADDR;
assign read_data=({32{select}}&REG1DATA)|({32{select}}&REG2DATA) ... ;
```

``` systemverilog
//WC
always@(posedge call or negedge rstn) begin
	if(!rstn) begin
		REG1<=0;
	end else if(WriteEnable&select) begin
		`ifdef W1C
			REG1<=(PWDATA==1)? 0:REG1;
		`else //W0C
			REG1<=(PWDATA==1)? REG1:0;
		`endif
	end
end
//WP
REG1=(WriteEnable&selected&(PWDATA==1));
```

最后附上一个 APB IIC IP的github 地址：
https://github.com/thuriyasun/ARM_M0_V1_2022/blob/main/RHBD_IP/apb_i2c/i2c_register.sv
可以集成在designstart 上