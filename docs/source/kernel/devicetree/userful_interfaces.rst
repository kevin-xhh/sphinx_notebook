userful interfaces
==============================

of_parse_phandle_with_args
----------------------------

.. code-block:: c

  /**
  * of_parse_phandle_with_args() - Find a node pointed by phandle in a list
  * @np:		pointer to a device tree node containing a list
  * @list_name:	property name that contains a list
  * @cells_name:	property name that specifies phandles' arguments count
  * @index:	index of a phandle to parse out
  * @out_args:	optional pointer to output arguments structure (will be filled)
  *
  * This function is useful to parse lists of phandles and their arguments.
  * Returns 0 on success and fills out_args, on error returns appropriate
  * errno value.
  *
  * Caller is responsible to call of_node_put() on the returned out_args->np
  * pointer.
  *
  * Example:
  *
  * phandle1: node1 {
  *	#list-cells = <2>;
  * }
  *
  * phandle2: node2 {
  *	#list-cells = <1>;
  * }
  *
  * node3 {
  *	list = <&phandle1 1 2 &phandle2 3>;
  * }
  *
  * To get a device_node of the `node2' node you may call this:
  * of_parse_phandle_with_args(node3, "list", "#list-cells", 1, &args);
  */
  int of_parse_phandle_with_args(const struct device_node *np, const char *list_name,
        const char *cells_name, int index,
        struct of_phandle_args *out_args)
  {
  int cell_count = -1;

  if (index < 0)
    return -EINVAL;

  /* If cells_name is NULL we assume a cell count of 0 */
  if (!cells_name)
    cell_count = 0;

  return __of_parse_phandle_with_args(np, list_name, cells_name,
              cell_count, index, out_args);
  }


Example:
^^^^^^^^^^^^^^^

.. code-block:: dts

  tcsr_mutex: hwlock@1f40000 {
    compatible = "qcom,tcsr-mutex";
    reg = <0x0 0x01f40000 0x0 0x40000>;
    #hwlock-cells = <1>;
  };

  smem@80900000 {
    compatible = "qcom,smem";
    reg = <0x0 0x80900000 0x0 0x200000>;
    hwlocks = <&tcsr_mutex 3>; // 3 is the index of the lock in the hwlock bank, tscr_mutex是上面硬件自旋锁实现者的设备树句柄
    no-map;
  };

在smem驱动中使用of_hwspin_lock_get_id接口获取id号，代码如下：

.. code-block:: c

  int of_hwspin_lock_get_id(struct device_node *np, int index)
  {
    struct of_phandle_args args;
    struct hwspinlock *hwlock;
    struct radix_tree_iter iter;
    void **slot;
    int id;
    int ret;

    ret = of_parse_phandle_with_args(np, "hwlocks", "#hwlock-cells", index,
            &args);
    ...
  }


of_parse_phandle
----------------------------

.. code-block:: c

  /**
  * of_parse_phandle - Resolve a phandle property to a device_node pointer
  * @np: Pointer to device node holding phandle property
  * @phandle_name: Name of property holding a phandle value
  * @index: For properties holding a table of phandles, this is the index into
  *         the table
  *
  * Returns the device_node pointer with refcount incremented.  Use
  * of_node_put() on it when done.
  */
  struct device_node *of_parse_phandle(const struct device_node *np,
              const char *phandle_name, int index)
  {
    struct of_phandle_args args;

    if (index < 0)
      return NULL;

    if (__of_parse_phandle_with_args(np, phandle_name, NULL, 0,
            index, &args))
      return NULL;

    return args.np;
  }