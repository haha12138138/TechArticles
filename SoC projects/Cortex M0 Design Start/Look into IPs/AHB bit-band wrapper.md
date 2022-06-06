%%
1.  bit band介绍
2. bit band wrapper 结构
3. 实现
	1. 地址变换
	2.  bit band数据变换
	3.  bitband wrapper 状态机
%%
#ahb #IP介绍 
### Bit-Band 介绍
bit-band 是一种特殊的储存器模型。这个模型允许将内存访问的粒度从1 Byte减少到1 Bit。拥有这个储存器模型的系统能只用一条指令就能单比特修改内存值。实现这个效果的原理很简单：

> 将支持这个操作的内存A映射到另一块内存B上，使得A中的每一个bit 对应着B中到一个word

在我们的DesignStart系统中，ARM规划出了两块区域
| Aliased Region          | Original Region         | Name              |
| ----------------------- | ----------------------- | ----------------- |
| 0x22000000 - 0x23fffffc | 0x20000000 - 0x200fffff | SRAM Region       |
| 0x42000000 - 0x43fffffc | 0x40000000 - 0x400fffff | Peripheral Region |
![[bit-band memory.png]]
当我们想使用bit-band 功能时，我们直接访问的是Aliased Region。一个特殊部件——Bit-Band Wrapper，将位于Aliased Region里面的地址变换到Original Region里面。
当我们要读Aliased 内部的数据的时候，这个Wrapper将AHB 的HADDR 转化之后直接输出。但是当我们想要写数据时，这个Wrapper 将会做Read-Modify-Write 操作
> RMW 操作
> 先读对应 Original Region里的数据，
> 根据写入的数据与地址修改某位
> 写回 Original Region

### Bit-Band Wrapper的实现
#### 地址变换
我们之前知道了地址的变换有这个规律：使得A中的每一个bit 对应着B中到一个word。那么A中的一个Byte就会占用8Word（32 Byte）。也就是说bit 5以上的数据才会被用于变换。另外当我们观察地址的高位部分可以发现：31-25 位是个固定的值：0x200 或 0x400。依照刚才的观察，我们可以将地址分为3个部分：
1. 31-25: 需根据地址判断到底是在SRAM Region 还是 Peripheral Region 并修改其值
2. 24-5：直接使用
3. 4-0：不用于地址变换。其中4-2 会用于修改数据。

所以使用如下代码进行变换：
```systemverilog
haddr_mux[31:20] = reg_bit_address[20] ? 12'h400 : 12'h200;
haddr_mux[19: 2] = reg_bit_address[19:2];
//reg_bitband_size用于确保访问时对齐的
haddr_mux[    1] = reg_bit_address[1] & (~reg_bitband_size[1]);
haddr_mux[    0] = reg_bit_address[0] & (reg_bitband_size[1:0]==2'b00);
```

#### 数据变换
##### 大小端变换
这个部分主要是将如何修改bit数据以及大小端数据的相互转换。主要涉及到的信号有两个：
1. bit_number_dp：确定word中哪个bit会被修改
2. byte_number_dp ：有效的数据在哪个byte里

先不考虑大小端的问题，假设全部是小端
bit_number_dp 很简单就是:
```systemverilog
assign bit_number_dp= reg_version_of_HADDR[6:2];
```
而byte_number_dp就是：
```systemverilog
always@(posedge clk or negedge rstn) begin
	if(!rstn) begin
		byte_number_dp <=0;
	end else begin
		case(HSIZE)
		0:
			byte_number_dp<=HADDR[1:0];
		1:
			byte_number_dp<={HADDR[1],1'b0};
		2:
			byte_number_dp<=0;
		default:
			byte_number_dp<=0;
		endcase
	end
end
```

要改变数据的大小端格式首先要知道总线的第x根线承载了第y个bit。
```systemverilog
小端数据：
x=y

大端数据:
x=y[4:0];//byte use lower 3 bit
x={y[4],~y[3],y[2:0]};// half word use lower 4 bit
x={~y[4:3],y[2:0]}; // word
```
也就是说大于1 Byte 的数据 byte之间会交叉互换。
最后的代码如下
```systemverilog
always @(BIGENDIAN or HSIZES or HADDRS)
  begin
    case (HSIZES[1:0])
      2'b00 : // bytes
        byte_number_ap = HADDRS[1:0];
      2'b01 : // half word
        byte_number_ap = {HADDRS[1], BIGENDIAN};
      2'b10 : // word
        byte_number_ap = {BIGENDIAN, BIGENDIAN};
      2'b11 : // double word - Invalid
        byte_number_ap = {BIGENDIAN, BIGENDIAN};
      default :
        byte_number_ap = 2'bxx;
    endcase
  end

  // Registering for data phase
  always @(posedge HCLK or negedge HRESETn)
    begin
    if (~HRESETn)
      byte_number_dp <= 2'b00;
    else if (HREADYS)
      byte_number_dp <= byte_number_ap;
    end


  always @(BIGENDIAN or HSIZES or bit_address_source)
  begin
    case (HSIZES[1:0])
      2'b00 : // bytes
        bit_number_ap = bit_address_source[4:0];
      2'b01 : // half word
        bit_number_ap = {bit_address_source[4], BIGENDIAN ^ bit_address_source[3], bit_address_source[2:0]};
      2'b10 : // word
        bit_number_ap = {BIGENDIAN ^ bit_address_source[4], BIGENDIAN ^ bit_address_source[3], bit_address_source[2:0]};
      2'b11 : // double word - Invalid
        bit_number_ap = {BIGENDIAN ^ bit_address_source[4], BIGENDIAN ^ bit_address_source[3], bit_address_source[2:0]};
      default :
        bit_number_ap = 5'bxxxxx;
    endcase
  end
  // Registering for data phase
  always @(posedge HCLK or negedge HRESETn)
    begin
    if (~HRESETn)
      bit_number_dp <= 5'b00000;
    else if (HREADYS)
      bit_number_dp <= bit_number_ap;
    end
```


##### 数据修改
读数据的话比较清晰，当正在执行的**不是**bit-band 的读取时，直接返回Slave送回的数据就好。 如果**是**的话，首先根据bit_number_dp当前读返回的bit。在根据byte_number_dp 将读回来的数据放在对应的总线（bus line）上。 
```systemverilog
assign  read_bit_value = HRDATAM[bit_number_dp];
  // Bit-band alias read value
  assign  HRDATAS[ 7: 0] = (bitband_read_dataphase) ?
             ((byte_number_dp==2'b00) ? {7'b0000000,read_bit_value} :  8'h00) :            HRDATAM[7:0];
  assign  HRDATAS[15: 8] = (bitband_read_dataphase) ?
             ((byte_number_dp==2'b01) ? {7'b0000000,read_bit_value} :  8'h00) :            HRDATAM[15:8];
  assign  HRDATAS[23:16] = (bitband_read_dataphase) ?
             ((byte_number_dp==2'b10) ? {7'b0000000,read_bit_value} :  8'h00) :            HRDATAM[23:16];
  assign  HRDATAS[31:24] = (bitband_read_dataphase) ?
             ((byte_number_dp==2'b11) ? {7'b0000000,read_bit_value} :  8'h00) :            HRDATAM[31:24];
```

还有另外一种实现方式也应该是可行的。因为CPU的内部机构能正确取出正确位置上的数据，我们将数据填充满总线也是可以的。
```systemverilog
/*
Word: 0 0 0 1/0
Short: 0 0/1 | 0 0/1
Byte: | 0/1 | 0/1 | 0/1 | 0/1 |
*/
HRDATAS = (bitband_read_dataphase)?
			  (BigEndian)? // fixed after compiling
			   {
			       {7'b0000000,read_bit_value},
			       {7'b0000000,read_bit_value&(!reg_HSIZE[1]&!reg_HSIZE[0])},
			       {7'b0000000,read_bit_value&(!reg_HSIZE[1])}，
			       {7'b0000000,read_bit_value&(!reg_HSIZE[1]&!reg_HSIZE[0])}
			   }
			  :{
				   {7'b0000000,read_bit_value&(!reg_HSIZE[1]&!reg_HSIZE[0])},
				   {7'b0000000,read_bit_value&(!reg_HSIZE[1])}，
				   {7'b0000000,read_bit_value&(!reg_HSIZE[1]&!reg_HSIZE[0])},
				   {7'b0000000,read_bit_value}
			   }
		  : HRDATAM;

		   
```

写操作的步骤很类似：通过bit_number_dp找到要修改的bit位置，byte_number_dp找到要写的是1 还是0 。
```systemverilog
assign  write_bit_value = ((byte_number_dp==2'b00) & HWDATAS[0]) |
                            ((byte_number_dp==2'b01) & HWDATAS[8]) |
                            ((byte_number_dp==2'b10) & HWDATAS[16]) |
                            ((byte_number_dp==2'b11) & HWDATAS[24]);
/*另外一种实现方式,对ARM M0有效
 assign write_bit_value = HWDATA[0]；
 M0的输出也会进行数据的padding。如在传输byte的时候32bit的总线上会重复出现4个byte值
 */
  assign bit_mask = 1 << bit_number_dp;

  // Create write data
  assign   nxt_data_buffer = (HRDATAM & (~bit_mask)) | (bit_mask & {32{write_bit_value}});

  // Registering for data phase - update at the end of the bit-band read data phase
  always @(posedge HCLK or negedge HRESETn)
    begin
    if (~HRESETn)
      reg_data_buffer <= {32{1'b0}};
    else if (bitband_read_dataphase & HREADYOUTM)
      reg_data_buffer <= nxt_data_buffer;
    end

  // Select HWDATAM output
  // Use data in buffer in state two (bit-band write)
  // and in state 3 (error state) if the error was taken place
  // during write operation (~bitband_read_dataphase).
  assign HWDATAM = ((reg_fsm_state==RMW_TWO)|
                    ((reg_fsm_state==RMW_ERR) & (~bitband_read_dataphase))) ?
                    reg_data_buffer : HWDATAS;
```