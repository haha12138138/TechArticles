### Clock Gating
Clock gating 是一种常用的低功耗方法，这种方法通过将暂时不工作的DFF的时钟置高或置低来达到减小功耗的目的。
⚠️clock gating **不能**直接使用一个与门或者或门来进行gating。因为这个控制信号有可能会和需要控制的clock是异步关系。就即使是同步信号，因为routing，crosstalk 等等原因有可能会出现**延时或者毛刺**，这些不理想因素往往会造成DFF的工作异常：**hold voilation，min-pulse-width violation** 等等。
解决它的问题也很简单，对于posedge DFF，我们只需要在clock 为1 的时候让控制信号稳定，在clock为0时接受外部控制信号的控制。通常来说ASIC library中有一种单元叫Integrated Clock gate（ICG）。
![[ICG.png]]
如果说我们在做行为级仿真的时候没有这样的cell，我们也可以用RTL代码暂时替代。
```systemverilog
input E,CLK;
output CLK_out;
logic E_temp;
always@(*)begin
	if(!CLK) begin
		E_temp=E;
	end
end // This is a latch
assign CLK_out=E_temp&CLK;
```
#### Clock Gating 效果
其实笔者之前并不认为clock gating 能够降低多少功耗，特意做了个小实验来看看clock gating 的效果。这个实验包含3个工况：
1.  正常工作： Clock 使能，D端一直输入数据
2. 空闲未工作： Clock 使能，D端输入常数
3. 待机未工作：Clock 失能，D端输入数据

| 状况 | Internal Power /mW | Leakage Power/mW       | Total Power/mW |
| ---- | ------------------ | ---------------------- | -------------- |
| 1    | 0.4590             | 0.5075$\times 10^{-3}$ | 0.4595         |
| 2    | 0.4333             | 0.5240$\times10^{-3}$  | 0.4338         |
| 3    | 0.0116             | 0.5072$\times 10^{-3}$ | 0.0121         |
惊讶的发现clock gating 竟然减小了近40倍的功耗。当然这与我使用的DFF cell相关，如果采用其他结构的DFF 功耗可能不会有如此大的差距。
![[DFF.png]]