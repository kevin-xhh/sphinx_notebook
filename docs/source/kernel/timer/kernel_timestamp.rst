内核时间戳
==============

内核中依赖时间戳的模块，例如printk、ftrace等，根据各自的需要选择不同的时间获取方式。

根据printk的调用路径，printk并没有使用timekeeping的时间接口，而是使用调度模块的接口sched_clock()，sched_clock直接使用时钟源的arch_counter_read()接口读取计数器寄存器值，来转换成时间戳。由此可知，printk的时间戳即使系统休眠也会累计。

.. code-block::

  printk()->_printk()->vprintk()-> vprintk_default()->vprintk_emit()->vprintk_store()-
    >local_clock()->sched_clock()->read_sched_clock()->arch_counter_read()

ftrace依赖timekeeping，通过sys节点可以看到ftrace可以使用的时间戳类型，默认使用boot开机时间作为时间戳，可以echo修改

.. code-block::

  # cat /sys/kernel/tracing/trace_clock
  local global counter uptime perf mono mono_raw [boot] tai