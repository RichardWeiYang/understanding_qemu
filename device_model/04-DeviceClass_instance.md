上节描述的设备模型抽象了的实例化流程，正是因为是一个抽象的实例化过程，虽然看着很简单，但也失去了很多真实场景下丰富的细节。

那现在我们来看看DeviceClass的实例化。因为很多具体的设备的父类就是这个DeviceClass，所以对这个类型实例化过程的了解对理解Qemu中大部分设备的初始化有很重要的帮助。

```
      +--------------------------+                           +----------------------+
      |                          |                           |                      |
      |   ObjectClass            | <-------------------------|    Object            |
      |     class_init           |                           |    instance_init     |
      |                          |                           |(object_instance_init)|
      +--------------------------+                           +----------------------+
                |                                                      |
                |                                                      |
                |                                                      |
                v                                                      v
      +--------------------------+                           +----------------------+
      |                          |                           |                      |
      |   DeviceClass            |  <---------------------   |   DeviceState        |
      |     class_init           |                           |      instance_init   |
      |     (device_class_init)  |                           |      (device_initfn) |
      |                          |                           |                      |
      |     realize              |  overwrite by child class |                      |
      |     unrealize            |                           |                      |
      +--------------------------+                           +----------------------+
```

这张图显示了DeviceClass相关的父类和对应的对象之间的关系。而我们现在关注的就是它的实例化函数**device_initfn**。

# device_initfn

这个函数并不长，看着也很简单。

```
static void device_initfn(Object *obj)
{
    DeviceState *dev = DEVICE(obj);
    ObjectClass *class;
    Property *prop;

    if (qdev_hotplug) {
        dev->hotplugged = 1;
        qdev_hot_added = true;
    }

    dev->instance_id_alias = -1;
    dev->realized = false;

    object_property_add_bool(obj, "realized",
                             device_get_realized, device_set_realized, NULL);
    object_property_add_bool(obj, "hotpluggable",
                             device_get_hotpluggable, NULL, NULL);
    object_property_add_bool(obj, "hotplugged",
                             device_get_hotplugged, NULL,
                             &error_abort);

    class = object_get_class(OBJECT(dev));
    do {
        for (prop = DEVICE_CLASS(class)->props; prop && prop->name; prop++) {
            qdev_property_add_legacy(dev, prop, &error_abort);
            qdev_property_add_static(dev, prop, &error_abort);
        }
        class = object_class_get_parent(class);
    } while (class != object_class_by_name(TYPE_DEVICE));

    object_property_add_link(OBJECT(dev), "parent_bus", TYPE_BUS,
                             (Object **)&dev->parent_bus, NULL, 0,
                             &error_abort);
    QLIST_INIT(&dev->gpios);
}
```

就我所知，这个函数主要做了两件事：

  * 给设备设置了三个共有的属性: realized, hotpluggable, hotplugged
  * 根据每个类型定义时的props字段，设置各自的属性

# 设备属性的设置

先来讲讲这个属性的设置是设置的什么。

当我们运行Qemu的时候，通常会在命令行写上一串参数来表示虚拟机的硬件配置。或者我们通过Qemu monitor添加设备时输入的硬件参数。

比如：

```
object_add memory-backend-ram,id=ram0,size=1G
device_add pc-dimm,id=dimm0,memdev=ram0,node=0
```

我们就看pc-dimm设备，其中有参数memdev=ram0。

从命令行上，这个参数的作用其实是关联pc-dimm和memory-backend-ram。那在代码中是如何做的呢？

先来看pc-dimm设备的属性props定义：

```
static Property pc_dimm_properties[] = {
    ...
    DEFINE_PROP_LINK(PC_DIMM_MEMDEV_PROP, PCDIMMDevice, hostmem,
                     TYPE_MEMORY_BACKEND, HostMemoryBackend *),
    ...
};
```

再回过去看device_initfn中那个对props的循环。在这里就关联了pc-dimm和memory-backend。

# realized属性的功效

那接着看看这个属性的被设置时都发生了些什么。

```
static void device_set_realized(Object *obj, bool value, Error **errp)
{
    DeviceState *dev = DEVICE(obj);
    DeviceClass *dc = DEVICE_GET_CLASS(dev);

    ...

    if (dc->realize) {
        dc->realize(dev, &local_err);
    }

    ...

}
```

从截取的代码片段中可以看到，当属性被设置时会调用DeviceClass中的realize函数。而这个realized函数就是那个被隐藏了的实例化套路。而很多设备初始化的细节都隐藏在realized函数中。

**注**：你再往细了看，device_set_realized函数本身也隐藏着很多实现的细节，这里我们就不展开了。

到这里我想大家一定会有两个问题。

  * 这个属性是什么时候设置的
  * 这个realize函数长什么样

那我们兵分两路，先来看看这个属性是什么时候设置的。

## 谁动了我的realized属性

这个东西还真不好找，而且可能存在多个设置属性的路径，我尝试用下面一个代码片段解释其中一个路径。

```
main()
  qemu_opts_foreach(qemu_find_opts("device"),device_init_func, NULL, NULL)
    qdev_device_add()
      dev = DEVICE(object_new(driver));
      object_property_set_bool(OBJECT(dev), true, "realized", &err);
```

这下明确了，当我们生成一个设备对象(object)后，就会调用方法来设置它的realized属性。

也就是我们会**人为**得去触发这个事件。

## 一个真实的realize函数

既然刚才讲到了pc-dimm，那我们就来看看pc-dimm的realize。

通常这个函数在class_init中设置，所以我们看到在pc_dimm_class_init中realize被设置成了pc_dimm_realize。

打开这个函数pc_dimm_realize可以看到它会进一步调用子类PCDIMMDeviceClass的realize。

这样就形成了一个类似面向对象的初始化过程。

```
    +--------------------------+
    |                          |
    |   DeviceClass            |
    |     realize              |    pc_dimm_realize()
    +--------------------------+
              |                 
              |                 
              |                 
              v                 
    +--------------------------+
    |                          |
    |   PCDIMMDeviceClass      |
    |     realize              |    who? leave it a question here :-)
    +--------------------------+
```
