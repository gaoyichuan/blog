---
title: "使用 SpinalHDL 实现 thinpad 模板工程"
date: 2023-04-09T20:00:00+08:00
tags: ['spinalhdl', 'fpga']
categories: ['hardware']
---

## 背景

最近突发奇想，想试一下用 [SpinalHDL](https://spinalhdl.github.io/SpinalDoc-RTD/master/index.html) 做一遍计原实验的感觉，趁机认真学习一下 SpinalHDL。

很多项目是使用 Spinal 开发部分模块，然后在 Verilog 里面例化，和顶层的信号连接起来。但我感觉这样很不优雅，于是想完全用 Spinal 实现 thinpad_top 工程，然后直接使用 Vivado 综合。

<!--more-->

## 顶层信号和接口定义

### 模块接口

为了尽量和 thinpad_top 原版工程保持一致，我希望顶层信号名字和 Verilog 完全一样，这样可以复用 xdc 的约束。由于 thinpad 有很多外设模块，先使用 Spinal 定义它们的接口，下面是 SRAM 接口的例子：

```scala
case class SramPort() extends Bundle {
  val addr = out UInt (20 bits)
  val data = inout Bits (32 bits)
  val be_n = out Bits (4 bits)

  val ce_n = out Bool ()
  val we_n = out Bool ()
  val oe_n = out Bool ()
}
```

可以注意到，这里使用的命名是 snake_case，而不是 Scala 习惯的 CamelCase，这样 Spinal 添加前缀后，刚好和 Verilog 一样，比如 `base_ram_ce_n`。

### 配置类

由于很多外设实验的时候用不到，如果加上这些端口，Spinal 会报 NO DRIVER 错误。这里定义一个 `ThinpadTopConfig` 的类，用于选择性开启一部分外设端口：

```scala
case class ThinpadTopConfig(
    CpldUartEnable: Boolean = false,
    BaseRamEnable: Boolean = true,
    ExtRamEnable: Boolean = true,
    Uart0Enable: Boolean = true,
    FlashEnable: Boolean = false,
    Sl811Enable: Boolean = false,
    Dm9kEnable: Boolean = false,
    VideoEnable: Boolean = false
)
```

### 顶层接口

最后定义一个总的 `ThinpadPorts` 类，根据传进来的 `config` 选择性添加端口就可以了。这里的 `clk_50M` 和 `reset_btn` 暂时先注释掉，后面会提到为什么。

```scala
case class ThinpadPorts(config: ThinpadTopConfig) extends Bundle {
  // val clk_50M = in Bool()
  val clk_11M0592 = in Bool ()
  val push_btn = in Bool ()
  // val reset_btn = in Bool()

  val touch_btn = in Bits (4 bits)
  val dip_sw = in Bits (32 bits)
  val leds = out Bits (16 bits)
  val dpy0 = out Bits (8 bits)
  val dpy1 = out Bits (8 bits)

  val uart = if (config.CpldUartEnable) CpldUartPort() else null
  val base_ram = if (config.BaseRamEnable) SramPort() else null
  val ext_ram = if (config.ExtRamEnable) SramPort() else null
  val uart0 = if (config.Uart0Enable) UartPort() else null
  val flash = if (config.FlashEnable) FlashPort() else null
  val sl811 = if (config.Sl811Enable) SL811UsbPort() else null
  val dm9k = if (config.Dm9kEnable) DM9000EthPort() else null
  val video = if (config.VideoEnable) VGAPort() else null
}
```

### 顶层 Component

最后，顶层元件里面只需要定义一个 `ThinpadPorts` 类的 `io` 就可以了，同时使用 `noIoPrefix()` 方法，去掉端口名字前面的 `io_` 前缀：

```scala
case class ThinpadTop() extends Component {
  val config = ThinpadTopConfig()
  val io = new ThinpadPorts(config)
  noIoPrefix()
 
  ...
}
```

### 信号名字的小坑

thinpad_top 里面，直连串口的信号名字是 `txd` 和 `rxd`，没有前缀。所以这里在定义 `UartPort` 的时候，手动指定了信号的名字。虽然不是很优雅，但至少是可以工作的：

```scala
case class UartPort() extends Bundle {
  val txd = out Bool ()
  val rxd = in Bool ()
  txd.setName("txd")
  rxd.setName("rxd")
}
```

## PLL Blackbox 和时钟域处理

由于 SpinalHDL 没法直接实现 Xilinx FPGA 的 PLL（当然用原语也可以拼出来，但没啥意义），此处使用 BlackBox 功能，引用 Vivado 生成的 IP。

### BlackBox 定义

```scala
class PLLBlackBox extends BlackBox {
  val io = new Bundle {
    val clk_in1 = in Bool()
    val clk_out1 = out Bool()
    ...
  }

  noIoPrefix()
  setBlackBoxName("pll_example")    // 设置与 Vivado 生成的 IP 名字一致
}
```

到这里，就可以让 Spinal 在例化 `PLLBlackBox` 这个模块的时候，生成和 Vivado IP 一样的模块和端口名字了。

### 仿真时的 PLL BlackBox 处理

还有一个小坑，对于一个空的 BlackBox，如果使用 Spinal + Verilator 去仿真，会报错：

```
%Error: .../ThinpadTop.v:43:3: Cannot find file containing module: 'pll_example'
%Error: .../ThinpadTop.v:43:3: This may be because there's no search path specified with -I<dir>.
   43 |   pll_example clkCtrl_pll (
      |   ^~~~~~~~~~~
```

因为此时 Spinal 不会生成一个 Verilog 的 `pll_example` 模块，那么 Verilator 自然找不到。由于我们也没有这个模块的仿真模型，所以给 PLL BlackBox 加上一些假的逻辑，来模拟一个 “PLL”：

```scala
  io.clk_out1 := io.clk_in1
  io.clk_out2 := io.clk_in1
  io.locked := ~io.reset
  spinalSimWhiteBox()
```

注意里面的 `spinalSimWhiteBox()` 语句，这个在 Spinal 文档里面并没有介绍，只有 [一部分 library](https://github.com/SpinalHDL/SpinalHDL/commit/b0f7dfdf3a965e79f893978a2b94618943de8a99) 中用到了。增加这个语句后，Spinal 在仿真时也会生成这个模块，这样 Verilator 就不会报错了。

### 时钟域处理

在这个设计里面，有两个时钟域，一个是输入的 50M 时钟作为默认，另一个是 PLL 产生的（例如 10M）时钟。后者的处理方法比较简单，在文档中也有类似的[样例](https://spinalhdl.github.io/SpinalDoc-RTD/master/SpinalHDL/Examples/Simple%20ones/pll_resetctrl.html)，这里直接贴上代码：

```scala
  val clkCtrl = new Area {
    // PLL blackbox
    val pll = new PLLBlackBox
    pll.io.clk_in1 := ClockDomain.current.readClockWire
    pll.io.reset := ClockDomain.current.readResetWire   // 把用户的复位信号直接连到 PLL 的复位端口

    // Clock domains
    val sysClkDomain = ClockDomain.internal(
      name = "sys",
      frequency = FixedFrequency(10 MHz)
    )
    sysClkDomain.clock := pll.io.clk_out1
    sysClkDomain.reset := ResetCtrl.asyncAssertSyncDeassert(    // 异步复位，同步释放
      input = pll.io.locked,    // 使用 PLL 锁定信号作为复位
      clockDomain = sysClkDomain,
      inputPolarity = LOW,      // 注意极性
      outputPolarity = HIGH,
    )
  }
```

然而，对于前者，由于我们不能替换掉 Spinal 自带的 `ClockDomain.current`，但还希望控制时钟和复位信号的名字，所以使用 `ClockDomain.current.renamePulledWires` 方法，指定信号名：

```scala
  ClockDomain.current.renamePulledWires(clock = "clk_50M", reset = "reset_btn")
```

后面的设计中，都是使用 `sysClkDomain` 这个时钟域，需要写一个 `ClockingArea`：

```scala
  val sys = new ClockingArea(clkCtrl.sysClkDomain) {
    // Logic here
  }
```

## 结论

经过上面一番折腾后，直接使用 `SpinalVerilog` 就可以生成 Verilog 代码了，生成的 Verilog 可以直接放进 Vivado 作为顶层模块进行综合，还是比较方便的。

SpinalHDL 在写硬件描述的时候很爽，但是在与其他工具互操作的时候，还是有一些小坑，不过都可以找到解决的办法。
