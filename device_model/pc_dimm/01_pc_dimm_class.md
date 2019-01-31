# 类型注册

类型看着比较简单。

```
static TypeInfo pc_dimm_info = {
    .name          = TYPE_PC_DIMM,
    .parent        = TYPE_DEVICE,
    .instance_size = sizeof(PCDIMMDevice),
    .instance_init = pc_dimm_init,
    .class_init    = pc_dimm_class_init,
    .class_size    = sizeof(PCDIMMDeviceClass),
    .interfaces = (InterfaceInfo[]) {
        { TYPE_MEMORY_DEVICE },
        { }
    },
};
```

# 类型初始化

初始化也比较简单。

```
static void pc_dimm_class_init(ObjectClass *oc, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(oc);
    PCDIMMDeviceClass *ddc = PC_DIMM_CLASS(oc);
    MemoryDeviceClass *mdc = MEMORY_DEVICE_CLASS(oc);

    dc->realize = pc_dimm_realize;
    dc->unrealize = pc_dimm_unrealize;
    dc->props = pc_dimm_properties;
    dc->desc = "DIMM memory module";

    ddc->get_memory_region = pc_dimm_get_memory_region;
    ddc->get_vmstate_memory_region = pc_dimm_get_memory_region;

    mdc->get_addr = pc_dimm_md_get_addr;
    /* for a dimm plugged_size == region_size */
    mdc->get_plugged_size = pc_dimm_md_get_region_size;
    mdc->get_region_size = pc_dimm_md_get_region_size;
    mdc->fill_device_info = pc_dimm_md_fill_device_info;
}
```

其重要着重的一点是设置了props。我们将在实例化过程中看到这个参数的作用。
