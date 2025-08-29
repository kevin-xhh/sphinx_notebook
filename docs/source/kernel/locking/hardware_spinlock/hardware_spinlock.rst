hardware_spinlock
=================

背景
------------------

- Read the fucking source code!
- Talk is cheap, show me the code.  -- Linus Torvalds
- A picture is worth a thousnd words.
- One look is worth a hundred reports.(百闻不如一见)

说明：
- kernel版本: 5.14
- ARM64处理器, Cortex A55
- 使用工具： vscode, Visio, Draw.io

概述
-------------------

  硬件自旋锁模块为异构处理器和非在单一共享操作系统下运行的处理器之间的同步和互斥提供硬件辅助，用于不同cpu之间或者不同Die之间实现锁机制。

	例如：Edge10中A55和C920之间的锁机制；Edge100 Die0和其他Die的A55与A55，或者A55与C920之间的锁机制。

	硬件自旋锁模块是一个硬件模块，用于在多个处理器之间实现同步和互斥。它可以用于保护共享资源，防止多个处理器同时访问同一资源，从而导致数据不一致或其他问题。硬件自旋锁模块可以用于实现各种同步和互斥机制，如互斥锁、读写锁等。

	硬件自旋锁模块的主要功能是提供一个简单的、快速的、可靠的同步和互斥机制。它可以用于保护共享资源，防止多个处理器同时访问同一资源，从而导致数据不一致或其他问题。硬件自旋锁模块可以用于实现各种同步和互斥机制，如互斥锁、读写锁等。

	硬件自旋锁模块的工作原理如下：

	1. 当一个处理器需要访问共享资源时，它会向硬件自旋锁模块发送一个锁请求。
	2. 硬件自旋锁模块会检查锁寄存器的状态，如果锁是空闲的，它会将锁寄存器设置为占用状态，并将状态寄存器设置为占用状态。
	3. 如果锁已经被占用，硬件自旋锁模块会将处理器置于等待状态，直到锁被释放。
	4. 当处理器完成对共享资源的访问后，它会向硬件自旋锁模块发送一个释放请求。
	5. 硬件自旋锁模块会将锁寄存器设置为空闲状态，并将状态寄存器设置为空闲状态。

关键数据结构
------------------------

	Struct hwspinlock_device是一个通常包含硬件锁库的设备。它是由底层hwspinlock实现使用hwspin_lock_register() API注册的。

.. code-block:: c

	/**
  * struct hwspinlock_device - 贯穿多个硬件锁库的设备
  * @dev: 底层设备，将用于调用运行时PM api
  * @ops: 特定于平台的hwspinlock处理接口
  * @base_id: 本设备中第一个锁的id索引
  * @num_locks: 本设备中锁的数量
  * @lock: 'struct hwspinlock'的动态分配数组
  */
  struct hwspinlock_device {
          struct device *dev;
          const struct hwspinlock_ops *ops;
          int base_id;
          int num_locks;
          struct hwspinlock lock[0];
  };

Struct hwspinlock_device包含一个hwspinlock结构数组，每个hwspinlock结构代表一个硬件锁。

.. code-block:: c

    /*
    * struct hwspinlock - 这个结构体代表一个hwspinlock实例
    * @bank: 拥有该锁的hwspinlock_device结构
    * @lock: 由hwspinlock核心初始化并使用
    * @priv: 私有数据，由底层平台特定的hwspinlock DRV拥有
    */
    struct hwspinlock {
        struct hwspinlock_device *bank;
        spinlock_t lock;
        void *priv;
    };

Struct hwspinlock_ops是hwspinlock设备的操作函数指针表。

.. code-block:: c

  /**
  * struct hwspinlock_ops - 特定于平台的hwspinlock处理接口
  *
  * @trylock: 尝试一次打开锁。失败时返回0，成功时返回true。可能不睡眠。
  * @unlock:  释放锁。总是成功。可能不睡眠。
  * @relax:   可选的，特定于平台的放松处理程序，由hwspinlock核心在锁上旋转时调用，在两次连续调用@trylock之间。可能不睡眠
  */
  struct hwspinlock_ops {
    int (*trylock)(struct hwspinlock *lock);
    void (*unlock)(struct hwspinlock *lock);
    void (*relax)(struct hwspinlock *lock);
  };


硬件自旋锁实现者的API
--------------------------------
.. code-block:: c

  int hwspin_lock_register(struct hwspinlock_device *bank, struct device *dev,
              const struct hwspinlock_ops *ops, int base_id, int num_locks);

注册hwspinlock设备, 从底层特定于平台的实现中调用，以便注册一个新的hwspinlock设备（它通常是一个由许多锁组成的库）。应该从进程上下文中调用（此函数可能处于休眠状态）。
成功时返回0，失败时返回适当的错误代码。

设备树中如下：

.. code-block:: dts

  tcsr_mutex: hwlock@1f40000 {
    compatible = "qcom,tcsr-mutex";
    reg = <0x0 0x01f40000 0x0 0x40000>;
    #hwlock-cells = <1>;
  };



.. code-block:: c

  int hwspin_lock_unregister(struct hwspinlock_device *bank);

注销hwspinlock设备,从底层特定于供应商的实现中调用，以注销hwspinlock设备（通常是许多锁的集合）。

应该从进程上下文中调用（此函数可能处于休眠状态）。

成功时返回hwspinlock的地址，错误时返回NULL（例如，如果hwspinlock仍在使用中）。


硬件自旋锁使用者的API
--------------------------------
.. code-block:: c

  struct hwspinlock *hwspin_lock_request(void);

动态分配一个hwspinlock并返回它的地址，或者在一个未使用的hwspinlock不可用的情况下返回NULL。此API的用户通常希望在使用锁id实现同步之前将其通信到远程核心。

应该从进程上下文中调用（可能是休眠）。

.. code-block:: c

  struct hwspinlock *hwspin_lock_request_specific(unsigned int id);

分配一个特定的hwspinlock id并返回它的地址，如果hwspinlock已经在使用，则返回NULL。通常，板代码将调用这个函数，以便为预定义的目的保留特定的hwspinlock id。
应该从进程上下文中调用（可能是休眠）。

.. code-block:: c

	int of_hwspin_lock_get_id(struct device_node *np, int index);

从设备树节点中获取hwspinlock id。这个函数将从设备树节点中获取一个hwspinlock id，以便在调用hwspin_lock_request_specific()时使用。
该函数在成功时返回一个锁id号，如果hwspinlock设备尚未注册到核心，则返回-EPROBE_DEFER，或者其他错误值。

设备树中可以如下定义,获取到的id为tcsr中定义的base id + 此处获取的index(3);

.. code-block:: dts

  smem@80900000 {
    compatible = "qcom,smem";
    reg = <0x0 0x80900000 0x0 0x200000>;
    hwlocks = <&tcsr_mutex 3>; // 3 is the index of the lock in the hwlock bank, tscr_mutex是上面硬件自旋锁实现者的设备树句柄
    no-map;
  };

.. code-block:: c

  int hwspin_lock_free(struct hwspinlock *hwlock);

释放先前分配的hwspinlock；成功时返回0，失败时返回一个适当的错误代码（例如，如果hwspinlock已经空闲，则返回-EINVAL）。

.. code-block:: c

  int hwspin_lock_timeout(struct hwspinlock *hwlock, unsigned int timeout);

用超时限制（以msecs指定）锁定先前分配的hwspinlock。如果hwspinlock已经被占用，该函数将在等待它被释放时进行忙循环，但在超时后放弃。在此函数成功返回后，禁用抢占，因此调用者不能休眠，并建议尽快释放hwspinlock，以尽量减少硬件互连上的远程内核轮询。

成功时返回0，否则返回适当的错误代码（最明显的是-ETIMEDOUT，如果hwspinlock在超时msecs后仍然繁忙）。函数永远不会休眠。

.. code-block:: c

  int hwspin_lock_timeout_irq(struct hwspinlock *hwlock, unsigned int timeout);

用超时限制（以msecs指定）锁定先前分配的hwspinlock。如果hwspinlock已经被占用，该函数将在等待它被释放时进行忙循环，但在超时后放弃。当从这个函数成功返回时，会禁用抢占和本地中断，因此调用方不能休眠，并建议尽快释放hwspinlock。

成功时返回0，否则返回适当的错误代码（最明显的是-ETIMEDOUT，如果hwspinlock在超时msecs后仍然繁忙）。函数永远不会休眠。

.. code-block:: c

  int hwspin_lock_timeout_irqsave(struct hwspinlock *hwlock, unsigned int to,
                                unsigned long *flags);

用超时限制（以msecs指定）锁定先前分配的hwspinlock。如果hwspinlock已经被占用，该函数将在等待它被释放时进行忙循环，但在超时后放弃。在这个函数成功返回时，将禁用抢占，禁用本地中断，并将它们的先前状态保存在给定的标志占位符中。调用方不能休眠，建议尽快释放hwspinlock。
成功时返回0，否则返回适当的错误代码（最明显的是-ETIMEDOUT，如果hwspinlock在超时msecs后仍然繁忙）。
函数永远不会休眠。

.. code-block:: c

  int hwspin_lock_timeout_raw(struct hwspinlock *hwlock, unsigned int timeout);

用超时限制（以msecs指定）锁定先前分配的hwspinlock。如果hwspinlock已经被占用，该函数将在等待它被释放时进行忙循环，但在超时后放弃。
注意：用户必须使用互斥锁或自旋锁来保护获取硬件锁的例程，避免死锁，这将使用户可以在硬件锁下进行一些耗时或可休眠的操作。
成功时返回0，否则返回适当的错误代码（最明显的是-ETIMEDOUT，如果hwspinlock在超时msecs后仍然繁忙）。
函数永远不会休眠。

.. code-block:: c

  int hwspin_lock_timeout_in_atomic(struct hwspinlock *hwlock, unsigned int to);

用超时限制（以msecs指定）锁定先前分配的hwspinlock。如果hwspinlock已经被占用，该函数将在等待它被释放时进行忙循环，但在超时后放弃。
此函数只能从原子上下文中调用，超时值不能超过几毫秒。
成功时返回0，否则返回适当的错误代码（最明显的是-ETIMEDOUT，如果hwspinlock在超时msecs后仍然繁忙）。
函数永远不会休眠。

.. code-block:: c

  int hwspin_trylock(struct hwspinlock *hwlock);

尝试锁定先前分配的hwspinlock，但如果它已被占用，则立即失败。
在此函数成功返回后，禁用抢占，因此调用者不能休眠，并建议尽快释放hwspinlock，以尽量减少硬件互连上的远程内核轮询。
成功时返回0，否则返回一个适当的错误代码（如果hwspinlock已经被占用，最明显的是-EBUSY）。函数永远不会休眠。

.. code-block:: c

  int hwspin_trylock_irq(struct hwspinlock *hwlock);

尝试锁定先前分配的hwspinlock，但如果它已被占用，则立即失败。
当从这个函数成功返回时，将禁用抢占和本地中断，因此调用者不能休眠，并建议尽快释放hwspinlock。
成功时返回0，否则返回一个适当的错误代码（如果hwspinlock已经被占用，最明显的是-EBUSY）。
函数永远不会休眠。

.. code-block:: c

  int hwspin_trylock_irqsave(struct hwspinlock *hwlock, unsigned long *flags);

尝试锁定先前分配的hwspinlock，但如果它已被占用，则立即失败。
在这个函数成功返回时，将禁用抢占，禁用本地中断，并将它们的先前状态保存在给定的标志占位符中。调用方不能休眠，建议尽快释放hwspinlock。
成功时返回0，否则返回一个适当的错误代码（如果hwspinlock已经被占用，最明显的是-EBUSY）。函数永远不会休眠。

.. code-block:: c

  int hwspin_trylock_raw(struct hwspinlock *hwlock);

尝试锁定先前分配的hwspinlock，但如果它已被占用，则立即失败。
注意：用户必须使用互斥锁或自旋锁来保护获取硬件锁的例程，避免死锁，这将使用户可以在硬件锁下进行一些耗时或可休眠的操作。
成功时返回0，否则返回一个适当的错误代码（如果hwspinlock已经被占用，最明显的是-EBUSY）。函数永远不会休眠。

.. code-block:: c

  int hwspin_trylock_in_atomic(struct hwspinlock *hwlock);

尝试锁定先前分配的hwspinlock，但如果它已被占用，则立即失败。
这个函数只能从原子上下文中调用。
成功时返回0，否则返回一个适当的错误代码（如果hwspinlock已经被占用，最明显的是-EBUSY）。函数永远不会休眠。

.. code-block:: c

  void hwspin_unlock(struct hwspinlock *hwlock);

解锁先前锁定的hwspinlock。总是成功，并且可以从任何上下文中调用（该函数从不休眠）。

.. note::
  代码永远不应该解锁一个已经解锁的hwspinlock（没有针对这种情况的保护）。

.. code-block:: c

  void hwspin_unlock_irq(struct hwspinlock *hwlock);

解锁先前锁定的hwspinlock并启用本地中断。调用者永远不应该解锁一个已经解锁的hwspinlock。
这样做被认为是一个bug（对此没有任何保护）。如果从该函数成功返回，则启用抢占和本地中断。这个函数永远不会休眠。

.. code-block:: c

  void hwspin_unlock_irqrestore(struct hwspinlock *hwlock, unsigned long flags);

解锁先前锁定的hwspinlock。
调用者永远不应该解锁一个已经解锁的hwspinlock。这样做被认为是一个bug（对此没有任何保护）。如果从该函数成功返回，则重新启用抢占，并且本地中断的状态恢复到在给定标志处保存的状态。这个函数永远不会休眠。

.. code-block:: c

  void hwspin_unlock_raw(struct hwspinlock *hwlock);

解锁先前锁定的hwspinlock。
调用者永远不应该解锁一个已经解锁的hwspinlock。这样做被认为是一个bug（对此没有任何保护）。这个函数永远不会休眠。

.. code-block:: c

  void hwspin_unlock_in_atomic(struct hwspinlock *hwlock);

解锁先前锁定的hwspinlock。
调用者永远不应该解锁一个已经解锁的hwspinlock。这样做被认为是一个bug（对此没有任何保护）。这个函数永远不会休眠。

.. code-block:: c

  int hwspin_lock_get_id(struct hwspinlock *hwlock);

检索给定hwspinlock的id号。当动态分配hwspinlock时需要这样做：在使用它与远程cpu实现互斥之前，应该将id号传达给我们想要同步的远程任务。
返回hwspinlock的id号，如果hwlock为空则返回-EINVAL。

用户使用示例
^^^^^^^^^^^^^^^^^

.. code-block:: c

  #include <linux/hwspinlock.h>
  #include <linux/err.h>

  int hwspinlock_example1(void)
  {
        struct hwspinlock *hwlock;
        int ret;

        /* dynamically assign a hwspinlock */
        hwlock = hwspin_lock_request();
        if (!hwlock)
                ...

        id = hwspin_lock_get_id(hwlock);
        /* probably need to communicate id to a remote processor now */

        /* take the lock, spin for 1 sec if it's already taken */
        ret = hwspin_lock_timeout(hwlock, 1000);
        if (ret)
                ...

        /*
        * we took the lock, do our thing now, but do NOT sleep
        */

        /* release the lock */
        hwspin_unlock(hwlock);

        /* free the lock */
        ret = hwspin_lock_free(hwlock);
        if (ret)
                ...

        return ret;
  }

  int hwspinlock_example2(void)
  {
        struct hwspinlock *hwlock;
        int ret;

        /*
        * assign a specific hwspinlock id - this should be called early
        * by board init code.
        */
        hwlock = hwspin_lock_request_specific(PREDEFINED_LOCK_ID);
        if (!hwlock)
                ...

        /* try to take it, but don't spin on it */
        ret = hwspin_trylock(hwlock);
        if (!ret) {
                pr_info("lock is already taken\n");
                return -EBUSY;
        }

        /*
        * we took the lock, do our thing now, but do NOT sleep
        */

        /* release the lock */
        hwspin_unlock(hwlock);

        /* free the lock */
        ret = hwspin_lock_free(hwlock);
        if (ret)
                ...

        return ret;
  }