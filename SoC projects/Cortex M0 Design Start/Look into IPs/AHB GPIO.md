%%
ahb gpio structure
ahb gpio function
ahb gpio sw interface
%%
#ahb #IP介绍 
这一章主要包括：
1. AHB GPIO 系统 的组成与结构
2. AHB GPIO 系统 的软件接口与功能
3. AHB GPIO 系统 具体的工作原理

### AHB GPIO IP 的组成与结构
GPIO 是一个SoC 的重要组成部分，这些接口承担着与片外系统的信息交互的功能。有别于很多外设（IIC，UART ...），GPIO很有可能会涉及到高速的信号传输，所以在DesignStart 这个系统中，这个GPIO IP 被放在AHB bus上。虽然 DesignStart 使用的是32位的CPU，但是一个这样的GPIO只有16位。 

这个IP 涉及到三个部分：
+ AHB 协议的转换
+ IO 寄存器
+ 寄存器与Pad 的连接n

⚠️ 这个IP并不包含具体的IO 引脚，这些引脚可以根据用户需要自行添加第三方库
![[ahb_gpio.drawio.png]]

这个IP 分的稍微散一些，可以看到这涉及到了至少4个文件。其中pin_mux是允许修改的，其他部分则最好不要更改。

### AHB GPIO IP 的软件接口与功能
#### 读写数据
这个GPIO的接口本质上就是一个Register File。CPU首先设置好每个port对应的方向以及功能，然后直接读写寄存器就好。

⚠️但是，如果仅仅将GPIO看成一个16位的寄存器来读写的话会有相当大的局限。这个局限性来源于对特定bit的控制。由于ARM CPU最小能访问的数据大小是8bit（1 byte），要想修改一个bit只能先读当前寄存器的值，经过CPU运算之后将新值写回寄存器中（Read-Modify-Write）。这便要求GPIO至少得将GPIO读出的数据与写入的数据分到两个不同的寄存器去储存。

同时由于这个Read-Modify-Write（RMW）操作涉及到CPU，这便使得操作一个bit会涉及到很多额外的步骤，使得效率变得很低。这里有两个解决方法：
1. 在总线上添加额外的部件以实现 [[AHB bit-band wrapper|Bit-Band]] 功能
2. 在IP内部一周期直接完成这个RMW操作
这两个方案的原理较为相似：将寄存器的bitX映射到另外一块大小位一个Word的内存空间Y中。CPU 对 Y的操作便会由硬件转化为对X 的RMW操作，从而将CPU移出这一系列操作，来达到提升速度的效果。

在C库中，这个映射被抽象成了两个int型的数组。访问这个数组时所使用的index会转化为对对应bit的使能，从而达到单bit操控的能力。
```C
typedef struct{
...
volatile int lower_mask_region[256];
volatile int high_mask_region[256];
...
}GPIO_RegType;
```

#### 配置寄存器 
除了直接访问管脚以外，这个GPIO系统还需要有能力对管脚的方向，管脚功能和中断功能进行配置。与操纵管脚输出不同，这些功能的配置需要借助两个寄存器：xxxSET，xxxCLR。 往xxxSET写就是将某一位置1，而往xxxCLR写就是将某一位置0 。虽然说会稍微降低访问效率，但是由于这些配置往往不需要快速响应，降低面积就很重要了。

GPIO能配置的功能有：
1. 方向：OUTENSET/OUTENCLR
2. 是否使用附加功能（由其他外设提供）：ALTFUNCSET/ALTFUNCCLR
3. 中断
	1. 使能： INTENSET/INTENCLR
	2. 中断触发类型（电平触发，沿触发）：INTTYPESET/INTTYPECLR
	3. 中断触发电平（低或负沿，高或正沿）：INTPOLSET/INTPOLCLR
	4. 中断标志：INTCLEAR，写入清除中断

下面给出完整的寄存器layout
| Address     | Type | Name                     |
| ----------- | ---- | ------------------------ |
| 0x0         | RW   | DATA                     |
| 0x4         | RW   | DATA Out                 |
| 0x10        | RW   |    |
| 0x14        | RW   |  |
| 0x18        | RW   | Alternate function set                         |
| 0x1c        | RW   | Alternate function clear                         |
| 0x20        | RW   | Interrupt enable set     |
| 0x24        | RW   | Interrupt enable clear   |
| 0x28        | RW   | Interrupt type set       |
| 0x2c        | RW   | Interrupt type clear     |
| 0x30        | RW   | interrupt polarity set   |
| 0x34        | RW   | Interrupt polarity clear |
| 0x38        | R    | Interrupt status         |
| 0x38        | W    | Interrupt clear          |
| 0x400-0x7fc | W    | Byte 0 masked access     |
| 0x800-0xbfc | W    | Byte 1 masked access     |

### AHB GPIO IP的具体实现
#### 解码
第一步是对输入的AHB信号进行解码，将本来处在两个时钟周期的控制信号与数据合并到一个周期，产生了IOSEL，IOWRITE，IOTRANS 三个控制信号。之后将AHB控制信号解码为valid + 位选择信号（iop_byte_strobe）
```systemverilog
  // Detect a valid write to this slave
  wire        write_trans  = IOSEL & IOWRITE & IOTRANS;
  wire  [1:0] iop_byte_strobe;
  // Generate byte strobes to allow the GPIO registers to handle different transfer sizes
  assign iop_byte_strobe[0] = (IOSIZE[1] | ((IOADDR[1]==1'b0) & IOSIZE[0]) | (IOADDR[1:0]==2'b00)) & IOSEL;
  assign iop_byte_strobe[1] = (IOSIZE[1] | ((IOADDR[1]==1'b0) & IOSIZE[0]) | (IOADDR[1:0]==2'b01)) & IOSEL;
```

--------------------------------------------------------------------
#### 读写
而GPIO内部使用的是小端方式储存数据，当系统的其他部分使用大端方式的时候需要对输入与输出的数据进行格式变换。当然这个变换只对要读写大于1byte的数据才有用。
```systemverilog
  // endian conversion
  assign bigendian = (BE!=0) ? 1'b1 : 1'b0;
always @(bigendian or IOSIZE or read_mux_le or IOWDATA)
  begin
    if ((bigendian)&(IOSIZE==2'b10))
      begin
      read_mux = {read_mux_le[ 7: 0],read_mux_le[15: 8],
                  read_mux_le[23:16],read_mux_le[31:24]};
      IOWDATALE = {IOWDATA[ 7: 0],IOWDATA[15: 8],IOWDATA[23:16],IOWDATA[ 31:24]};
      end
    else if ((bigendian)&(IOSIZE==2'b01))
      begin
      read_mux = {read_mux_le[23:16],read_mux_le[31:24],
                  read_mux_le[ 7: 0],read_mux_le[15: 8]};
      IOWDATALE = {IOWDATA[23:16],IOWDATA[ 31:24],IOWDATA[ 7: 0],IOWDATA[15: 8]};
      end
    else
      begin
      read_mux = read_mux_le;
      IOWDATALE = IOWDATA;
      end
  end
```
对于GPIO来说，外部的输入信号都是与处理器异步的信号。在读取的时候需要用Synchroniser 采样同步到处理器时钟域才能使用。 ⚠️这会导致读取IO时有一定延时，要注意。
```systemverilog
  // ----------------------------------------------------------
  // Synchronize input with double stage flip-flops
  // ----------------------------------------------------------
  // Signals for input double flop-flop synchroniser
  reg    [PortWidth-1:0] reg_in_sync1;
  reg    [PortWidth-1:0] reg_in_sync2;

  always @(posedge FCLK or negedge HRESETn)
  begin
    if (~HRESETn)
      begin
      reg_in_sync1 <= {PortWidth{1'b0}};
      reg_in_sync2 <= {PortWidth{1'b0}};
      end
    else
      begin
      reg_in_sync1 <= PORTIN;
      reg_in_sync2 <= reg_in_sync1;
      end
  end

  assign reg_datain = reg_in_sync2;
  // format to 32-bit for data read
  assign reg_datain32 = {{32-PortWidth{1'b0}},reg_datain};
  ```
对于Data register 来说有两种写的方式：普通的寄存器访问，使用mask访问。
对于普通的写来说需要判断：
1. 是否在寄存器地址：处在 0x4
2.  strobe 信号：选择写高位还是低位

对于mask写来说需要：
1. 是否在mask地址：
2. 通过mask地址判断mask 是多少
3. 通过mask进行读写
```systemverilog
  // ----------------------------------------------------------
  // Data Output register
  // ----------------------------------------------------------
  wire [32:0] current_dout_padded;
  wire [PortWidth-1:0] nxt_dout_padded;
  reg  [PortWidth-1:0] reg_dout_padded;
  wire        reg_dout_normal_write0;
  wire        reg_dout_normal_write1;
  wire        reg_dout_masked_write0;
  wire        reg_dout_masked_write1;

  assign      reg_dout_normal_write0 = write_trans &((IOADDR[11:2]  == 10'h000)|(IOADDR[11:2]  == 10'h001)) & iop_byte_strobe[0];
  assign      reg_dout_normal_write1 = write_trans & ((IOADDR[11:2]  == 10'h000)|(IOADDR[11:2]  == 10'h001)) & iop_byte_strobe[1];
  assign      reg_dout_masked_write0 = write_trans &(IOADDR[11:10] == 2'b01) & iop_byte_strobe[0];
  assign      reg_dout_masked_write1 = write_trans &(IOADDR[11:10] == 2'b10) & iop_byte_strobe[1];

  // padding to 33-bit for easier coding
  assign current_dout_padded = {{(33-PortWidth){1'b0}},reg_dout};

  // byte #0
  assign nxt_dout_padded[7:0] = // simple write
     (reg_dout_normal_write0) ? IOWDATALE[7:0] :
     // write lower byte with bit mask
     ((IOWDATALE[7:0] & IOADDR[9:2])|(current_dout_padded[7:0] & (~(IOADDR[9:2]))));

  // byte #0 registering stage
  always @(posedge HCLK or negedge HRESETn)
  begin
    if (~HRESETn)
      reg_dout_padded[7:0] <= 8'h00;
    else if (reg_dout_normal_write0 | reg_dout_masked_write0)
      reg_dout_padded[7:0] <= nxt_dout_padded[7:0];
  end

  // byte #1
  assign nxt_dout_padded[15:8] = // simple write
     (reg_dout_normal_write1) ? IOWDATALE[15:8] :
     // write higher byte with bit mask
     ((IOWDATALE[15:8] & IOADDR[9:2])|(current_dout_padded[15:8] & (~(IOADDR[9:2]))));

  // byte #1 registering stage
  always @(posedge HCLK or negedge HRESETn)
  begin
    if (~HRESETn)
      reg_dout_padded[15:8] <= 8'h00;
    else if (reg_dout_normal_write1 | reg_dout_masked_write1)
      reg_dout_padded[15:8] <= nxt_dout_padded[15:8];
  end

  assign reg_dout[PortWidth-1:0] = reg_dout_padded[PortWidth-1:0];
```
对于其他配置寄存器的写操作就很简单：根据地址判断出到底是set 还是clr 以及对应的byte。
如果是set 就对寄存器同步置1， 反之是clr 就对寄存器同步清0 。
```systemverilog
reg     [PortWidth-1:0] reg_douten_padded;
  integer                 loop1;              // loop variable for register
  wire    [PortWidth-1:0] reg_doutenclr;
  wire    [PortWidth-1:0] reg_doutenset;


  assign    reg_doutenset[7:0]   = ((write_trans == 1'b1) & (IOADDR[11:2]  == 10'h004)& (iop_byte_strobe[0] == 1'b1)) ?  IOWDATALE[7:0] : {8{1'b0}};

  assign    reg_doutenset[15:8]  = ((write_trans == 1'b1) & (IOADDR[11:2]  == 10'h004)& (iop_byte_strobe[1] == 1'b1)) ? IOWDATALE[15:8] : {8{1'b0}};

  assign    reg_doutenclr[7:0]   = ((write_trans == 1'b1) & (IOADDR[11:2]  == 10'h005)& (iop_byte_strobe[0] == 1'b1)) ? IOWDATALE[7:0] : {8{1'b0}};

  assign    reg_doutenclr[15:8]  = ((write_trans == 1'b1) & (IOADDR[11:2]  == 10'h005)& (iop_byte_strobe[1] == 1'b1)) ? IOWDATALE[15:8] : {8{1'b0}};


  // registering stage
  always @(posedge HCLK or negedge HRESETn)
  begin
    if (~HRESETn)
      reg_douten_padded <= {PortWidth{1'b0}};
    else
      for (loop1 = 0; loop1 < PortWidth; loop1 = loop1 + 1)
      begin
        if (reg_doutenset[loop1] | reg_doutenclr[loop1])
          reg_douten_padded[loop1] <= reg_doutenset[loop1];
      end
  end

  assign reg_douten[PortWidth-1:0] = reg_douten_padded[PortWidth-1:0];
```
产生中断的部分实际上就是根据
1. 引脚当前与前一周期的电平
2. 配置的寄存器值

--------------------------------------------------------------------
#### 中断
来产生一系列互斥的信号。将这些信号或起来作为中断信号。当然如果系统不要求对外部中断快速的响应，可以将这些中断合并为一个输出。
```systemverilog
  // ----------------------------------------------------------
  // Interrupt generation
  // ----------------------------------------------------------
  // reg_datain is the synchronized input

  reg   [PortWidth-1:0] reg_last_datain; // last state of synchronized input
  wire  [PortWidth-1:0] high_level_int;
  wire  [PortWidth-1:0] low_level_int;
  wire  [PortWidth-1:0] rise_edge_int;
  wire  [PortWidth-1:0] fall_edge_int;

  // Last input state for edge detection
  always @(posedge FCLK or negedge HRESETn)
  begin
    if (~HRESETn)
      reg_last_datain <= {PortWidth{1'b0}};
    else  if (|reg_inttype)
      reg_last_datain <= reg_datain;
  end

  assign high_level_int =   reg_datain  &                        reg_intpol  & (~reg_inttype);
  assign low_level_int  = (~reg_datain) &                      (~reg_intpol) & (~reg_inttype);
  assign rise_edge_int  =   reg_datain  & (~reg_last_datain) &   reg_intpol  &   reg_inttype;
  assign fall_edge_int  = (~reg_datain) &   reg_last_datain  & (~reg_intpol) &   reg_inttype;
  assign new_raw_int    = high_level_int | low_level_int | rise_edge_int | fall_edge_int;

  // ----------------------------------------------------------
  // Output to external
  // ----------------------------------------------------------
  assign PORTOUT = reg_dout;
  assign PORTEN  = reg_douten;
  assign PORTFUNC = reg_altfunc;

  assign IORDATA   = read_mux;

  // Connect interrupt signal to top level
  assign GPIOINT = reg_intstat;
  assign COMBINT = (|reg_intstat);

```
