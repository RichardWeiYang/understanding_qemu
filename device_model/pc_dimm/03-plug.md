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

所以当设备初始化完后，还要添加到整个系统中。

# 计算插入位置

在实际插入设备前需要做一些检测工作，并且计算出该插入到哪里，是不是可一个插入。

这个工作由pc_memory_pre_plug()完成。

计算出合适的位置后，讲这个值保存在PCDIMMDevice.addr字段中。

# 插入MemoryRegion

准备工作基本都做完了，剩下的就是把backend的MemoryRegion插入到系统中。

这个任务就交给了pc_memory_plug()。
