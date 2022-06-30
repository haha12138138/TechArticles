#APB #IP介绍 

这一章主要包括：
1. APB Bridge 的构成与接口
2. APB Bridge 是如何工作的

### APB Bridge的构成与接口
在Deisgn Start 中，APB Bridge是一个非常重要的部件。它连接了挂在高速总线AHB上的设备和挂在APB上的外设，也就是说它是一个[[APB System|Adapter]]。它上面的接口也很明显了，需要包括所有的AHB 和 APB 信号，当然可以选择是否添加中断和功耗控制引脚。
从架构图中可以看到，整个Bridge 被分成了3个部分:
1. AHB Interface，指的是将AHB信号解码成一系列标志
2. Bridge FSM， 这里将产生AHB 中的HREADYout，HRESP等信号以模拟一个AHB 从机，同时也会产生 APB的各种信号：充当APB总线的主机。
3. Buffer，这是一个可选的部件。主要目的是通过寄存器隔断较长的时序路径：从AHB主机一直到APB从机的跨越两个Bus的路径。
> 在ARM 提供的默认版本中，读回来的数据是会经过Buffer的，而写入的数据是默认不带Buffer的。当然如果说对系统的时钟频率很低，建议将Buffer去掉以节约一个周期的时间。

![[APB Bridge diagram.png]]
%%
structure pic
Interfaces pic
%%
 ![[APB Bridge fsm2.png]]
### APB Bridge 是如何工作的
#### APB Bridge 的状态转换
首先可以看到APB Bridge 中使用了6个状态：

| State name   | meaning                                    |
| ------------ | ------------------------------------------- |
| ST_IDLE      | 等待AHB访问APB                              |
| ST_APB_WAIT  | 等待APB的时钟沿                             |
| ST_APB_TRNF  | 产生APB的第一个cycle的信号                  |
| ST_APB_TRNF2 | 产生APB的第二个cycle的信号                  |
| ST_APB_ENDOK | 响应AHB主机，已经读进Buffer中               |
| ST_APB_ERR1  | 响应AHB主机，第一个AHB Error Response cycle |
| ST_APB_ERR2  | 响应AHB主机，第二个AHB Error Resonse cycle  | 


%% Bridge State Transition pic%%

#### APB Bridge 工作流程
我们先来看看在正常情况下AHB 时序是如何与APB时序对应起来的。
![[APB Bridge main path.png]] 
在IDLE的时候AHB总线发送了地址和控制信号。由于AHB的地址，控制信号与数据不是同时到达的，我们就需要一个Buffer信息保存起来。
```systemverilog
logic [31:0]addr_reg;
...
assign apb_select=HSEL & HTRANS[1] & HREADY;
always @(posedge HCLK or negedge HRESETn) begin
	if(!HRESETn) begin
		// Initialize ...
	end else if(apb_select) begin
		addr_reg<=HADDR;
		write_reg<= HWRITE;
		// store AHB control signals to be decoded
		// into APB control signals 
		...
	end
end 
```
**在APB Clk 到来的时候**，状态进入到TRNF1 。这个时候AHB传输的所有信号都到齐了便可以进行第一个周期的APB传输了。在下一个APB clk到来的时候Bridge就产生对应APB第二个周期的信号，并同时接收数据。正常来说到这里APB传输就结束了，如果说我们希望缩短RDATA的时序路径长度，便可以在该路径上加入Buffer。由于这个Buffer的存在，使得数据晚了1个AHB clk cycle，我们便需要一个额外的state：ENDOK来提示AHB主机数据到了。

接下来我们要考虑的是：**一个传输发送完成之后如何接受下一个传输**。在这里可以看到有两种转移方式。ENDOK 和 ERR2这两个state 都同时可能直接转移到TRNF中去，而TRNF2则不行。这个我个人认为是一个纰漏。虽然功能是正确的，但是从TRNF2 到WAIT 再到TRNF1 就整整浪费掉了1个PCLK CYCLE。**这两类state的最大区别是TRNF2必须要等到PCLKEN=1才能进行状态转移**，这是由于必须要保证APB信号只能在PCLK来临的时候才能变化。ENDOK 与 ERR2 本身与APB的控制信号无关。
![[APB Bridge return path2.png]] 

WAIT 这个状态很有意思，他有两个功能：
1. 等待PCLK到来并转移到TRNF1中。
2. 保证WDATA buffer的时序。

第一个功能其实就是保证APB总线的时序：只有PCLK来临的时候APB信号才能变化。第二个功能其实是ARM 做的hack了。如果我们想要切分WDATA路径以提高频率，我们就需要一个WDATA Buffer 来暂存这个数据。这样一来数据和地址等控制信号就相差2个AHB周期了，这时候我们便可以利用WAIT来做一个延时。
![[APB Bridge wait.png]]
Error Response 就很简单粗暴了完全按照[[AHB System | AHB Error Response]] 的步骤来的，这里就不详述了。

#### APB Bridge 的Low Power signal
APB Bridge 中有两个涉及Low Power 的信号：apb_active，PCLKEN。
apb_active 的意义就是只要是：
1. ahb要访问这个bridge
2. bridge 正在工作

就会是高电平。
PCLKEN就是[[Clock Gating]] 的信号，用来对PCLK进行分频。
```systemverilog
assign apb_active= (state!=ST_APB_IDLE)|(HSEL&HTRANS[1]);
```