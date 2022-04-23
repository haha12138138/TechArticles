#APB #IP介绍 
%% 
APB 介绍 及系统构成
%%
### APB介绍
APB 系统是一个一主多从的总线系统，主机和从机使用APB协议进行数据交换。APB系统通常不会单独出现，它通常会作为另一个高速总线的子系统而出现。这个系统主打的就是简单与低功耗所以就用来连接低速的外设。

在这里主要介绍一下APB系统所具有的功能：
1.  数据交换
	- 读写数据
	- 发送错误/等待 等流控制信息
1.  发送中断信息
3. 功耗控制

其中读写数据以及能提供等待与错误这两种流控制信号是必须的功能，因为这样才能相对完整的实现APB协议。给系统双方提供一个交流通道。在design start工程中，ARM提供了一个APB Subsystem 的实现（使用了APB3协议），后文将基于这个设计并对它进行讲解与功能升级。
### APB系统构成
先来宏观的了解一下APB系统有哪些组成部分
![[SoC projects/Cortex M0 Design Start/pics/APBsystem.drawio 2.drawio.png]]

从上面的系统图可以看到在这个系统里面所有的slave 共用了 Address Writedata 等等从Adapter主机方产生的信号，从机产生的Response 和 Data则会经过MUX 选择之后才被接受。对于interrupt 和 power control的部分都是和每个从机相连并最后汇入到主机中去的。至于Adapter，通常来说是一个桥：将其他总线协议转化为APB协议。

大致了解了系统的组成部分，接下来就该看他们之间是如何交互的了。
%%
APB 接口与时序
1.  分组 cmd，data，handshaking，interrupt
2. 时序图 正常传输，wait，error response
%%
### APB接口
![[APB Master Ports.jpg]]
⚠️这里的master 包含了Adapter 和 Decoder。 将master的信号方向反转之后便得到了slave 的接口。可以看到这里我们将所有信号划分成了两组：request 和 response。这样分组之后能够很轻易的用握手的方式理解APB 的传输，同时这套模型还能同时应用在AHB 和AXI 的分析上。这里还有需要注意的是PADDR的宽度是APB从机所占据的地址宽度，可以小于系统总线的宽度。

下图中展示的是MUX的接口，MUX通过输进来的sel信号来选择输出。这里的sel就是master interface里的以独热码编码的sel信号。当然直接输入PADDR也是可以的就是面积太大，多了一个decoder。
![[APB MUX Ports.jpg]]
### APB时序
如何将APB总线抽象成带有valid，ready加data的接口呢？ ready 我们已经找到了直接就是PREADY信号，而valid 则直接就是PSEL。我们来看看ARM 给出的example waveform。
#todo %%加入 wavedrom 的 wave%%
![[APB wait trans.png]]
可以看到当传输的valid 置高之后必须要等ready置高才能被清除，期间传输的内容都不能发生变化。这个规则将贯穿APB AHB AXI 3种常用的嵌入式总线协议里。APB总线节能和省面积的原因也就在这里：
1.  no speculation，不进行推测执行。只有等到一整个传输确认完成之后才能启动下一个传输。这样便不需要终止预测执行的传输。
2. 而且独立运行的通道数量最少，即valid-ready pair的数量最少。节省了将不同通道中的数据连接起来的操作。
而不是很多博主说的因为减少了信号翻转次数而节能。

PENABLE 这个信号看起来好像是多余的，有很多博主仅仅把它当成一个区分状态的信号，这个理解就不太对了。PENABLE事实上是一个mask信号，去筛选出属于这次传输的response。 而当PENABLE\==0 时且同时这次是写传输时，PENABLE 则是写使能。

其余的transfer 如 error response则依此类推不再详述。具体可以参考ARM 的 specification。
### 总结
本节就讲述了基本的APB系统的组成以及APB协议的一些讲解。
下一章将实操一个APB register 并由此讲APB的编程模型（programmer model）
[[APB Register]]