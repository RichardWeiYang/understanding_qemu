NVDIMM是PCDIMM的子类型，所以整体的流程基本差不多，着重讲一下几个不同之处。

# 分配label区域

按照当前的实现，使用nvdimm设备时，在分配空间的最后挖出一块做label。这个工作在nvdimm_prepare_memory_region中完成。

# 填写nfit信息

这部分需要在计算好了dimm地址后进行，有nvdimm_build_fit_buffer完成

# 构建acpi

最终nvdimm的信息要填写到acpi中，一共有两张表：SSDT和NFIT。

这部分工作在nvdimm_build_acpi中完成。

# 全局

用一个图来标示一下这几个工作对应在哪里完成的。

```
main()
    qemu_opts_foreach(qemu_find_opts("device"), device_init_func, NULL, NULL)
        ...
        device_set_realized()
            hotplug_handler_pre_plug
                ...
                memory_device_pre_plug
                    nvdimm_prepare_memory_region                (1)
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

所以也就是这三个地方和PCDIMM设备的添加有所不同。
