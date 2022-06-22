#AHB #IP介绍 
### AHB Memory Controller 结构
AHB Memory 是[[AHB System]]中非常重要的一个组成部分。这个部分的设计会影响到整个系统的大小以及运行速度。由于DesignStart这个项目主要的目标是设计小而低功耗的系统，Controller便需要：
+ 适配不同宽度的SRAM（减小floorplan的面积损失）
+ 允许使用单端口的SRAM
+ 适配速度不同的SRAM

ARM提供的Controller：
+ 允许接8 bit/ 16 bit的SRAM
+ 允许使用单端口的SRAM
+ 允许SRAM有不同的READ CYCLETIME，WRITE CYCLETIME，TURN AROUND TIME。以适配不同时序的SRAM

可以从下图中看到整个controller由2部分构成：AHB FSM 和 MEM FSM。AHB FSM 提供了AHB信号的解码功能以及数据总线宽度的转换功能。MEM FSM则将AHB的读写信号，控制信号等转化为Memory的控制信号。同时 MEM FSM 起到计时的功能以保证SRAM的时序。
![[ahbsram_struct.drawio.png]]

### AHB Memory Controller 实现
#### AHB FSM
![[ahbsram_ahbfsm.drawio.png]]
经过简化之后的状态转移如图所示。可以看到有3种类型的变迁：红，绿，黄。这些状态的转换都需要等到DONE信号有效才行。

| Transition | Meaning                                     |
| ---------- | ------------------------------------------- |
| 红         | 需要4次传输：8 bit Memory 接收到32 bit 传输 |
| 黄         | 需要2次传输：8 bit Memory 接收到16 bit 传输 |
| 黄         | 需要2次传输：16 bit Memory 接收32 bit 传输  |
| 绿         | 需要1次传输：传输宽度$\le$Memory宽度       |

每一次的变迁都需要生成新的ADDR，以及Buffer的控制信号。这里我们先讲解ADDR的变化。

| Transition       | Action                                     |
| ---------------- | ------------------------------------------ |
| 红               | 保持Addr\[31:2\],低2位依次为: 00->01->02->03 |
| 黄（8 bit Mem） | 保持Addr\[31:1\],最低一位依次为 0->1        |
| 黄（16 bit Mem） | 保持Addr\[31:2\]与Addr\[0\],第1位依次为 0->1    |
| 绿               | 保持Addr                                   |

要注意一点的是，在这里地址的变化**没有用加法器去实现**是取了一个巧：**Armv6（Cortex M0 的指令集**）是不支持非对齐操作的。也就是说对于红黄两种变迁最开始的第一个地址的低2位一定是0，如果不是0的话CPU内部会产生一个Fault 阻止程序继续运行。所以这里可以直接用状态机硬编码低2位地址。

每一个在图中标注出来的Transition都对应着一个向Memory发出的请求，只有当Done信号为1时这个请求才算是处理完毕了。可以看到IDLE这个状态既对应着**空闲**又对应着**等待最后一个操作**。为了区分这两种情况，我们多加一个信号reg_active。可以看到hreadyout对应的条件其实是```(state==IDLE)&((!reg_active)|DONE)```：只有当传输完成或压根没传输的时候才ready。 之所以这么做而不是多加一个state来表示传输完成，是因为这样做可以省一个时钟周期：Done 为1已经代表可以接收下一个传输了。
```systemverilog
// pseudo code
always@(posedge clk or negedge rstn) begin
	if(!rstn) begin
		reg_active<=0;
		...
	end else if(hready) begin
		reg_active<=trans_valid;
		// working
		...
	end 
end 

assign hreadyout = (!reg_active)|((state==IDLE)&DONE);
              // = (!reg_active)&(state==IDLE)|(state==IDLE)&DONE
              // = (state==IDLE)&((!reg_active)|DONE)

```

#### AHB FSM Data Path
这个部件是这个IP中最重要的一个模块了，这个地方起到了匹配输入输出宽度的作用。
![[ahbsram_wdata.drawio.png]]
对于写数据来说，这个部分无非就是两个MUX：上面的MUX决定着输出的高8位数据的来源，下面的MUX则决定着低8位的来源。如果外接的是16位的Memory，直接将地址线接到A输出就行，如果是8位的，就接到A的低8位B上。

![[ahbsram_rdata.drawio2.png]]
读数据则稍微复杂一些，由于数据位宽不同，需要用寄存器存储之前读到的值，而且还需要根据地址将数据放在对应的输出上。可以根据对应的code来看看是如何选择的。
```systemverilog
i_hrdata = { DATAIN[ 7:0],   DATAIN[7:0],   DATAIN[7:0],   DATAIN[7:0]}; // Byte  read on 8-bit memory
i_hrdata = { DATAIN[ 7:0], read_buffer_2,   DATAIN[7:0], read_buffer_0}; // HWord read on 8-bit memory
i_hrdata = { DATAIN[ 7:0], read_buffer_2, read_buffer_1, read_buffer_0}; // Word  read on 8-bit memory
i_hrdata = { DATAIN[15:8],   DATAIN[7:0],  DATAIN[15:8],   DATAIN[7:0]}; // Byte  read on 16-bit memory
i_hrdata = { DATAIN[15:8],   DATAIN[7:0],  DATAIN[15:8],   DATAIN[7:0]}; // HWord read on 16-bit memory
i_hrdata = { DATAIN[15:8],   DATAIN[7:0], read_buffer_1, read_buffer_0}; // Word  read on 16-bit memory
```

下表展示的是在不同的变迁下，AHB FSM 和 State之间的关系。有助于理解到底什么时候数据到来，以及应该放置的位置。有些传输中存放的reg地址取决于地址
| Transition            | State IDLE      | State 1       | State 2 | State 3 | State 4 |
| ------------------- | --------------- | ------------- | ------- | ------- | ------- |
| green               | reg deactivated | --            | --      | --      | --      |
| red                 | reg deactivated | --            | reg0    | reg1    | reg2    |
| yellow (8 bit Mem)  | reg deactivated | reg2 or reg 0 | --      | --      | --      |
| yellow (16 bit Mem) | reg deactivated | reg0 and reg1     | --      | --      | --      |
#### MEM FSM
![[ahbsram_memfsm.drawio 2.png]]
| Start State | End State | time to next transition | trigger              |
| ----------- | --------- | ----------------------- | -------------------- |
| IDLE        | READ1     | 1                       | RDREQ                |
| READ1       | IDLE      | TURN AROUND TIME        | !RDREQ               |
| IDLE        | WRITE1    | 1                       | WDREQ                |
| WRITE1      | WRITE2    | WRITE CYCLE             | None                 |
| WRITE2      | WRITE3    | 1                       | TIME UP(WRITE CYCLE) |
| WRITE3      | WRITE1    | 1                       | WDREQ                |
| WRITE3      | IDLE      | TURN AROUND TIME        | !WDREQ               |

Mem FSM相对来说就简单很多了，它的主要工作就是去模拟SRAM内部的延时。并通过DONE信号来返回操作是否完成。time to next transition 这个列代表的意思就是经过这个transition之后会在End State 停留多久。 比如第二行代表的意思就是 write1 到 write2 之后 在write2会等待WRITE CYCLE 这么长时间之后才能再次跳转（SRAM 已经成功写入）。