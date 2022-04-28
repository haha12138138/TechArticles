### Power Gating
Power Gating 是除了[[Clock Gating]]另外的一种常用的低功耗手段。
Clock Gating，通过减少时钟的翻转率来降低动态功耗。这个方法通过在电源轨到cmos s 端添加了一个做为开关的power gating cell，降低由漏电流引起的静态功耗。为了将这部分功耗尽可能的降低通常会使用较高Vt的晶体管来作为开关。同时又为了能够不限制正常工作时的电流，这些开关需要尽可能的并联。不过无论怎么并联，这个开关总会消耗一定的电压，在设计时需要注意。

Power gate 的开启也有一定讲究，不能一次性把所有的休眠电路都使能，这样有可能会造成远高于正常工作时的峰值电流，可能造成寄存器无法正确工作或复位。

### Retention
%% 
1. concept
2. retention cell structure。 control timing·
%%
由于系统中存在DFF等易失性储存元件，这些元件将需要有"断电保存  上电回复"的功能：在主电源关掉的同时，由副电源供电的HVT的影子寄存器来保存状态。这样的Retention FF 从外部看来通常会比普通的DFF 多出一个Restore/Retention 信号 以及一对电源接口。下面就看两个Retention Cell 的结构以及设计时需要注意的地方。

#### Live Slave
![[Retention_Reg1.png]]
可以看到这是一个Master Slave 结构的DFF。Slave Latch 是处于always-on 区域，而Master 则是在可关断的区域内。从图中可以看到当Retain == 0 时，相当于Clock 是0，Slave Latch 便与master Latch 断开，数据得以保存在Slave Latch 中。相反，Slave Latch 将会在Clock == 1 时保存在Master 中的数据。
⚠️ 对于这个FF 来说 Retain只能在Clock == 0 的情况下改变。
		1. 可以将Retain 信号想成Clock Gating 信号，必须[[Clock Gating|要满足这个要求]] 
		2. 假如在clock高电平的时候power 断电，Slave Latch 就无法保存来自Master的数据。
		3. 假如在clock高电平的时候恢复供电，Slave Latch 就会采样到 未初始化的Master里的值。

由于HVt cell 处在正常工作路径上，这会导致这个DFF的速度比一般DFF 慢。
#### Balloon Register
![[Retention_Reg2.png]]
与 Live Slave 不同的地方在于：Balloon Register 内部储存信息需要一个周期存进Slave Latch 中才可见。 当然这也是一个优势，仔细分析一下之后发现NRETAIN 这个信号无需必须在clock == 0 的时候变化。
%% #todo 时序图%%










