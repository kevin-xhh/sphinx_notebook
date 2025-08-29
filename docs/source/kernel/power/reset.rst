linux reset子系统
======================

背景
------------------

- Read the fucking source code!
- Talk is cheap, show me the code.  -- Linus Torvalds
- A picture is worth a thousnd words.
- One look is worth a hundred reports.(百闻不如一见)

说明：

- kernel版本: 5.10
- ARM64处理器, Cortex A55
- 使用工具： vscode, Visio, Draw.io

概述
--------------

在讲reset framework前，首先要搞明白，什么是复位，它是用来做什么的？简单来说，复位是使器件（芯片内的IP或芯片）进入可以稳定工作的初始状态，避免器件在上电后进入到随机不可控的状态。在芯片设计中，复位是一个很重要的功能。复位被用来将数字电路中的触发器强制设置到一个确定的初始值上，从而使状态机和其他控制电路可以从一个已知的初始状态开始工作。这个初始状态，一般兼具了基础功能、低功耗的特性。所以复位也是低功耗常用的手段，一般会配合时钟控制及电源控制一起使用。

现在芯片大都是SoC（System on Chip），上面包含了cpu、mcu、dsp、cnn、isp、codec、usb等模块功能，这些功能模块相对比较独立，并且在不同场景下，有些是不需要工作的，因此这些不需要工作的功能模块会被关闭（关钟、关电、复位）。在其他场景中，某些模块又要工作，这些需要工作的模块会被打开（开钟、上电、解复位）。各模块的复位会对应到复位寄存器中的某1 bit或者几bit，这样便会有成百上千的复位bit。

Linux为了方便管理及使用这些复位bit，抽象出来一个简单的框架：reset framework。Reset framework为reset的provider（提供复位的驱动）提供统一的复位管理，并为reset的consumer（使用复位的驱动）提供方便且统一的控制接口。这个reset framework的思路、实现和使用都非常简单易懂，比clock framework还要简单易懂，由于时钟设备是级联的，前后之间是有关联的，所以时钟框架将成百上千的时钟设备维护成了一个时钟树，每个枝干就描述了pll、mux、div、gate设备的级联关系，但是复位更简单，一般他并没有这种层级关系，每一个或几个复位bit直接对应一个模块，这些复位bit之间是相互独立的。不过虽然reset framework麻雀虽小，但五脏俱全，它包含了设备模型、分层设计、provider/consumer等设计思想。

