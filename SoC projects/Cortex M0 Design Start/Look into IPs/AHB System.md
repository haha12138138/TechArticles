#AHB #系统介绍 

### Design Start 中 AHB 介绍
Advanced High performance Bus（AHB）是ARM提出的一种简单的具有较高带宽的片上总线。这个系统允许有多个主机，多个从机同时存在。当然随着其他更利于多主多从总线的出现现在比较少会使用到多主机这个功能。于是ARM 便提出了AHB-Lite 的协议，在不存在额外电路的情况下只支持一个主机的存在以简化电路。DesignStart 这个项目中便使用了AHB-Lite 来构建高速总线系统。

AHB系统所需要实现的功能：
1. 数据交换
	- 读写数据
	- 发送错误/等待 等流控制信息
2.  支持多主机 （可选）
### AHB系统构成
先从宏观的角度看看这个系统：
![[AHB system.png]]
可以看到，这个系统与[[APB System]] 提到的结构差不多：
1. 一个 Decoder 和 Mux
2. 一个 Master 

这个Master在这里可以是很多东西：
1. CPU 的 AHB 接口
2. Cache
3. 另一个AHB System
等等......

因为没有一个固定的东西，所以便用Master 来代替了。不过在DesignStart System 中， 这个Master 默认就是CPU。可以看到这里与APB相比少了和中断功耗相关的信号，这是因为这里已经是SoC的top位置了，可以直接将信号连接到CPU 或者PMU中，没有必要再绕一下。

了解了系统的构造之后便来看一下各个组件是如何交互的
%%
#todo
AHB Interface pic

AHB Protocol pic, normal, error
handshaking
special transfer
	purpose of burst
	purpose of HPROT
%%

%%
AHB IP free: GPIO, Memory, bit-banding
paid: Mux, Bridge
%%