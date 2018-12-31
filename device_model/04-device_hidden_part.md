上一节中看到设备实例化的流程实在是太简单了。然额，对于device设备类型，还有更深的套路。或者说对于很多设备类型，qemu中还隐藏这另一条实例化的脉络。

> 哥们儿玩得够开心啊

这个脉络就是设备的"realized"属性。

# 从device实例化函数开始

对于一个device设备，其初始化函数是device_initfn()。

```
static const TypeInfo device_type_info = {
    .name = TYPE_DEVICE,
    .parent = TYPE_OBJECT,
    .instance_size = sizeof(DeviceState),
    .instance_init = device_initfn,
    ...
};
```

当你仔细看这个函数的时候会发现，这个函数做了几件有意思的事情。其中之一就是给device设备添加了一个名为"realized"的属性。

```
    object_property_add_bool(obj, "realized",
                         device_get_realized, device_set_realized, NULL);
```

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

从截取的代码片段中可以看到，当属性被设置时会调用DeviceClass中的realize函数。而这个realized函数就是那个被隐藏了的实例化套路。

到这里我想大家一定会有两个问题。

  * 这个属性是什么时候设置的
  * 这个realize函数长什么样

那我们兵分两路，先来看看这个属性是什么时候设置的。

# 谁动了我的realized属性

这个东西还真不好找，而且可能存在多个设置属性的路径，我尝试用下面一个代码片段解释其中一个路径。

```
main()
  qemu_opts_foreach(qemu_find_opts("device"),device_init_func, NULL, NULL)
    qdev_device_add()
      dev = DEVICE(object_new(driver));
      object_property_set_bool(OBJECT(dev), true, "realized", &err);
```

这下明确了，当我们生成一个设备(object)后，就会调用方法来设置它的realized属性。

那至于第二个问题realize函数长什么样子，还是听我下回分解～
