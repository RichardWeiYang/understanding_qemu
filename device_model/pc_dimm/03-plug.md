# 插入系统

本来以为设备实例化就算完事了，结果还没完。

这还要从device_set_realized开始看。

```
device_set_realized()
  if (hotplug_ctrl) {
      hotplug_handler_pre_plug(hotplug_ctrl, dev, &local_err);
  }

  if (dc->realize) {
    dc->realize(dev, &local_err);
  }

  if (hotplug_ctrl) {
    hotplug_handler_plug(hotplug_ctrl, dev, &local_err);
  }
```

从上面简化的框架来看，除了设备实例化本身，初始化时在实例化前后分别做了一些准备和善后：

  * hotplug_handler_pre_plug  -> pc_memory_pre_plug
  * hotplug_handler_plug      -> pc_memory_plug

# pc_memory_pre_plug

这个函数在实际插入设备前做一些检测工作：

  * 检测系统是否支持热插拔：有没有acpi，是不是enable了
  * 计算出该插入到哪里，是不是有空间可以个插入

计算出合适的位置后，讲这个值保存在PCDIMMDevice.addr字段中。
这个addr就是该内存条在虚拟机中的物理地址。该地址addr的计算在函数memory_device_get_free_addr()中实现。

# pc_memory_plug

对于普通的dimm设备，插入工作也分成两个步骤：

  * 添加MemoryRegion  -> pc_dimm_plug
  * 添加acpi          -> piix4_device_plug_cb

前者将dimm设备的内存注册到系统中，这个通过memory_region_add_subregion来实现。

后者做的工作主要是在真的hotplug的情况下发送一个acpi的事件，这样虚拟机的内核才能触发内存热插的工作。
假如没有理解错，这个acpi事件会通过中断告知虚拟机。
