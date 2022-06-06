#APB #系统介绍  
%%
Cortex M0 APB：APB Active 信号
clock gating
	principle&reduction: need to syn and get .saif file
	rtlcode_behavioral ICG cell: code and pic
power gating
	principle: leakage, pgcell pic
LPI-Q 
%%
这一章中我们会提到：
1.  Design Start 中APB子系统中的与Power相关的信号
2. Design Start 中APB子系统中的与Clock相关的信号
3. Low Power Interface（LPI）在APB子系统中的使用


### APB系统的Power Control 信号
我们之前提到过，在ARM的design start kit 里使用的是APB4协议，但是单单查看APB4的spec却没有发现有关于power control 的任何信息。这给我们提供了足够的自由度来适配我们自己的设备。在[[APB Bridge]] 这章中， 我们看到在标准的APB信号之外还多有了2个信号：APBACTIVE 和 PCLKEN 这两个信号。其中APBACTIVE 是通知Power Management Unit（PMU），APB是否正在工作以决定是否切换模式，而PCLKEN则是与时钟分频相关的信号。
#### APB 系统的Clock 信号
我们知道APB是用来对接低速设备的，而为了降低这些设备的运行功耗，往往这些设备需要工作在（与总线相比）较低的工作频率。一般来说APB会接到一个固定或者是可编程的时钟分频器上。 在ARM DesignStart 的设计中是通过[[Clock Gating]]来实现这个分频的。这个Gating 的控制信号由一个计数器产生，每到计满时就输出一个脉冲来控制一个clock pulse 通过。而这个PCLKEN就是输出的这个时钟。
![[pclkenwave.png]]

⚠️由于这个时钟不是50%的占空比，对于那些[[Clock Synthesis|半周期路径]]来说会有非常大的限制，很可能无法满足时序。所以在后续设计APB IP的时候尽量让所有的DFF使用同一个时钟沿。当然我们也可以将计数器产生一个50%占空比的信号作为外设的时钟，只是这样做会有较大的skew，而且产生的频率有限制。

### APB 系统中的 LPI 接口
虽然说之前提到的APBACTIVE，PCLKEN确实简单有用，但是它功能简单而且不是很规范不便于复用。Low Power Interface（LPI）就是ARM提出的一套规范的接口用来进行功耗控制。LPI接口包含了Q Channel 和 P Channel，Q Channel包含基本的Request 和 Response，P Channel 则能提供更复杂的Power检测与调整。


#### LPI-Q
Q Channel 包含了4个信号：QREQn，QAcceptn，QActive，QDENY。
![[Q Channel.drawio.png]]
在这个接口里面所有的信号都必须是通过寄存器输出，输入的话都要用同步器打拍。这样设计的原因其实就是为了在不同的时钟域可靠地传播信号。同时这个接口交换数据时使用的是握手协议，更加保证了数据的可靠传递。接下来我们来看看具体的时序图。

⚠️在这个协议中QREQn 代表的意义是：当QREQn == 0时，请求进入低功耗模式。QAcceptn == 0 代表的是已经进入低功耗模式了。

ARM 将整个传输过程分为了不同状态：

| State     | Meaning                             |
| --------- | ----------------------------------- |
| QRUN      | 外设正常工作                        |
| QREQ      | PMU请求外设进入低功耗，外设正常工作 |
| QSTOP     | 外设停止工作                        |
| QEXIT     | PMU请求外设重新工作，外设停止中     |
| QDENY     | 外设收到请求后拒绝，外设工作中      |
| QCONTINUE | PMU接受到DENY响应，外设工作中       |

![[QChannel Wave.png]]
上表列出的这些state 需要在PMU中一一实现来实现协议。对于外设来说就没有必要去实现所有state，只需要实现3个state：Working，Denying，Stopped。外设中的Q Channel 从机通过接收QChannel 传来的信息+ 外设本身的status 来产生Response。

#### APB + Q Channel 实际运用
这个设计分为两部分：
1.  APB mux。接收外部输入的APB传输并连接到 APB slave 中。
2. APB Slave。这里有两个slave 一个是Timer，另一个是很简单的APB Register。

完全理解整个设计会涉及到[[UPF]] [[Clock Gating]] [[Power Gating and Retention]] 的内容，在这里我们直接运行其中的脚本来观察整个系统是如何运作的。

%%
#todo 
1. 整个 APB 关断
2. 发出 启动信号
3. check是否 正常启动
4. 读写 寄存器
5. 设置并启动timer
6. 发出 关断信号
7. Check是否 Deny
8. 等待 timer 停止，关机
9. Check 是否关机
10. 发出启动信号
11. Check 寄存器的值
%%