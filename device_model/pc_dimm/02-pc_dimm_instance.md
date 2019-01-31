# 实例化

有了类型就可以实例化一个设备了。一台机器默认情况下就又pc-dimm设备，但是我们也可以通过参数人为添加。这样我们可以进一步了解设备注册和生成的步骤。

比如我们可以通过qemu monitor输入一下命令进行内存热插：

> object_add memory-backend-ram,id=ram0,size=1G
> device_add pc-dimm,id=dimm0,memdev=ram0,node0

其中第二行命令就是添加pc-dimm设备的，而第一行命令是表示真实使用的是什么内存。

既然如此在实例化的过程中有一个重要的步骤就是找到第一个object并关联他们。

# devic_initfn

因为PCDIMM是一个设备类型，所以实例化PCDIMM之前需要调用这个函数。

其中重要的一个步骤就是设置属性。

```
do {
    for (prop = DEVICE_CLASS(class)->props; prop && prop->name; prop++) {
        qdev_property_add_legacy(dev, prop, &error_abort);
        qdev_property_add_static(dev, prop, &error_abort);
    }
    class = object_class_get_parent(class);
} while (class != object_class_by_name(TYPE_DEVICE));
```

还记得在PCDIMM类型初始化时设置的props么？在这里就用到了。

```
static Property pc_dimm_properties[] = {
    ...
    DEFINE_PROP_LINK(PC_DIMM_MEMDEV_PROP, PCDIMMDevice, hostmem,
                     TYPE_MEMORY_BACKEND, HostMemoryBackend *),
    DEFINE_PROP_END_OF_LIST(),
};
```

属性中的这个成员memdev，就是用来关联前后端的。至于具体怎么操作，实在是有点复杂。这里略过。

# pc_dimm_realize

那这个具体类型的初始化函数干了什么呢？

仔细一看，这是个空架子。主要就是判断我们的前后端有没有关联好，如果没有则失败。
