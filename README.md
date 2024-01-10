# FPGA时序约束教程

公众号：傅里叶的猫，专注于FPGA、硬件加速、信号处理



- [前言](#前言)
- [读万卷书--时序约束理论篇](#读万卷书--时序约束理论篇)
  - [1. 周期约束理论](#1-周期约束理论)
    - [建立/保持时间](#建立保持时间)
    - [时序模型](#时序模型)
  - [2. I/O约束](#2-io约束)
    - [管脚约束](#管脚约束)
    - [延迟约束](#延迟约束)
  - [3. 时钟周期约束](#3-时钟周期约束)
  - [4. 两种时序例外](#4-两种时序例外)
    - [多周期路径](#多周期路径)
    - [伪路径](#伪路径)
  - [5. xdc约束优先级](#5-xdc约束优先级)
- [行万里路--时序约束实战篇](#行万里路--时序约束实战篇)
  - [1. 梳理时钟树](#1-梳理时钟树)
  - [2. 约束主时钟](#2-约束主时钟)
  - [3. 约束衍生时钟](#3-约束衍生时钟)
  - [4. 延迟约束](#4-延迟约束)
  - [5. 伪路径约束](#5-伪路径约束)
  - [6. 多周期路径约束](#6-多周期路径约束)
- [时序约束辅助工具](#时序约束辅助工具)
  - [1. 时序约束编辑器](#1-时序约束编辑器)
  - [2. 时序约束向导](#2-时序约束向导)
- [Vivado时序约束中Tcl命令的对象及属性](#vivado时序约束中tcl命令的对象及属性)
  - [1. port](#1-port)
  - [2. cell](#2-cell)
  - [3. pin](#3-pin)
  - [4. net](#4-net)




# 前言

&emsp;&emsp;时序约束是FPGA设计中最基本也是最重要的步骤之一，当然，也是难点之一。相信很多朋友都有过跟我一样的经历，看网上很多讲时序约束的文章，对建立/保持时间一顿分析，自己好不容易理解了，发现并不知道这东西在实际中怎么应用。而且网上的时序约束文章虽然不少，但没有一个系统的教程，大部分都是只言片语，因此就萌生了做一个详细的关于时序约束教程的想法。本教程综合整理了网上和书上关于时序约束的资料，在后面也有一章节是专门针对实际工程进行的时序约束，对于刚刚接触FPGA的同学来讲，相信会有不小的帮助，对于FPGA的老手来说，也希望能起到温故而知新的效果。好，废话不多说，开始正文。

本教程参考内容

 - 《vivado从此开始》
 - 《FPGA设计实战演练 高级技巧篇》
 - 《ee5375_timing_fpga》
 - Xilinx的各种UserGuide
 - 网上各个论坛（csdn、知乎、简书）
           

# 读万卷书--时序约束理论篇           

## 1. 周期约束理论

&emsp;&emsp;首先来看什么是时序约束，泛泛来说，就是我们告诉软件（Vivado、ISE等）从哪个pin输入信号，输入信号要延迟多长时间，时钟周期是多少，让软件PAR(Place and  Route)后的电路能够满足我们的要求。因此如果我们不加时序约束，软件是无法得知我们的时钟周期是多少，PAR后的结果是不会提示时序警告的。

&emsp;&emsp;周期约束就是告诉软件我们的时钟周期是多少，让它PAR后要保证在这样的时钟周期内时序不违规。大多数的约束都是周期约束，因为时序约束约的最多是时钟。

&emsp;&emsp;在讲具体的时序约束前，我们先介绍两个概念，在下面的讲解中，会多次用到：

 - 发起端/发起寄存器/发起时钟/发起沿：指的是产生数据的源端
 - 接收端/接收寄存器/捕获时钟/捕获沿：指的是接收数据的目的端

### 建立/保持时间

&emsp;&emsp;讲时序约束，这两个概念要首先介绍，因为周期约束其实就是为了满足建立/保持时间。对于DFF的输入而言，

 - 在clk上升沿到来之前，数据提前一个最小时间量“预先准备好”，这个最小时间量就是建立时间；
 - 在clk上升沿来之后，数据必须保持一个最小时间量“不能变化”，这个最小时间量就是保持时间。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial0.png)

</center>

建立和保持时间是由器件特性决定了，当我们决定了使用哪个FPGA，就意味着建立和保持时间也就确定了。Xilinx FPGA的setup time基本都在0.04ns的量级，hold time基本在0.2ns的量级，不同器件会有所差异，具体可以查对应器件的DC and AC Switching Characteristics，下图列出K7系列的建立保持时间。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial1.png)

</center>





### 时序模型

&emsp;&emsp;典型的时序模型如下图所示，一个完整的时序路径包括源时钟路径、数据路径和目的时钟路径，也可以表示为触发器+组合逻辑+触发器的模型。

<img src="https://technomania.oss-cn-shanghai.aliyuncs.com/image-20240110221407217.png" alt="image-20240110221407217" style="zoom:80%;" />




&emsp;&emsp;该时序模型的要求为

```math
Tclk ≥ Tco＋Tlogic＋Trouting＋Tsetup - Tskew   \tag{公式1} 
```

其中，Tco为发端寄存器时钟到输出时间；Tlogic为组合逻辑延迟；Trouting为两级寄存器之间的布线延迟；Tsetup为收端寄存器建立时间；Tskew为两级寄存器的时钟歪斜，其值等于时钟同边沿到达两个寄存器时钟端口的时间差；Tclk为系统所能达到的最小时钟周期。

&emsp;&emsp;前面几项相加都很容易理解，最后一个Tskew的减法是什么意思呢？因为skew分为两种，positive skew和negative skew，其中positive skew见下图，这相当于增加了后一级寄存器的触发时间。

<img src="https://technomania.oss-cn-shanghai.aliyuncs.com/image-20240110221526663.png" alt="image-20240110221526663" style="zoom:80%;" />

但对于negative skew，则相当于减少了后一级寄存器的触发时间，如下图所示。

<img src="https://technomania.oss-cn-shanghai.aliyuncs.com/image-20240110221716354.png" alt="image-20240110221716354" style="zoom:80%;" />



&emsp;&emsp;在这里，当系统稳定后，都会是positive skew的状态，因此要减去Tskew。但即便是positive skew，综合工具在计算时序时，也不会把多出来的Tskew算进去。

对于同步设计Tskew可忽略(认为其值为0)，因为FPGA中的时钟树会尽量保证到每个寄存器的延迟相同。用下面这个图来表示时序关系就更加容易理解了。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial3.png)


</center>

上面的公式也可以简单的表示为：

```math
Tdata\_path + Tsetup <= Tclk\_path + Tclk \tag{公式2} 
```

公式中提到了建立时间，那保持时间在什么地方体现呢？

&emsp;&emsp;保持时间比较难理解，它的意思是reg1的输出不能太快到达reg2，这是为了防止采到的新数据太快而冲掉了原来的数据。保持时间约束的是同一个时钟边沿，而不是对下一个时钟边沿的约束。


<center>

![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial4.png)
</center>


&emsp;&emsp;reg2在边沿2时刻刚刚捕获reg1在边沿1时刻发出的数据，若reg1在边沿2时刻发出的数据过快到达reg2，则会冲掉前面的数据。因此保持时间约束的是同一个边沿。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial5.png)

</center>

在时钟沿到达之后，数据要保持Thold的时间，因此，要满足：

```math
Tdata\_path =  Tco＋Tlogic＋Trouting ≥ Tskew + Thold \tag{公式3}
```


&emsp;&emsp;这两个公式是FPGA的面试和笔试中经常问到的问题，因为这种问题能反映出应聘者对时序的理解。



&emsp;&emsp;在公式1中，Tco跟Tsu一样，也取决于芯片工艺，因此，一旦芯片型号选定就只能通过Tlogic和Trouting来改善Tclk。其中，Tlogic和代码风格有很大关系，Trouting和布局布线的策略有很大关系。


&emsp;&emsp;关于时序约束的基本理论就讲这么多，下面讲具体的约束。

## 2. I/O约束

&emsp;&emsp;I/O约束是必须要用的约束，又包括管脚约束和延迟约束。

### 管脚约束

&emsp;&emsp;管脚约束就是指管脚分配，我们要指定管脚的PACKAGE_PIN和IOSTANDARD两个属性的值,前者指定了管脚的位置,后者指定了管脚对应的电平标准。

&emsp;&emsp;在vivado中，使用如下方式在xdc中对管脚进行约束。

```
set_property -dict {PACKAGE_PIN AJ16  IOSTANDARD  LVCMOS18} [get_ports "led[0]"    ]
```

除了管脚位置和电平，还有一个大家容易忽略但很容易引起错误的就是端接，当我们使用差分电平时比如LVDS，在在V6中我们使用`IBUFDS`来处理输入的差分信号时，可以指定端接为TRUE。

```
   IBUFDS #(
      .DIFF_TERM("TRUE"),       // Differential Termination
      .IOSTANDARD("DEFAULT")     // Specify the input I/O standard
   ) IBUFDS_inst (
      .O(O),  // Buffer output
      .I(I),  // Diff_p buffer input (connect directly to top-level port)
      .IB(IB) // Diff_n buffer input (connect directly to top-level port)
   );
```

但在Ultrascale中的IBUFDS，却把端接这个选项去掉了

```
   IBUFDS #(
      .DQS_BIAS("FALSE")  // (FALSE, TRUE)
   )
   IBUFDS_inst (
      .O(O),   // 1-bit output: Buffer output
      .I(I),   // 1-bit input: Diff_p buffer input (connect directly to top-level port)
      .IB(IB)  // 1-bit input: Diff_n buffer input (connect directly to top-level port)
   );
```

我们必须要在xdc或I/O Pors界面中，手动指定，否则可能会出错。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial6.png)

</center>

笔者之前就采过一个坑，当连续输入的数据为11101111这种时，中间那个0拉不下来，还是1，同样也会发生在000010000，这样就导致数据传输错误，后来才发现是端接忘记加。

&emsp;&emsp;当综合完成后，我们可以点击DRC，进行设计规则检查，这一步可以报出一些关键问题，比如时钟端口未分配在时钟引脚上等。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial7.png)

</center>




### 延迟约束

&emsp;&emsp;延迟约束用的是`set_input_delay`和`set_output_delay`，分别用于input端和output端，其时钟源可以是时钟输入管脚，也可以是虚拟时钟。**但需要注意的是，这个两个约束并不是起延迟的作用**，具体原因下面分析。

 - set_input_delay

&emsp;&emsp;这个约束跟ISE中的`OFFSET=IN`功能相同，但设置方式不同。下图所示即为input delay的约束说明图。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial8.png)

</center>

从图中很容易理解，

```math
T\_inputdelay = Tco + TD
```

当满足图中的时序时，最大延迟为2ns，最小延迟为1ns。

因此，需要加的时序约束为：

```
create_clock -name sysclk -period 10 [get_ports clkin]
set_input_delay 2 -max -clock sysclk [get_ports Datain]
set_input_delay 1 -min -clock sysclk [get_ports Datain]
```



 - set_output_delay

&emsp;&emsp;set_output_delay的用法跟set_input_delay十分相似，但平时用的也不太多，这里就不再展开讲了。我们上面讲set_input_delay的描述中，大家可以看到，这个约束是告诉vivado我们的输入信号和输入时钟之间的延迟关系，跟下面要讲的时钟周期约束是一个原理，让vivado在这个前提下去Place and Route。**并不是调节输入信号的延迟**，因为身边有不少的FPGA工程师在没用过这个约束指令之前，都以为这是调节延迟的约束。

&emsp;&emsp;如果要调整输入信号的延迟，只能使用IDELAY，在V6中，IDELAY模块有32个tap值，每个tap可延迟78ps，这样总共差不多是2.5ns。




## 3. 时钟周期约束

&emsp;&emsp;时钟周期约束，顾名思义，就是我们对时钟的周期进行约束，这个约束是我们用的最多的约束了，也是最重要的约束。

&emsp;&emsp;下面我们讲一些Vivado中常用的约束指令。

 **1. Create_clock**

&emsp;&emsp;在Vivado中使用`create_clock`来创建时钟周期约束。使用方法为：

```
create_clock -name <name> -period <period> -waveform {<rise_time> <fall_time>} [get_ports <input_port>]
```


| 参数      | 含义                                                         |
| --------- | ------------------------------------------------------------ |
| -name     | 时钟名称                                                     |
| -period   | 时钟周期，单位为ns                                           |
| -waveform | 波形参数，第一个参数为时钟的第一个上升沿时刻，第二个参数为时钟的第一个下降沿时刻 |
| -add      | 在同一时刻源上定义多个时钟时使用                             |

&emsp;&emsp;这里的时钟必须是主时钟`primary clock`，**主时钟**通常有两种情形:一种是时钟由外部时钟源提供，通过时钟引脚进入FPGA，该时钟引脚绑定的时钟为主时钟:另一种是高速收发器(GT)的时钟RXOUTCLK或TXOUTCLK。对于7系列FPGA，需要对GT的这两个时钟手工约束：对于UltraScale FPGA，只需对GT的输入时钟约束即可，Vivado会自动对这两个时钟约束。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial9.png)

</center>

&emsp;&emsp;如何确定主时钟是时钟周期约束的关键，除了根据主时钟的两种情形判断之外，还可以借助Tcl脚本判断。

&emsp;&emsp;在vivado自带的example project里面，打开CPU(HDL)的工程，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial10.png)

</center>

把工程的xdc文件中，`create_clock`的几项都注释掉。这里解释下端口（Port）和管脚（Pin）。端口通常用get_ports命令获取，管脚使用get_pins命令获取。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial11.png)

</center>

再`Open Synthesized Design`或者`Open Implementation Design`，并通过以下两种方式查看主时钟。

 - 方式一

运行tcl指令`report_clock_networks -name mainclock`，显示结果如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial12.png)

</center>

 - 方式二
   运行tcl指令`check_timing -override_defaults no_clock`，显示结果如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial13.png)

</center>

***Vivado中的tcl命令行相当好用，有很多的功能，大家可以开始习惯用起来了。***

&emsp;&emsp;对于高速收发器的时钟，我们也以Vivado中的CPU example工程为例，看下Xilinx官方是怎么约束的。


```
# Define the clocks for the GTX blocks
create_clock -name gt0_txusrclk_i -period 12.8 [get_pins mgtEngine/ROCKETIO_WRAPPER_TILE_i/gt0_ROCKETIO_WRAPPER_TILE_i/gtxe2_i/TXOUTCLK]
create_clock -name gt2_txusrclk_i -period 12.8 [get_pins mgtEngine/ROCKETIO_WRAPPER_TILE_i/gt2_ROCKETIO_WRAPPER_TILE_i/gtxe2_i/TXOUTCLK]
create_clock -name gt4_txusrclk_i -period 12.8 [get_pins mgtEngine/ROCKETIO_WRAPPER_TILE_i/gt4_ROCKETIO_WRAPPER_TILE_i/gtxe2_i/TXOUTCLK]
create_clock -name gt6_txusrclk_i -period 12.8 [get_pins mgtEngine/ROCKETIO_WRAPPER_TILE_i/gt6_ROCKETIO_WRAPPER_TILE_i/gtxe2_i/TXOUTCLK]
```






&emsp;&emsp;当系统中有多个主时钟，且这几个主时钟之间存在确定的相位关系时，需要用到`-waveform`参数。如果有两个主时钟，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial14.png)

</center>


则时钟约束为：

```
create_clock -name clk0 -period 10.0 -waveform {0 5} [get_ports clk0]
create_clock -name clk1 -period 8.0 -waveform {2 8} [get_ports clk1]
```

约束中的数字的单位默认是ns，若不写`wavefrom`参数，则默认是占空比为50%且第一个上升沿出现在0时刻。使用`report_clocks`指令可以查看约束是否生效。还是上面的CPU的例子，把约束都还原到最初的状态。执行`report_clocks`后，如下所示，我们只列出其中几项内容。

```
Clock Report

Clock           Period(ns)  Waveform(ns)    Attributes  Sources
sysClk          10.000      {0.000 5.000}   P           {sysClk}
gt0_txusrclk_i  12.800      {0.000 6.400}   P           {mgtEngine/ROCKETIO_WRAPPER_TILE_i/gt0_ROCKETIO_WRAPPER_TILE_i/gtxe2_i/TXOUTCLK}
...


====================================================
Generated Clocks
====================================================

Generated Clock   : clkfbout
Master Source     : clkgen/mmcm_adv_inst/CLKIN1
Master Clock      : sysClk
Multiply By       : 1
Generated Sources : {clkgen/mmcm_adv_inst/CLKFBOUT}

Generated Clock   : cpuClk_4
Master Source     : clkgen/mmcm_adv_inst/CLKIN1
Master Clock      : sysClk
Edges             : {1 2 3}
Edge Shifts(ns)   : {0.000 5.000 10.000}
Generated Sources : {clkgen/mmcm_adv_inst/CLKOUT0}
...

```


&emsp;&emsp;一般来讲，我们的输入时钟都是差分的，此时我们只对P端进行约束即可。如果同时约束了P端和N端，通过`report_clock_interaction`命令可以看到提示unsafe。这样既会增加内存开销，也会延长编译时间。

 **2. create_generated_clock**

其使用方法为：

```
create_generated_clock -name <generated_clock_name> \
                       -source <master_clock_source_pin_or_port> \
                       -multiply_by <mult_factor> \
                       -divide_by <div_factor> \
                       -master_clock <master_clk> \
                       <pin_or_port>
```

| 参数         | 含义               |
| ------------ | ------------------ |
| -name        | 时钟名称           |
| -source      | 产生该时钟的源时钟 |
| -multiply_by | 源时钟的多少倍频   |
| -divide_by   | 源时钟的多少分频   |



&emsp;&emsp;从名字就能看出来，这个是约束我们在FPGA内部产生的衍生时钟， 所以参数在中有个`-source`，就是指定这个时钟是从哪里来的，这个时钟叫做`master clock`，是指上级时钟，区别于`primary clock`。
它可以是我们上面讲的primary clock，也可以是其他的衍生时钟。该命令不是设定周期或波形，而是描述时钟电路如何对上级时钟进行转换。这种转换可以是下面的关系：

 - 简单的频率分频
 - 简单的频率倍频
 - 频率倍频与分频的组合，获得一个非整数的比例，通常由MMCM或PLL完成
 - 相移或波形反相
 - 占空比改变
 - 上述所有关系的组合

<br />

&emsp;&emsp;**衍生时钟**又分两种情况：

  1. Vivado自动推导的衍生时钟
  2. 用户自定义的衍生时钟

&emsp;&emsp;首先来看第一种，如果使用PLL或者MMCM，则Vivado会自动推导出一个约束。大家可以打开Vivado中有个叫`wavegen`的工程，在这个工程中，输入时钟经过PLL输出了2个时钟，如下图所示。
（补充：关于DCM/DLL/PLL/MMCM的区别，可参考我写的另一篇文章[DCM/DLL/PLL/MMCM区别](https://mp.weixin.qq.com/s?__biz=MzU4ODY5ODU5Ng==&mid=2247484106&idx=1&sn=82983a8086732717298436e067a64d4d&chksm=fdd98441caae0d57c99c5b22cf72bfaee2372824406014680be9df1f8d85b3071182fe43656c&mpshare=1&scene=1&srcid=0928ySJ3ud0vfaGS85Teu5Xw&sharer_sharetime=1571051171309&sharer_shareid=296cfe717a7da125d89d5a7bcdf65c18&key=6234e09828e71f223a5bbb62942587523cffdc550c50d6713403e50f0f1a03c87c5b1a6fae054a425e6f27eabfd6e48eb8fd421c5841d8d8b3b054113d8e8650ff4a65e51fa211ebe10dc0a436635167&ascene=1&uin=MzkzMzM2Nzc1&devicetype=Windows+7&version=62070141&lang=zh_CN&pass_ticket=b15xLCDB%2FBp7ALLPd%2FcqR3Z4qIXNCYrUowj5c4g9AzKzb29vj6R%2F%2BP8z8RJvomTk)

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial15.png)

</center>


&emsp;&emsp;但在xdc文件中，并未对这2个输出时钟进行约束，只对输入的时钟进行了约束，若我们使用`report_clocks`指令，则会看到：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial16.png)

</center>

*注：有三个约束是因为PLL会自动输出一个反馈时钟*

&emsp;&emsp;自动推导的好处在于当MMCM/PLL/BUFR的配置改变而影响到输出时钟的频率和相位时，用户无需改写约束，Vivado仍然可以自动推导出正确的频率/相位信息。劣势在于，用户并不清楚自动推导出的衍生钟的名字，当设计层次改变时，衍生钟的名字也有可能改变。但由于该衍生时钟的约束并非我们自定义的，因此可能会没有关注到它名字的改变，当我们使用者这些衍生时钟进行别的约束时，就会出现错误。


&emsp;&emsp;解决办法是用户自己手动写出自动推导的衍生时钟的名字，也仅仅写出名字即可，其余的不写。如下所示。

```
create_generated_clock -name <generated_clock_name> \
                       -source <master_clock_source_pin_or_port>
```

这一步很容易会被提示critical warning，其实有个很简单的方法，就是name和source都按照vivado中生成的来。具体我们到后面的例子中会讲到。


 **3. set_clock_groups**

 使用方法为：

 ```
set_clock_groups -asynchronous -group <clock_name_1> -group <clock_name_2>
set_clock_groups -physically_exclusive  -group <clock_name_1> -group <clock_name_2>
 ```

&emsp;&emsp;这个约束常用的方法有三种，第一种用法是当两个主时钟是异步关系时，使用`asynchronous`来指定。这个在我们平时用的还是比较多的，一般稍微大点的工程，都会出现至少两个主时钟，而且这两个时钟之间并没有任何的相位关系，这时就要指定：

```
create_clock -period 10 -name clk1 [get_ports clk1]
create_clock -period 8 -name clk2 [get_ports clk2]
set_clock_groups -asynchronous -group clk1 -group clk2
```

&emsp;&emsp;第二种用法是当我们需要验证同一个时钟端口在不同时钟频率下能否获得时序收敛时使用。比如有两个异步主时钟clk1和clk2，需要验证在clk2频率为100MHz，clk1频率分别为50MHz、100MHz和200MHz下的时序收敛情况，我们就可以这样写。

```
create_clock -name clk1A -period 20.0 [get_ports clk1]
create_clock -name clk1B -period 10.0 [get_ports clk1] -add
create_clock -name clk1C -period 5.0  [get_ports clk1] -add 
create_clock -name clk2 -period 10.0 [get_ports clk2]
set_clock_groups -physically_exclusive -group clk1A -group clk1B -group clk1C
set_clock_groups -asynchronous -group "clk1A clk1B clk1C" -group clk2
```

&emsp;&emsp;第三种用法就是当我们使用BUFGMUX时，会有两个输入时钟，但只会有一个时钟被使用。比如MMCM输入100MHz时钟，两个输出分别为50MHz和200MHz，这两个时钟进入了BUFGMUX，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial17.png)
</center>

在这种情况下，我们需要设置的时序约束如下：

```
set_clock_groups -logically_exclusive \
-group [get_clocks -of [get_pins inst_mmcm/inst/mmcm_adv_inst/CLKOUT0]] \
-group [get_clocks -of [get_pins inst_mmcm/inst/mmcm_adv_inst/CLKOUT1]]
```


 **4. 创建虚拟时钟**

&emsp;&emsp;虚拟时钟通常用于设定对输入和输出的延迟约束，这个约束其实是属于IO约束中的延迟约束，之所以放到这里来讲，是因为虚拟时钟的创建，用到了本章节讲的一些理论。时序时钟和前面讲的延迟约束的使用场景不太相同。顾名思义，虚拟时钟，就是没有与之绑定的物理管脚。
虚拟时钟主要用于以下三个场景：

 - 外部IO的参考时钟并不是设计中的时钟
 - FPGA I/O路径参考时钟来源于内部衍生时钟，但与主时钟的频率关系并不是整数倍
 - 针对I/O指定不同的jitter和latency


&emsp;&emsp;简而言之，之所以要创建虚拟时钟，对于输入来说，是因为输入到FPGA数据的捕获时钟是FPGA内部产生的，与主时钟频率不同；或者PCB上有Clock Buffer导致时钟延迟不同。对于输出来说，下游器件只接收到FPGA发送过去的数据，并没有随路时钟，用自己内部的时钟去捕获数据。



&emsp;&emsp;如下图所示，在FPGA的A和B端口分别有两个输入，其中捕获A端口的时钟是主时钟，而捕获B端口的时钟是MMCM输出的衍生时钟，而且该衍生时钟与主时钟的频率不是整数倍关系。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial18.png)

</center>


&emsp;&emsp;这种情况下时序约束如下：

```
create_clock -name sysclk -period 10 [get_ports clkin]
create_clock -name virclk -period 6.4
set_input_delay 2 -clock sysclk [get_ports A]
set_input_delay 2 -clock virclk [get_ports B]
```

可以看到，创建虚拟时钟用的也是`create_clock`约束，但后面并没有加`get_ports`参数，因此被称为虚拟时钟。

&emsp;&emsp;再举个输出的例子，我们常用的UART和SPI，当FPGA通过串口向下游器件发送数据时，仅仅发过去了uart_tx这个数据，下游器件通过自己内部的时钟去捕获uart_tx上的数据，这就需要通过虚拟时钟来约束；而当FPGA通过SPI向下游器件发送数据时，会发送sclk/sda/csn三个信号，其中sclk就是sda的随路时钟，下游器件通过sclk去捕获sda的数据，而不是用自己内部的时钟，这是就不需要虚拟时钟，直接使用`set_output_delay`即可。


注意，虚拟时钟必须在约束I/O延迟之前被定义。


  5. 最大最小延迟约束

&emsp;&emsp;顾名思义，就是设置路径的max/min delay，主要应用场景有两个：

 - 输入管脚的信号经过组合逻辑后直接输出到管脚
 - 异步电路之间的最大最小延迟

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial19.png)

</center>

设置方式为：

```
set_max_delay <delay> [-datapath_only] [-from <node_list>][-to <node_list>][-through <node_list>]
set_min_delay <delay> [-from <node_list>] [-to <node_list>][-through <node_list>]
```

| 参数     | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| -from    | 有效的起始节点包含:时钟,input(input)端口,或时序单元(寄存器,RAM)的时钟引脚. |
| -to      | 有效的终止节点包含:时钟,output(inout)端口或时序单元的数据端口. |
| -through | 有效的节点包含:引脚,端口,线网.                               |

&emsp;&emsp;max/min delay的约束平时用的相对少一些，因为在跨异步时钟域时，我们往往会设置`asynchronous`或者`false_path`。对于异步时钟，我们一般都会通过设计来保证时序能够收敛，而不是通过时序约束来保证。


## 4. 两种时序例外


### 多周期路径


&emsp;&emsp;上面我们讲的是时钟周期约束，默认按照单周期关系来分析数据路径，即数据的发起沿和捕获沿是最邻近的一对时钟沿。如下图所示。


<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial20.png)

</center>

&emsp;&emsp;默认情况下,保持时间的检查是以建立时间的检查为前提,即总是在建立时间的前一个时钟周期确定保持时间检查。这个也不难理解，上面的图中，数据在时刻1的边沿被发起，建立时间的检查是在时刻2进行，而保持时间的检查是在时刻1（如果这里不能理解，再回头看我们讲保持时间章节的内容），因此保持时间的检查是在建立时间检查的前一个时钟沿。


&emsp;&emsp;但在实际的工程中，经常会碰到数据被发起后，由于路径过长或者逻辑延迟过长要经过多个时钟周期才能到达捕获寄存器；又或者在数据发起的几个周期后，后续逻辑才能使用。这时如果按照单周期路径进行时序检查，就会报出时序违规。因此就需要我们这一节所讲的多周期路径了。

多周期约束的语句是：

```
set_multicycle_path <num_cycles> [-setup|-hold] [-start|-end][-from <startpoints>] [-to <endpoints>] [-through <pins|cells|nets>]
```


| 参数                       | 含义                    |
| -------------------------- | ----------------------- |
| num_cycles [-setup  -hold] | 建立/保持时间的周期个数 |
| [-start  -end]             | 参数时钟选取            |
| -from <startpoint>         | 发起点                  |
| -to <endpoint>             | 捕获点                  |
| -through <pins/cells/nets> | 经过点                  |


对于建立时间，num_cycles是指多周期路径所需的时钟周期个数；对于保持时间，num_cycles是指相对于默认的捕获沿，实际捕获沿应回调的周期个数。


发起沿和捕获沿可能是同一个时钟，也可能是两个时钟，参数`start`和`end`就是选择参考时钟是发送端还是接收端。


 - start表示参考时钟为发送端（发端）所用时钟，对于保持时间的分析，若后面没有指定`start`或`end`，则默认为为-start；
 - end表示参考时钟为捕获端（收端）所用时钟,对于建立时间的分析，若后面没有指定`start`或`end`，则默认为为-end；

上面这两句话也不难理解，因为setup-time是在下一个时钟沿进行捕获时的约束，因此默认是对接收端的约束；而hold-up-time是对同一个时钟沿的约束，目的是发送端不能太快，是对发送端的约束。

&emsp;&emsp;对于单周期路径来说，setup的num_cycles为1，hold的num_cycles为0.


&emsp;&emsp;多周期路径要分以下几种情况进行分析：

**1. 单时钟域**

&emsp;&emsp;即发起时钟和捕获时钟是同一个时钟，其多周期路径模型如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial21.png)

</center>

&emsp;&emsp;单时钟域的多周期路径常见于带有使能的电路中，我们以双时钟周期路径为例，其实现电路如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial22.png)

</center>

&emsp;&emsp;若我们没有指定任何的约束，默认的建立/保持时间的分析就像我们上面所讲的单周期路径，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial23.png)

</center>

&emsp;&emsp;但由于我们的的数据经过了两个时钟周期才被捕获，因此建立时间的分析时需要再延迟一个周期的时间。

采用如下的时序约束：

```
set_multicycle_path 2 -setup -from [get_pins data0_reg/C] -to [get_pins data1_reg/D]
```

在建立时间被修改后，保持时间也会自动调整到捕获时钟沿的前一个时钟沿，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial24.png)

</center>

很明显，这个保持时间检查是不对的，因为保持时间的检查针对的是同一个时钟沿，因此我们要把保持时间往回调一个周期，需要再增加一句约束：

```
set_multicycle_path 1 -hold -end -from [get_pins data0_reg/C]  -to [get_pins data1_reg/D]
```

这里加上`-end`参数是因为我们要把捕获时钟沿往前移，因此针对的是接收端，但由于我们这边讲的是单时钟域，发送端和接收端的时钟是同一个，因此`-end`可以省略。这样，完整的时序约束如下：

```
set_multicycle_path 2 -setup -from [get_pins data0_reg/C] -to [get_pins data1_reg/D]
set_multicycle_path 1 -hold  -from [get_pins data0_reg/C]  -to [get_pins data1_reg/D]
```

约束完成后，建立保持时间检查如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial25.png)
</center>

&emsp;&emsp;在单时钟域下，若数据经过N个周期到达，则约束示例如下：

```
set_multicycle_path N -setup -from [get_pins data0_reg/C] -to [get_pins data1_reg/D]
set_multicycle_path N-1 -hold  -from [get_pins data0_reg/C]  -to [get_pins data1_reg/D]
```


**2. 时钟相移**

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial26.png)

</center>

&emsp;&emsp;前面我们讨论的是在单时钟域下，发送端和接收端时钟是同频同相的，如果两个时钟同频不同相怎么处理？

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial27.png)

</center>

&emsp;&emsp;如上图所示，时钟周期为4ns，接收端的时钟沿比发送端晚了0.3ns，若不进行约束，建立时间只有0.3ns，时序基本不可能收敛；而保持时间则为3.7ns，过于丰富。可能有的同学对保持时间会有疑惑，3.7ns是怎么来的？还记得我们上面讲的保持时间的定义么，在0ns时刻，接收端捕获到发送的数据后，要再过3.7ns的时间发送端才会发出下一个数据，因此本次捕获的数据最短可持续3.7ns，即保持时间为3.7ns。

&emsp;&emsp;因此，在这种情况下，我们应把捕获沿向后移一个周期，约束如下：

```
set_multicycle_path 2 -setup -from [get_clocks CLK1] -to [get_clocks CLK2]
```

对setup约束后，hold会自动向后移动一个周期，此时的建立保持时间检查如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial28.png)
</center>

&emsp;&emsp;那如果接收端的时钟比发送端的时钟超前了怎么处理？

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial29.png)
</center>

&emsp;&emsp;同样的，时钟周期为4ns，但接收端时钟超前了0.3ns，从图中可以看出，此时setup是3.7ns，而保持时间是0.3ns。这两个时间基本已经满足了Xilinx器件的要求，因此无需进行约束。


**3. 慢时钟到快时钟的多周期**

&emsp;&emsp;当发起时钟慢于捕获时钟时，我们应该如何处理？

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial30.png)
</center>


&emsp;&emsp;假设捕获时钟频率是发起时钟频率的3倍，在没有任何约束的情况下，Vivado默认会按照如下图所示的建立保持时间进行分析。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial31.png)
</center>


&emsp;&emsp;但我们可以通过约束让建立时间的要求更容易满足，即

```
set_multicycle_path 3 -setup -from [get_clocks CLK1] -to [get_clocks CLK2]
```

跟上面讲的一样，设置了setup，hold会自动变化，但我们不希望hold变化，因此再增加：

```
set_multicycle_path 2 -hold -end -from [get_clocks CLK1] -to [get_clocks CLK2]
```

这里由于发起和捕获是两个时钟，因此`-end`参数是不可省的。加上时序约束后，Vivado会按照下面的方式进行时序分析。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial32.png)
</center>






**4. 快时钟到慢时钟的多周期**

&emsp;&emsp;当发起时钟快于捕获时钟时，我们应该如何处理？

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial33.png)
</center>

&emsp;&emsp;假设发起时钟频率是捕获时钟频率的3倍，在没有任何约束的情况下，Vivado默认会按照如下图所示的建立保持时间进行分析。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial34.png)
</center>

&emsp;&emsp;同理，我们可以通过约束，让时序条件更加宽裕。

```
set_multicycle_path 3 -setup -start -from [get_clocks CLK1] -to [get_clocks CLK2]
set_multicycle_path 2 -hold -from [get_clocks CLK1] -to [get_clocks CLK2]
```

这里的hold约束中没有加`-end`参数，是因为我们把发起时钟回调2个周期，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial35.png)
</center>

针对上面讲的几种多周期路径，总结如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial36.png)
</center>





### 伪路径

&emsp;&emsp;什么是伪路径？伪路径指的是该路径存在，但该路径的电路功能不会发生或者无须时序约束。如果路径上的电路不会发生，那Vivado综合后会自动优化掉，因此我们无需考虑这种情况。

&emsp;&emsp;为什么要创建伪路径？创建伪路径可以减少工具运行优化时间,增强实现结果,避免在不需要进行时序约束的地方画较多时间而忽略了真正需要进行优化的地方。

&emsp;&emsp;伪路径一般用于：

 - 跨时钟域
 - 一上电就被写入数据的寄存器
 - 异步复位或测试逻辑
 - 异步双端口RAM

&emsp;&emsp;可以看出，伪路径主要就是用在异步时钟的处理上，我们上一节讲的多周期路径中，也存在跨时钟域的情况的，但上面我们讲的是两个同步的时钟域。

伪路径的约束为：

```
set_false_path [-setup] [-hold] [-from <node_list>] [-to <node_list>] [-through <node_list>]
```

 - `-from`的节点应是有效的起始点.有效的起始点包含时钟对象,时序单元的clock引脚,或者input(or inout)原语;

 - `-to`的节点应包含有效的终结点.一个有效的终结点包含时钟对象,output(or inout)原语端口,或者时序功能单元的数据输入端口;

 - `-through`的节点应包括引脚,端口,或线网.当单独使用-through时,应注意所有路径中包含-through节点的路径都将被时序分析工具所忽略.

需要注意的是，`-through`是有先后顺序的，下面的两个约束是不同的约束：

```
set_false_path -through cell1/pin1 -through cell2/pin2
set_false_path -through cell2/pin2 -through cell1/pin1
```

因为它们经过的先后顺序不同，伪路径的约束是单向的，并非双向的，若两个时钟域相互之间都有数据传输，则应采用如下约束：

```
set_false_path -from [get_clocks clk1] -to [get_clocks clk2]
set_false_path -from [get_clocks clk2] -to [get_clocks clk1]
```

也可以直接采用如下的方式，与上述两行约束等效：

```
set_clock_groups -async -group [get_clocks clk1] -to [get_clocks clk2]
```

&emsp;&emsp;还有一些其他的约束，比如case analysis、disabling timing和bus_skew等，由于平时用的比较少，这里就不讲了。


## 5. xdc约束优先级

&emsp;&emsp;在xdc文件中，按约束的先后顺序依次被执行，因此，针对同一个时钟的不同约束，只有最后一条约束生效。

&emsp;&emsp;虽然执行顺序是从前到后，但优先级却不同；就像四则运算一样，+-x÷都是按照从左到右的顺序执行，但x÷的优先级比+-要高。

时序例外的优先级从高到低为：

  1. Clock Groups (set_clock_groups)
  2. False Path (set_false_path)
  3. Maximum Delay Path (set_max_delay) and Minimum Delay Path (set_min_delay)
  4. Multicycle Paths (set_multicycle_path)


`set_bus_skew`约束并不影响上述优先级且不与上述约束冲突。原因在于set_bus_skew并不是某条路径上的约束，而是路径与路径之间的约束。

&emsp;&emsp;对于同样的约束，定义的越精细，优先级越高。各对象的约束优先级从高到低位：

  1. ports->pins->cells
  2. clocks。

&emsp;&emsp;路径声明的优先级从高到低位：

  1. -from -through -to
  2. -from -to
  3. -from -through
  4. -from
  5. -through -t
  6. -to
  7. -through。

***优先考虑对象，再考虑路径。***

&emsp;&emsp;Example1：

```
set_max_delay 12 -from [get_clocks clk1] -to [get_clocks clk2]
set_max_delay 15 -from [get_clocks clk1]
```

该约束中，第一条约束会覆盖第二条约束。

&emsp;&emsp;Example2：

```
set_max_delay 12 -from [get_cells inst0] -to [get_cells inst1]
set_max_delay 15 -from [get_clocks clk1] -through [get_pins hier0/p0] -to
[get_cells inst1]
```

该约束中，第一条约束会覆盖第二条约束。


&emsp;&emsp;Example3：

```
set_max_delay 4 -through [get_pins inst0/I0]
set_max_delay 5 -through [get_pins inst0/I0] -through [get_pins inst1/I3]
```

这个约束中，两条都会存在，这也使得时序收敛的难度更大，因为这两条语句合并成了：

```
set_max_delay 4 -through [get_pins inst0/I0] -through [get_pins inst1/I3]
```



# 行万里路--时序约束实战篇


&emsp;&emsp;我们以Vivado子带的`wave_gen`工程为例，该工程的各个模块功能较为明确，如下图所示。为了引入异步时钟域，我们在此程序上由增加了另一个时钟--`clkin2`，该时钟产生脉冲信号`pulse`，`samp_gen`中在`pulse`为高时才产生信号。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial37.png)
</center>


下面我们来一步一步进行时序约束。

## 1. 梳理时钟树

&emsp;&emsp;我们首先要做的就是梳理时钟树，就是工程中用到了哪些时钟，各个时钟之间的关系又是什么样的，如果自己都没有把时钟关系理清楚，不要指望综合工具会把所有问题暴露出来。

&emsp;&emsp;在我们这个工程中，有两个主时钟，四个衍生时钟，如下图所示。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial38.png)
</center>

&emsp;&emsp;确定了主时钟和衍生时钟后，再看各个时钟是否有交互，即clka产生的数据是否在clkb的时钟域中被使用。

&emsp;&emsp;这个工程比较简单，只有两组时钟之间有交互，即：

 - `clk_rx`与`clk_tx`
 - `clk_samp`与`clk2`

其中，`clk_rx`和`clk_tx`都是从同一个MMCM输出的，两个频率虽然不同，但他们却是同步的时钟，因此他们都是从同一个时钟分频得到（可以在Clock Wizard的Port Renaming中看到VCO Freq的大小），因此它们之间需要用`set_false_path`来约束；而`clk_samp`和`clk2`是两个异步时钟，需要用`asynchronous`来约束。


<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial39.png)
</center>

完成以上两步，就可以进行具体的时钟约束操作了。

## 2. 约束主时钟

&emsp;&emsp;在这一节开讲之前，我们先把`wave_gen`工程的`wave_gen_timing.xdc`中的内容都删掉，即先看下在没有任何时序约束的情况下会综合出什么结果？

对工程综合并Implementation后，Open Implemented Design，会看到下图所示内容。


<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial40.png)
</center>

&emsp;&emsp;可以看到，时序并未收敛。可能到这里有的同学就会有疑问，我们都已经把时序约束的内容都删了，按我们第一讲中提到的“因此如果我们不加时序约束，软件是无法得知我们的时钟周期是多少，PAR后的结果是不会提示时序警告的”，这是因为在该工程中，用了一个MMCM，并在里面设置了输入信号频率，因此这个时钟软件会自动加上约束。


&emsp;&emsp;接下来，我们在tcl命令行中输入`report_clock_networks -name main`，显示如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial41.png)
</center>

&emsp;&emsp;可以看出，Vivado会自动设别出两个主时钟，其中clk_pin_p是200MHz，这个是直接输入到了MMCM中，因此会自动约束；另一个输入时钟clk_in2没有约束，需要我们手动进行约束。

或者可以使用`check_timing -override_defaults no_clock`指令，这个指令我们之前的内容讲过，这里不再重复讲了。

&emsp;&emsp;在tcl中输入

```
create_clock -name clk2 -period 25 [get_ports clk_in2]
```

注：在Vivado中，可以直接通过tcl直接运行时序约束脚本，运行后Vivado会自动把这些约束加入到xdc文件中。

再执行`report_clock_networks -name main`，显示如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial42.png)
</center>

可以看到，主时钟都已被正确约束。




## 3. 约束衍生时钟

&emsp;&emsp;系统中有4个衍生时钟，但其中有两个是MMCM输出的，不需要我们手动约束，因此我们只需要对`clk_samp`和`spi_clk`进行约束即可。约束如下：

```
create_generated_clock -name clk_samp -source [get_pins clk_gen_i0/clk_core_i0/clk_tx] -divide_by 32 [get_pins clk_gen_i0/BUFHCE_clk_samp_i0/O]
create_generated_clock -name spi_clk -source [get_pins dac_spi_i0/out_ddr_flop_spi_clk_i0/ODDR_inst/C] -divide_by 1 -invert [get_ports spi_clk_pin]
```

这里需要注意的是，如果该约束中使用`get_pins`（即产生的时钟并非输出到管脚），那么无论是source的时钟还是我们衍生的时钟，在`get_pins`后面的一定是这个时钟最初的产生位置。在视频中我们会具体展示）。

&emsp;&emsp;我们再运行`report_clocks`，显示如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial43.png)
</center>

我们在理论篇的“create_generated_clock”一节中讲到，我们可以重新设置Vivado自动生成的衍生时钟的名字，这样可以更方便我们后续的使用。按照前文所讲，只需设置`name`和`source`参数即可，其中这个`source`可以直接从`report_clocks`中得到，因此我们的约束如下：

```
create_generated_clock -name clk_tx -source [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1] [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT1]
create_generated_clock -name clk_rx -source [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1] [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT0]
```

&emsp;&emsp;大家可以对比下`report_clocks`的内容和约束指令，很容易就能看出它们之间的关系。

把上述的约束指令在tcl中运行后，我们再运行一遍`report_clocks`，显示如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial44.png)
</center>

在时序树的分析中，我们看到，`clk_samp`和`clk2`两个异步时钟之间存在数据交互，因此要进行约束，如下：

```
set_clock_groups -asynchronous -group [get_clocks clk_samp] -group [get_clocks clk2]
```


## 4. 延迟约束

&emsp;&emsp;对于延迟约束，相信很多同学是不怎么用的，主要可能就是不熟悉这个约束，也有的是嫌麻烦，因为有时还要计算PCB上的走线延迟导致的时间差。而且不加延迟约束，Vivado也只是在Timing Report中提示warning，并不会导致时序错误，这也会让很多同学误以为这个约束可有可无。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial45.png)
</center>



&emsp;&emsp;但其实这种想法是不对的，比如在很多ADC的设计中，输出的时钟的边沿刚好是数据的中心位置，而如果我们不加延迟约束，则Vivado会默认时钟和数据是对齐的。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial46.png)
</center>

&emsp;&emsp;对于输入管脚，首先判断捕获时钟是主时钟还是衍生时钟，如果是主时钟，直接用`set_input_delay`即可，如果是衍生时钟，要先创建虚拟时钟，然后再设置delay。对于输出管脚，判断有没有输出随路时钟，若有，则直接使用`set_output_delay`，若没有，则需要创建虚拟时钟。

&emsp;&emsp;在本工程中，输入输出数据管脚的捕获时钟如下表所示：


| 管脚          | 输入输出 | 捕获时钟  | 时钟类型 | 是否有随路时钟 | 是否需要虚拟时钟 |
| ------------- | -------- | --------- | -------- | -------------- | ---------------- |
| rxd_pin       | 输入     | clk_pin_p | 主时钟   | x              | No               |
| txd_pin       | 输出     | clk_tx    | x        | 无             | Yes              |
| lb_sel_pin    | 输入     | clk_tx    | 衍生时钟 | x              | Yes              |
| led_pins[7:0] | 输出     | clk_tx    | x        | 无             | Yes              |
| spi_mosi_pin  | 输出     | spi_clk   | x        | 有             | No               |
| dac_*         | 输出     | spi_clk   | x        | 有             | No               |

&emsp;&emsp;根据上表，我们创建的延迟约束如下，其中的具体数字在实际工程中要根据上下游器件的时序关系（在各个器件手册中可以找到）和PCB走线延迟来决定。未避免有些约束有歧义，我们把前面的所有约束也加进来。

```
# 主时钟约束
create_clock -period 25.000 -name clk2 [get_ports clk_in2]

# 衍生时钟约束
create_generated_clock -name clk_samp -source [get_pins clk_gen_i0/clk_core_i0/clk_tx] -divide_by 32 [get_pins clk_gen_i0/BUFHCE_clk_samp_i0/O]
create_generated_clock -name spi_clk -source [get_pins dac_spi_i0/out_ddr_flop_spi_clk_i0/ODDR_inst/C] -divide_by 1 -invert [get_ports spi_clk_pin]
create_generated_clock -name clk_tx -source [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1] [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT1]
create_generated_clock -name clk_rx -source [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1] [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT0]

# 设置异步时钟
set_clock_groups -asynchronous -group [get_clocks clk_samp] -group [get_clocks clk2]

# 延迟约束
create_clock -period 6.000 -name virtual_clock
set_input_delay -clock [get_clocks -of_objects [get_ports clk_pin_p]] 0.000 [get_ports rxd_pin]
set_input_delay -clock [get_clocks -of_objects [get_ports clk_pin_p]] -min -0.500 [get_ports rxd_pin]
set_input_delay -clock virtual_clock -max 0.000 [get_ports lb_sel_pin]
set_input_delay -clock virtual_clock -min -0.500 [get_ports lb_sel_pin]
set_output_delay -clock virtual_clock -max 0.000 [get_ports {txd_pin {led_pins[*]}}]
set_output_delay -clock virtual_clock -min -0.500 [get_ports {txd_pin {led_pins[*]}}]
set_output_delay -clock spi_clk -max 1.000 [get_ports {spi_mosi_pin dac_cs_n_pin dac_clr_n_pin}]
set_output_delay -clock spi_clk -min -1.000 [get_ports {spi_mosi_pin dac_cs_n_pin dac_clr_n_pin}]
```


## 5. 伪路径约束

&emsp;&emsp;在本章节的“2 约束主时钟”一节中，我们看到在不加时序约束时，Timing Report会提示很多的error，其中就有跨时钟域的error，我们可以直接在上面右键，然后设置两个时钟的伪路径。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial47.png)
</center>

这样会在xdc中自动生成如下约束：

```
set_false_path -from [get_clocks -of_objects [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT0]] -to [get_clocks -of_objects [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT1]]
```

&emsp;&emsp;其实这两个时钟我们已经在前面通过generated指令创建过了，因此get_pins那一长串就没必要重复写了，所以我们可以手动添加这两个时钟的伪路径如下：

```
set_false_path -from [get_clocks clk_rx] -to [get_clocks clk_tx]
```

伪路径的设置是单向的，如果两个时钟直接存在相互的数据的传输，则还需要添加从`clk_tx`到`clk_rx`的路径，这个工程中只有从rx到tx的数据传输，因此这一条就可以了。

&emsp;&emsp;在伪路径一节中，我们讲到过异步复位也需要添加伪路径，`rst_pin`的复位输入在本工程中就是当做异步复位使用，因此还需要添加一句：

```
set_false_path -from [get_ports rst_pin]
```

&emsp;&emsp;对于`clk_samp`和`clk2`，它们之间存在数据交换，但我们在前面已经约束过`asynchronous`了，这里就可以不用重复约束了。

&emsp;&emsp;这里需要提示一点，添加了上面这些约束后，综合时会提示xdc文件的的warning。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial48.png)
</center>

但这可能是Vivado的综合过程中，读取到该约束文件时，内部电路并未全都建好，就出现了没有发现`clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1`等端口的情况，有如下几点证明：

  1. 若把该xdc文件，设置为仅在Implementation中使用，则不会提示该warning
  2. 在Implementation完成后，无论是Timing Report还是通过tcl的`report_clocks`指令，都可以看到这几个时钟已经被正确约束。下图所示即为设置完上面的约束后的Timing Report。






## 6. 多周期路径约束

&emsp;&emsp;多周期路径，我们一般按照以下4个步骤来约束：

  1. 带有使能的数据

&emsp;&emsp;首先来看带有使能的数据，在本工程中的Tming Report中，也提示了同一个时钟域之间的几个路径建立时间不满足要求（具体见“5.伪路径约束”的第一个图），其实这几个路径都是带有使能的路径，使能的周期为2倍的时钟周期，本来就应该在2个时钟周期内去判断时序收敛。因此，我们添加时序约束：

```
set_multicycle_path -from [get_cells {cmd_parse_i0/send_resp_data_reg[*]} -include_replicated_objects] -to [get_cells {resp_gen_i0/to_bcd_i0/bcd_out_reg[*]}] 2
set_multicycle_path -hold -from [get_cells {cmd_parse_i0/send_resp_data_reg[*]} -include_replicated_objects] -to [get_cells {resp_gen_i0/to_bcd_i0/bcd_out_reg[*]}] 1
```

同样的，我们也可以直接点击右键通过GUI的方式进行约束，效果都是一样的。


至于`include_replicated_objects`和`get_cells`这些tcl指令的具体使用方法，我们将会在下一章中讲到。


&emsp;&emsp;在工程的`uart_tx_ctl.v`和`uart_tx_ctl.v`文件中，也存在带有使能的数据，但这些路径在未加多路径约束时并未报出时序错误或者警告。

在接收端，捕获时钟频率是200MHz，串口速率是115200，采用16倍的Oversampling，因此使能信号周期是时钟周期的200e6/115200/16=108.5倍。


在接收端，捕获时钟频率是200MHz，串口速率是115200，采用16倍的Oversampling，因此使能信号周期是时钟周期的166.667e6/115200/16=90.4倍。

&emsp;&emsp;因此，时序约束如下：

```
# 串口接收端
set_multicycle_path -from [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] 108
set_multicycle_path -hold -from [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] 107
# 串口发送端
set_multicycle_path -from [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] 90
set_multicycle_path -hold -from [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] 89
```

约束中的`filter`参数也将在下一章节具体讲解。

  2. 两个有数据交互的时钟之间存在相位差

&emsp;&emsp;在本工程中，没有这种应用场景，因此不需要添加此类约束。



  3. 存在快时钟到慢时钟的路径

&emsp;&emsp;在本工程中，没有这种应用场景，因此不需要添加此类约束。



  4. 存在慢时钟到快时钟的路径

&emsp;&emsp;在本工程中，没有这种应用场景，因此不需要添加此类约束。


综上，我们所有的时序约束如下：

```
# 主时钟约束
create_clock -period 25.000 -name clk2 [get_ports clk_in2]

# 衍生时钟约束
create_generated_clock -name clk_samp -source [get_pins clk_gen_i0/clk_core_i0/clk_tx] -divide_by 32 [get_pins clk_gen_i0/BUFHCE_clk_samp_i0/O]
create_generated_clock -name spi_clk -source [get_pins dac_spi_i0/out_ddr_flop_spi_clk_i0/ODDR_inst/C] -divide_by 1 -invert [get_ports spi_clk_pin]
create_generated_clock -name clk_tx -source [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1] [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT1]
create_generated_clock -name clk_rx -source [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKIN1] [get_pins clk_gen_i0/clk_core_i0/inst/mmcm_adv_inst/CLKOUT0]

# 设置异步时钟
set_clock_groups -asynchronous -group [get_clocks clk_samp] -group [get_clocks clk2]

# 延迟约束
create_clock -period 6.000 -name virtual_clock
set_input_delay -clock [get_clocks -of_objects [get_ports clk_pin_p]] 0.000 [get_ports rxd_pin]
set_input_delay -clock [get_clocks -of_objects [get_ports clk_pin_p]] -min -0.500 [get_ports rxd_pin]
set_input_delay -clock virtual_clock -max 0.000 [get_ports lb_sel_pin]
set_input_delay -clock virtual_clock -min -0.500 [get_ports lb_sel_pin]
set_output_delay -clock virtual_clock -max 0.000 [get_ports {txd_pin {led_pins[*]}}]
set_output_delay -clock virtual_clock -min -0.500 [get_ports {txd_pin {led_pins[*]}}]
set_output_delay -clock spi_clk -max 1.000 [get_ports {spi_mosi_pin dac_cs_n_pin dac_clr_n_pin}]
set_output_delay -clock spi_clk -min -1.000 [get_ports {spi_mosi_pin dac_cs_n_pin dac_clr_n_pin}]

# 伪路径约束
set_false_path -from [get_clocks clk_rx] -to [get_clocks clk_tx]
set_false_path -from [get_ports rst_pin]

# 多周期约束
set_multicycle_path -from [get_cells {cmd_parse_i0/send_resp_data_reg[*]} -include_replicated_objects] -to [get_cells {resp_gen_i0/to_bcd_i0/bcd_out_reg[*]}] 2
set_multicycle_path -hold -from [get_cells {cmd_parse_i0/send_resp_data_reg[*]} -include_replicated_objects] -to [get_cells {resp_gen_i0/to_bcd_i0/bcd_out_reg[*]}] 1

# 串口接收端
set_multicycle_path -from [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] 108
set_multicycle_path -hold -from [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_rx_i0/uart_rx_ctl_i0/* -filter IS_SEQUENTIAL] 107
# 串口发送端
set_multicycle_path -from [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] 90
set_multicycle_path -hold -from [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] -to [get_cells uart_tx_i0/uart_tx_ctl_i0/* -filter IS_SEQUENTIAL] 89
```

&emsp;&emsp;重新Synthesis并Implementation后，可以看到，已经没有了时序错误

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial49.png)

</center>


&emsp;&emsp;仅有的两个warning也只是说rst没有设置input_delay，spi_clk_pin没有设置output_delay，但我们已经对rst设置了伪路径，而spi_clk_pin是我们约束的输出时钟，无需设置output_delay。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial49.png)

</center>


&emsp;&emsp;如果设置了多周期路径后，还是提示`Intra-Clocks Paths`的`setup time`不过，那就要看下程序，是否写的不规范。比如

```
always @ (posedge clk)
begin
    regA <= regB;
    
    if(regA != regB)
        regC <= 4'hf;
    else 
		regC <= {regC[2:0], 1'b0};
    
    if((&flag[3:0]) && regA != regB) 
        regD <= regB;
end
```

这么写的话，如果时钟频率稍微高一点，比如250MHz，就很容易导致从regB到regD的setup time不满足要求。因为begin end里面的代码都是按顺序执行的，要在4ns内完成这些赋值与判断的逻辑，挑战还是挺大的。因此，我们可以改写为：

```
always @ (posedge clk)
begin
    regA <= regB;
end 

always @ (posedge clk)
begin
    if(regA != regB)
        regC <= 4'hf;
    else 
		regC <= {regC[2:0], 1'b0};
end 

always @ (posedge phy_clk)
begin
    if((&flag[3:0]) && regA != regB) 
        regD <= regB;
end 
```

把寄存器的赋值分开，功能还是一样的，只是分到了几个always中，这样就不会导致时序问题了。

# 时序约束辅助工具

&emsp;&emsp;上面我们讲的都是xdc文件的方式进行时序约束，Vivado中还提供了两种图形界面的方式，帮我们进行时序约束：时序约束编辑器（Edit Timing Constraints ）和时序约束向导（Constraints Wizard）。两者都可以在综合或实现后的Design中打开。

## 1. 时序约束编辑器

&emsp;&emsp;打开之后就可显示出我们之前做的所有约束，当然，还可以再添加、删除或修改时序约束。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial51.png)
</center>
    
&emsp;&emsp;比如我们要新添加一个主时钟，先选中左边的`Create Clock`，再点击`+`号添加约束，然后就会看到下面的界面，按下图中步骤操作。
    

<center>

![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial52.png)
</center>

其中，选择时钟按钮会弹出一个新的窗口，如下图所示，我们只需根据时钟名字进行查找并选择即可。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial53.png)
</center>

## 2. 时序约束向导

&emsp;&emsp;时序约束向导可以自动识别出未约束的主时钟，我们把wave_gen工程的xdc文件中对clk2的时钟约束注释掉，重新综合并实现后，打开时序约束向导，可以看到clk2被检测出未约束，点击编辑按钮，设置参数后就可完成约束。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial54.png)
</center>

&emsp;&emsp;时序约束向导会按照主时钟约束、衍生时钟约束、输入延迟约束、输出延迟约束、时序例外约束、异步时钟约束等的顺序引导设计者创建约束。

# Vivado时序约束中Tcl命令的对象及属性

&emsp;&emsp;在前面的章节中，我们用了很多Tcl的指令，但有些指令并没有把所有的参数多列出来解释，这一节，我们就把约束中的Tcl指令详细讲一下。

我们前面讲到过`get_pins`和`get_ports`的区别，而且我们也用过`get_cells`、`get_clocks`和`get_nets`这几个指令，下面就通过一张图直观展现它们的区别。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial55.png)
</center>

`get_clocks`后面的对象是我们之前通过`create_clocks`或者`create_generated_clocks`创建的时钟，不在硬件上直接映射。


&emsp;&emsp;我们再来看下各个命令的属性。

## 1. port

我们可以通过Tcl脚本查看port的所有属性，比如上面的wave_gen工程中，有一个port是`clk_pin_p`，采用如下脚本：

```
set inst [get_ports clk_pin_p]
report_property $inst
```

显示如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial56.png)
</center>

`get_ports`的使用方法如下：

```
# 获取所有端口
get_ports *

# 获取名称中包含data的端口
get_ports *data*

# 获取所有输出端口
get_ports -filter {DIRECTION == OUT}

# 获取所有输入端口
all_inputs

# 获取输入端口中名字包含data的端口
get_ports -filter {DIRECTION == IN} *data*

# 获取总线端口
get_ports -filter {BUS_NAME != ""}
```

## 2. cell

按照上面的同样的方式，获取cell的property，如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial57.png)
</center>

`get_cells`的使用方法如下：

```
# 获取顶层模块
get_cells *

# 获取名称中包含字符gen的模块
get_cells *gen*

# 获取clk_gen_i0下的所有模块
get_cells clk_gen_i0/*

# 获取触发器为FDRE类型且名称中包含字符samp
get_cells -hier filter {REF_NAME == FDRE} *samp*

# 获取所有的时序单元逻辑
get_cells -hier -filter {IS_SEQUENTIAL == 1}

# 获取模块uart_rx_i0下两层的LUT3
get_cells -filter {REF_NAME == LUT3} *uart_tx_i0/*/*
```

## 3. pin

获取pin的property，如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial58.png)
</center>


`get_pins`的使用方法如下：

```
# 获取所有pins
get_pins *

# 获取名称中包含字符led的引脚
get_pins -hier -filter {NAME =~ *led*}

# 获取REF_PIN_NAME为led的引脚
get_pins -hier -filter {REF_PIN_NAME == led}

# 获取时钟引脚
get_pins -hier -filter {IS_CLOCK == 1}

# 获取名称中包含cmd_parse_i0的使能引脚
get_pins -filter {IS_ENABLE == 1} cmd_parse_i0/*/*

# 获取名称中包含字符cmd_parse_i0且为输入的引脚
get_pins -filter {DIRECTION == IN} cmd_parse_i0/*/*
```





## 4. net

获取pin的property，如下：

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial59.png)
</center>



`get_nets`的使用方法如下：

```
# 获取所有nets
get_nets *

# 获取名称中包含字符send_resp_val的网线
get_nets -hier *send_resp_val*
get_nets -filter {NAME =~ *send_resp_val*} -hier

# 获取穿过边界的同一网线的所有部分
get_nets {resp_gen_i0/data4[0]} -segments

# 获取模块cmd_parse_i0下的所有网线
get_nets -filter {PARENT_CELL == cmd_parse_i0} -hier

# 获取模块cmd_parse_i0下的名称中包含字符arg_cnt[]的网线
get_nets -filter {PARENT_CELL == cmd_parse_i0} -hier *arg_cnt[*]
```

&emsp;&emsp;这5个tcl指令的常用选项如下表：

| 命令       | -hierarchy | -filter | -of_objects | -regexp | -nocase |
| ---------- | ---------- | ------- | ----------- | ------- | ------- |
| get_cells  | √          | √       | √           | √       | √       |
| get_nets   | √          | √       | √           | √       | √       |
| get_pins   | √          | √       | √           | √       | √       |
| get_ports  |            | √       | √           | √       | √       |
| get_clocks |            | √       | √           | √       | √       |

&emsp;&emsp;这5个Tcl命令对应的5个对象之间也有着密切的关系，下图所示的箭头的方向表示已知箭头末端对象可获取箭头指向的对象。

<center>


![image](https://raw.githubusercontent.com/Bounce00/pic/master/fpga/timing_toturial60.png)
</center>

&emsp;&emsp;以wave_gen中的`clk_gen_i0`模块为例来说明上面的操作：


```
# 获取模块的输入引脚
get_pins -of [get_cells {clk_gen_i0/clk_core_i0}] -filter {DIRECTION == IN}

# 已知引脚名获取所在模块
get_cells -of [get_pins clk_gen_i0/clk_core_i0/clk_in1_n]

# 已知模块名获取与该模块相连的网线
get_nets -of [get_cells {clk_gen_i0/clk_core_i0}]

# 已知引脚名获取与该引脚相连的网线
get_nets -of [get_pins clk_gen_i0/clk_core_i0/clk_rx]

# 已知时钟引脚获取时钟引脚对应的时钟
get_clocks -of [get_pins clk_gen_i0/clk_core_i0/clk_rx]
```

需要注意的是：

  1. -hier不能和层次分隔符“/”同时使用，但“/”可出现在-filter中
  2. 可根据属性过滤查找目标对象
  3. -filter中的属性为：“==”（相等）、“!=”（不相等）、"=~"（匹配）、"!~"（不匹配），若有多个表达式，其返回值为bool类型时，支持逻辑操作（&& ||）
