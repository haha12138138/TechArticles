#ic_basic #low_power 

这章主要会提到：
1. Power Gating 的目的与方法
2. Retention 的意义与Retention Cell的结构。
3. Retention Cell 的时序

### Power Gating
Power Gating 是除了[[Clock Gating]]另外的一种常用的低功耗手段。
Clock Gating，通过减少时钟的翻转率来降低动态功耗。这个方法通过在电源轨到cmos s 端添加了一个做为开关的power gating cell，降低由漏电流引起的静态功耗。为了将这部分功耗尽可能的降低通常会使用较高Vt的晶体管来作为开关。同时又为了能够不限制正常工作时的电流，这些开关需要尽可能的并联。不过无论怎么并联，这个开关总会消耗一定的电压，在设计时需要注意。

Power gate 的开启也有一定讲究，不能一次性把所有的休眠电路都使能，这样有可能会造成远高于正常工作时的峰值电流，可能造成寄存器无法正确工作或复位。

### Retention 寄存器
%% 
1. concept
2. retention cell structure。 control timing·
%%
由于系统中存在DFF等易失性储存元件，这些元件将需要有"断电保存  上电回复"的功能：在主电源关掉的同时，由副电源供电的HVT的影子寄存器来保存状态。这样的Retention FF 从外部看来通常会比普通的DFF 多出一个Restore/Retention 信号 以及一对电源接口。下面就看两个Retention Cell 的结构以及设计时需要注意的地方。

#### Live Slave 结构
![[Retention_Reg1.png]]
可以看到这是一个Master Slave 结构的DFF。Slave Latch 是处于always-on 区域，而Master 则是在可关断的区域内。从图中可以看到当Retain == 0 时，相当于Clock 是0，Slave Latch 便与master Latch 断开，数据得以保存在Slave Latch 中。相反，Slave Latch 将会在Clock == 1 时保存在Master 中的数据。
⚠️ 对于这个FF 来说 Retain只能在Clock == 0 的情况下改变。
		1. 可以将Retain 信号想成Clock Gating 信号，必须[[Clock Gating|要满足这个要求]] 
		2. 假如在clock高电平的时候power 断电，Slave Latch 就无法保存来自Master的数据。
		3. 假如在clock高电平的时候恢复供电，Slave Latch 就会采样到 未初始化的Master里的值。
由于HVt cell 处在正常工作路径上，这会导致这个DFF的速度比一般DFF 慢。

#### Balloon Register 结构
![[Retention_Reg2.png]]
与 Live Slave 不同的地方在于：Balloon Register 内部储存的信息在CK为高的时候才能从Q观察到。当然这也是一个优势，将高Vt的部件移除了关键路径，提高了运行的速度。

### Retention Register 仿真
这一部分会涉及到：
1. Retention Register 的hdl 描述
2. 驱动 Retention Register 的时序

这种涉及到Power 的行为级仿真会比普通的行为级仿真更加复杂。除了hdl 文件之外还需要有[[UPF]] 文件来描述电源的连接以及通断规则，library 文件来描述引脚的功能。即使是hdl文件，我们也需要添加语句来保证仿真的正确性。

#### Retention Register 的hdl 描述
通过观察这个结构图可以发现这里面有一些线是有多个driver 驱动的，这种情况虽然可以用门级描述，但是这样不直观而且比较容易出错。为了能使用行为级描述，我们便不能直接按照电路图来。

##### Live Slave 的hdl 描述
之前提到过，Live Slave 结构的 Retention Register 和 Master - Slave Latch是一模一样的。所以我们直接用always块搭出Latch然后拼成 M-S 触发器就好了。这些Latch的Clock Pin 都由NRET 这个信号来控制。  
 ```systemverilog
module DFF (
input D,
input CLK,
input NRET, // save when pulled to 0
output Q
);
	logic clk_t;
	logic QMST,QSLV;
	// Use AND Gate as a clock gate
	assign clk_t=NRET&CLK;
	// Describe Latch
	 always @(*) begin
		if (!clk_t) begin
			 QMST=D;
		end
	end 
	always @(*) begin
		if (clk_t) begin
			 QSLV=QMST;
		end
	end
	assign Q=QSLV;

endmodule
```
为了使我们的module能够仿真与power 相关的内容：指定电源脚，断电输出x 等功能还需要添加一些语句。
```systemverilog
...
output Q,
(* pg_type= "primary_power" *) input VDD,
(* pg_type= "primary_gound" *) input VSS,
(* pg_type= "backup_power"  *) input TVDD,
(* pg_type= "backup_gound"  *) input TVSS  // ok to ignore if ground is shared
);
	...
	always@(*) begin
		if ((VDD==0)||(VSS!=0))begin
			QMST=1'bx;
		end 
		...
	end 
	always@(*) begin
		if ((TVDD==0)||(TVSS!=0))begin
			QSLV=1'bx;
		end 
		...
	end
	
```

##### Balloon 的hdl 描述
这个Register 的结构会稍微偏离M-S 触发器的结构，因为Slave Latch 的数据可能来自两个地方：
1. Master Latch
2. Shadow Slave Latch

我们使用一个Mux 来描述这种选择，这样便能够避免多driver驱动的问题。下面直接上代码：
```systemverilog 
module DFF_Balloon (
input clk, // Clock
input NRET,
input D,
output Q,
(* pg_type = "primary_power" *) input VDD,
(* pg_type = "primary_ground" *) input VSS,
(* pg_type = "backup_power" *) input TVDD
);
	reg QMST,QSLV,QRET;
	wire clk_t,QSLV_mux;
	assign clk_t=clk;
	always @(*) begin
		if((VDD ==0) || (VSS!=0) ) begin
			QMST=1'bx;
		end else if(!clk_t) begin
			QMST=D;
		end 
	end
	always @(*) begin
		if((VDD ==0)||(VSS!=0)) begin
			QSLV=1'bx;
		end else if(clk_t) begin
			QSLV=QSLV_mux;
		end 
	end
	always @(*) begin
		if((TVDD ==0)||(VSS!=0)) begin
			QRET=1'bx;
		end else if(NRET) begin
			QRET=Q;
		end 
	end
	assign QSLV_mux=(NRET)?QMST:QRET;
	assign Q=QSLV;
endmodule : DFF_Balloon
```

#### Retention Register 的仿真结果
Live Slave 的：
![[slive_slave.png]]
Balloon 的：
![[balloon.png]]

这两个仿真用了相同的驱动波形。在准备断电之前都是先在时钟下沿的时候NRET -> 0，再在下一个时钟上沿断电。restore的情况则是相反：先上电在下一个时钟沿NRET -> 1 。
可以看到Live Slave结构的寄存器在主电源断掉的情况下一直可以保持存储的值，而balloon 结构则是无法保持输出的值。balloon结构只能在上电之后，在第一个时钟高电平的时候恢复输出。对于两个电源都断掉的情况，他们都无法恢复值。