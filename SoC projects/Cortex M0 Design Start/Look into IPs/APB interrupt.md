%%
interface: pic
how to join multiple interrupt
APB peripherals that integrate interrupts
%%
这一章中我们会提到：
1. Design Start 中与APB有关的中断信号
2. 扩展APB中断的方法

### APB 中断信号
这些信号和[[APB Power Ctrl]] 里面提到的功耗控制引脚一样没有包含在APB Bus中。这些信号被称作 side-band signal，这些信号能够提供额外的功能。APB 中断信号能够直接展示APB 从机重要的状态，而无需进行Bus读写来获得状态。
在DesignStart kit中，APB上带有的中断信号有14个：
1. 每个对应的UART设备带有3个中断：tx_int，rx_int，overflow_int
2. 每个计时器一个中断
3. 看门狗有2个中断：reset_int，普通的int

除了看门狗的中断之外，其他的中断都被接到了一个叫做apbsubsys_interrupt的32位信号，所有未在APB中使用的中断，除了11号，都已经被其他外设占用了。这里可以知道，Cortex M0的中断资源相当紧张。

### 扩展APB中断
为了节约中断的数量我们可以考虑将一些无需快速响应的中断进行合并，成为一个信号。当中断发生的时候，CPU 读取APB上一个特殊的外设以获得现在需要具体执行的中断函数，然后再进行跳转。由于需要进入多一层函数，所以合并了APB中断之后会显著增大响应时间，同时会占用多一点内存。
											                    ![[Combined_Int.png]]
我们使用一个自定义的特殊的外设来提供需要执行的中断，这代表着我们可以自行实现需要的中断仲裁算法，而不用必须得依靠Cortex M0 自带的固定优先级仲裁。这便提供了一点点额外的灵活性。
#### 实现这个特殊的外设
我们假设我们的APB总线上还有位置，同时为了方便实现，我们依然使用固定优先级算法进行仲裁。
由于我们只是读取这个外设，所以它便是一个RO 寄存器。这个寄存器一端接在APB总线上，另一端则直接连在输入的中断上。由于整个APB系统是共用一个时钟的，所以无需在输入端加入同步器。下面就是固定优先级编码器供参考。
```systemverilog
input [31:0] Int_in;
output logic [4:0] Int_num;

integer i;
  always @* begin
   Int_num = 0; // default value if 'in' is all 0's
    for (i=31; i>=0; i=i-1)
        if (Int_in[i]) Int_num = i;
  end

```
