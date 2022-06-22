#AHB #系统介绍 

### Design Start 中 AHB System介绍
Advanced High performance Bus（AHB）是ARM提出的一种简单的具有较高带宽的片上总线。这个系统允许有多个主机，多个从机同时存在。当然随着其他更利于多主多从总线的出现现在比较少会使用到多主机这个功能。于是ARM 便提出了AHB-Lite 的协议，在不存在额外电路的情况下默认只支持一个主机的存在以简化电路。DesignStart 这个项目中便使用了AHB-Lite 来构建高速总线系统。

AHB Lite系统所需要实现的功能：
1. 数据交换
	- 读写数据
	- 发送错误/等待 等流控制信息
2.  支持多主机 （可选）

### AHB系统构成
先从宏观的角度看看这个系统：
![[AHB System.png]]
可以看到，这个系统与[[APB System]] 提到的结构差不多：
1. 一个 Decoder 和 Mux
2. 一个 Master 

这个Master在这里可以是很多东西：
1. CPU 的 AHB 接口
2. Cache
3. 另一个AHB System
等等......

在DesignStart System 中， 这个Master 默认就是CPU。可以看到这里与APB相比少了和中断功耗相关的信号，这个是由于
1. 我们的系统中的AHB部分是始终处在工作状态下
2. AHB 未提供除APB中断外的其他中断

除此之外，AHB系统对Slave部分有一个要求：

> **必须存在至少一个Slave**。如果当Master 访问没有Slave 占据的位置时，这个Default Slave 就会被选中，并提供必要的response 保证系统能够继续运行。

了解了系统的构造之后便来看一下各个组件是如何交互的。

### AHB 接口

上左图给出的是AHB Master的接口。**所有黄色的信号都是 *输出*，红色的都是 *输入*** 。
上右图给出的则是Slave 的接口。**所有黄色的信号都是*输入*，红色的都是*输出***。这里注意一点Slave 的HSEL 比Master HSEL的信号宽许多倍，因为这个信号起到选择Slave 的作用。
依照我们之前分析APB的手段，现在我们要找出哪些信号作为的是Request，哪些作为的是Response。 

### AHB 时序
在看具体的传输时序之前先来看看几个比较重要的信号：

| Name      | Functions                    | values                  |
| --------- | ---------------------------- | ----------------------- |
| HTRANS    | 给出现在传输的状态           | IDLE，BUSY，NONSEQ，SEQ |
| HREADY    | 现在HTRANS指示的传输能否进行 | No（0）Yes（1）         | 
| HSEL      | 选择Slave                    | ---                     |
| HREADYout | Slave 能否接收传输           | BUSY（0）Free（1）      |
| HRESP     | 总线传输是否有误             | Error（1）OK（0）       |

一个AHB从机判断是否有主机向它发送信息是通过：HTRANS，HSEL 和 HREADY 来判断的。
```systemverilog
logic valid_transfer;
assign valid_transfer= HTRANS[1]&HSEL&HREADY;
// parameter IDLE_TRANS=0;
// parameter BUSY_TRANS=1;
// NONSEQ_TRANS=2;
// SEQ_TRANS= 3;
// this equals to ((HTRANS!= IDLE_TRANS)|(HTRANS!= BUSY_TRANS))&HSEL&HREADY;
```
> 这个valid_transfer 就是我们需要的Request。至于Response 那就很简单了 就是HREADYout。
   在我们的DesignStart 项目中，HREADYout 默认时和HREADY直接用线连起来的。

#### Single Transfer
AHB 之所以性能比APB更高的原因在于它的pipeline access 的传输方式。控制信号+地址 在传输的第一个周期传给Slave，该 transfer对应的数据则最快在第二个周期取回。在传输数据的时候，下一个transfer 的控制信号和地址则同时传输。这样传输使得在理想情况下主机可以每个周期进行transfer。

![[AHB Single Trans.png]]
从图中可以看到：
 - Address Request 和HTRANS，（HSEL认为选中）直接挂钩
 - Address Response 和HREADY（HREADYout）一样
 - Data Request 比 Address Requst & Address Response 晚一周期
 - Data Response 只有在 有Request 情况下和HREADY（HREADYout）一样。

> 总结一下：**AHB 存在两个握手的通道，这两个通道有一个固定为1 Cycle 的时序关系**。
#### Error Response
![[AHB Error Trans.png]]
AHB 的Error Response至少需要2个Cycle 才能完成。图中的transfer States 展现的是当前总线的状态。可以看到error 响应占了2个周期：
+ 第一个周期从机向主机发回错误标志。
+ 在主机采样之后，第二个周期主机取消之前准备发送的传输。
+ 如果从机需要多个周期$(\ge 2 主机周期)$ 来从之前传输的错误中恢复，则在HRESP =0 后置HREADYout =0 直到准备好接收下个传输为止。

#### Burst Transfer
支持Burst Transfer 是 AHB与APB之间最大的不同。这种传输会告知Slave时候需要连续接收/提供数据，这时候Slave可以根据这个信息进行预取操作来掩盖延时。
Burst transfer 分 3种：
+ stream
	+ INCR ： 不定长的连续传输，通常在实现的时候是用多个定长传输拼接而成。
+ fixed size
	+ INCRx：定长的传输，每次传输的地址递增。
	+ WRAPx： 定长的传输，传输的地址有一个范围限制，超出最大的地址后会回到最小的地址。

#### Lock Transfer
Lock Transfer 涉及到HMASTLOCK 这个信号。它的目的是，在有多个主机的存在时，使得某个主机的传输不被其他主机打断。这个操作常常用来让多个处理器共享资源。当然这在Design Start中这个问题不存在，因为只有一个主机。

%%
#todo
~~HB Interface pic~~
~~AHB Protocol pic, normal, error~~
~~handshaking~~
special transfer
	~~purpose of burst~~
	~~purpose of HPROT~~
%%

%%
AHB IP free: GPIO, Memory, bit-banding
paid: Mux, Bridge
%%