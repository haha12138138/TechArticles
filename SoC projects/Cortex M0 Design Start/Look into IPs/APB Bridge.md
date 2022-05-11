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

%%
structure pic
Interfaces pic
%%
![[APB Bridge diagram.png]]
### APB Bridge 是如何工作的
#### APB Bridge 的状态转换
首先可以看到APB Bridge 中使用了6个状态：

| State name   | Function                                    |
| ------------ | ------------------------------------------- |
| ST_IDLE      | 等待AHB访问APB                              |
| ST_APB_WAIT  | 等待APB的时钟沿                             |
| ST_APB_TRNF  | 产生APB的第一个cycle的信号                  |
| ST_APB_TRNF2 | 产生APB的第二个cycle的信号                  |
| ST_APB_ENDOK | 响应AHB主机，已经读进Buffer中               |
| ST_APB_ERR1  | 响应AHB主机，第一个AHB Error Response cycle |
| ST_APB_ERR2  | 响应AHB主机，第二个AHB Error Resonse cycle  | 

相比于标准的APB的状态转换来说，桥的转换多了3个状态：ST_APB_ERRx 是为了满足AHB对错误的响应要求，ST_APB_WAIT 则是满足APB的低功耗需求。注意到在代码中常常出现的
```systemverilog
if (PCLKEN ....)
```
当PCLKEN == 1时，意味着APB总线会在即将到来的时钟沿做出反应。所以所有与会引起APB信号变化的状态转换都需要这个判断条件。ST_APB_ERR1->ST_APB_ERR2 这个存粹为了满足AHB时序的转换就不需要判断PCLKEN。下图则是状态转换图

%% Bridge State Transition pic%%
![[APB_Bridge.drawio.png]]
#### APB Bridge 如何响应AHB 总线
APB总线与AHB总线有一个很大的不同：APB的地址，控制，数据 3 个信号是同时到的而 AHB的数据在地址和控制信号后一个周期到达。所以有这么一段代码将地址与控制信号存起来。
```systemverilog
logic [31:0]addr_reg;
...
assign apb_select=HSEL & HTRANS[1] & HREADY;
always @(posedge HCLK or negedge HRESETn) begin
	if(!HRESETn) begin
		// Initialize ...
	end else if(apb_select) begin
		addr_reg<=HADDR;
		...
	end
end 
```
同时 apb_select这个信号也是 ST_APB_IDLE -> ST_APB_TRNF1 的触发信号。 当数据到达时（如果没有WDATA buffer的话，同时也会出现在APB总线上），地址与控制信号已经存了起来（出现在APB总线上），状态也从IDLE 变为了 TRNF1 。由于在TRNF1 的时候APB传输还未执行完，所以便拉低HREADYout，暂停AHB的传输。

当APB传输结束的时候，也就是apb_tran_end这个信号拉高的时候，返回的数据将会出现在APB总线上。如果没有RDATA Buffer，这时便可以响应AHB主机，并把HREADY置高。如果有Buffer，则会进入ENDOK这个状态，再响应主机。

Error Response 就很简单粗暴了完全按照[[AHB System | AHB Error Response]] 的步骤来的，这里就不详述了。

#### APB Bridge 中的特殊状态与转换
ST_APB_IDLE的又一个状态变迁挺有意思：
```systemverilog
if (PCLKENWAVE & apb_select & ~(reg_wdata_cfg& HWRITE)) begin
	// jump to TRNF1
end else if (apb_select) begin
	//jump to WAIT
end  ...
```
这里的reg_wdata_cfg 指的是是否有WDATA buffer。这里可以发现，即使PCLKEN等于1， 如果有WDATA Buffer 的话，状态依然是转换到ST_APB_WAIT的。因为如果直接转换到TRNF1的话，WDATA 还未进入Buffer就开始传输了。由于保存WDATA的时候错过了一个PCLK 周期，所以必须等下一次PCLKEN 到来的时候才能进入TRNF。刚好可以利用WAIT这个状态。

另外一个有意思的转换出现在TRNF2结束的时候的时候:
```systemverilog
...
else if (PREADY &(~PSLVERR) &PCLKEN) begin
	if(reg_r_data_cfg) begin
		next_state = ST_APB_ENDOK;
	end else begin
		next_state = (apb_select)? 
						ST_APB_WAIT  // PREADY&(~PSLVERR)&PCLKEN&apb_select
						:ST_APB_IDLE;
	end 
end 
...
```
但是其他状态都是:
```systemverilog
if(PCLKEN & apb_select & ~(reg_wdata_cfg & HWRITE)) begin
	// PREADY&(~PSLVERR)&PCLKEN&apb_select 
	next_state = ST_APB_TRNF;
else begin
	next_state = (apb_select)? ST_APB_WAIT:ST_APB_IDLE;
end 
```
一个转换到了ST_APB_TRNF 另一个却到了WAIT，在没有Buffer 的情况下是这样就很奇怪了。个人觉得是一个纰漏，虽然功能正确可是速度慢了。

#### APB Bridge 的Low Power signal
APB Bridge 中有两个涉及Low Power 的信号：apb_active，PCLKEN。
apb_active 的意义就是只要是：
1. ahb要访问这个bridge
2. bridge 正在工作

就会是高电平。
PCLKEN就是[[Clock Gating]] 的信号。
```systemverilog
assign apb_active= (state!=ST_APB_IDLE)|(HSEL&HTRANS[1]);
```