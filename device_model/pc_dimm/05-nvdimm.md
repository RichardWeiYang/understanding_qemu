NVDIMM是PCDIMM的子类型，所以整体的流程基本差不多，着重讲一下几个不同之处。

# 全局

我们先来看一下全局上，一个nvdimm设备初始化并加入系统有哪些步骤。

```
main()
    qemu_opts_foreach(qemu_find_opts("device"), device_init_func, NULL, NULL)
        ...
        device_set_realized()
            hotplug_handler_pre_plug
                ...
                memory_device_pre_plug
                    nvdimm_prepare_memory_region                (1)

            if (dc->realize) {
              dc->realize(dev, &local_err);
            }

            hotplug_handler_plug
                ...
                nvdimm_plug
                    nvdimm_build_fit_buffer                     (2)
    qemu_run_machine_init_done_notifiers()
        pc_machine_done
            acpi_setup
                acpi_build
                    nvdimm_build_acpi                           (3)
                        nvdimm_build_ssdt
                        nvdimm_build_nfit
```

从上面的图中可以看到NVDIMM设备初始化和PCDIMM设备类似，也是有这么几个步骤

  * 插入准备
  * 实例化
  * 插入善后
  * 添加ACPI表

其中大部分的工作和PCDIMM一样，只有标注了1，2，3的三个地方需要额外的工作。

分别是：

  1. 添加nvdimm label区域
  2. 创建NFIT表的内容
  3. 创建SSDT和NFIT并加入acpi

# 分配label区域

按照当前的实现，使用nvdimm设备时，在分配空间的最后挖出一块做label。这个工作在nvdimm_prepare_memory_region中完成。

这块label区域在虚拟机内核中通过DSM方法来操作，后面我们会看到DSM方法是如何虚拟化的。

# 填写nfit信息

nfit表是用来描述nvdimm设备上地址空间组织形式的，所以这部分需要在计算好了dimm地址后进行，有nvdimm_build_fit_buffer完成。

展开这个函数我们可以看到

```
for (; device_list; device_list = device_list->next) {
    DeviceState *dev = device_list->data;

    /* build System Physical Address Range Structure. */
    nvdimm_build_structure_spa(structures, dev);

    /*
     * build Memory Device to System Physical Address Range Mapping
     * Structure.
     */
    nvdimm_build_structure_memdev(structures, dev);

    /* build NVDIMM Control Region Structure. */
    nvdimm_build_structure_dcr(structures, dev);
}
```

也就是分别构建了 SPA, MEMDEV和DCR三个信息。而这些信息就在内核函数add_table中处理。

不看具体内容，我们只看add_table中类型的定义：

```
enum acpi_nfit_type {
	ACPI_NFIT_TYPE_SYSTEM_ADDRESS = 0,
	ACPI_NFIT_TYPE_MEMORY_MAP = 1,
	ACPI_NFIT_TYPE_INTERLEAVE = 2,
	ACPI_NFIT_TYPE_SMBIOS = 3,
	ACPI_NFIT_TYPE_CONTROL_REGION = 4,
	ACPI_NFIT_TYPE_DATA_REGION = 5,
	ACPI_NFIT_TYPE_FLUSH_ADDRESS = 6,
	ACPI_NFIT_TYPE_CAPABILITIES = 7,
	ACPI_NFIT_TYPE_RESERVED = 8	/* 8 and greater are reserved */
};
```

上面三个类型对应了这个枚举类型的0，1，4。

# 构建acpi

最终nvdimm的信息要填写到acpi中，一共有两张表：SSDT和NFIT。这部分工作在nvdimm_build_acpi中完成。

其中NFIT表在nvdimm_build_nfit函数中完成，其实没有太多内容。就是将上面创建好的表内容拷贝过来。

而SSDT这张表就讲究多了，这里面涉及了acpi的一些概念。
