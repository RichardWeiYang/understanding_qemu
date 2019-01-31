# 一一对应

注册类型TypeImpl之后就需要初始化该类型，如果打开TypeImpl这个结构体，可以看到其中有个叫class的成员。初始化其实就是初始化的它。

```
    TypeImpl                               ObjectClass
    +----------------------+               +----------------------+
    |class                 |<------------->|type                  |
    |     (ObjectClass*)   |               |     (TypeImpl *)     |
    +----------------------+               +----------------------+
```

进行初始化的具体函数是 type_initialize()。

摘取其中主要的流程如下：

```
  type_initialize()
    ...
    parent = type_get_parent(ti);
    if (parent) {
      type_initialize(parent);
      ...
      for (i = 0; i < ti->num_interfaces; i++) {
        TypeImpl *t = type_get_by_name(ti->interfaces[i].typename);
        ...
        type_initialize_interface(ti, t, t);
      }
    }

    ...

    while (parent) {
      if (parent->class_base_init) {
        parent->class_base_init(ti->class, ti->class_data);
      }
      parent = type_get_parent(parent);
    }

    if (ti->class_init) {
      ti->class_init(ti->class, ti->class_data);
    }
```

其中主要做了几件事：

  * 设置类型的大小并创建该类型
  * 设置类型实例的大小，为实例化设备做准备
  * 初始化父设备类型，如果没有的话
  * 初始化接口类型
  * 调用父设备类型的class_base_init初始化自己
  * 调用class_init初始化自己

在这里看不出什么具体的东西，因为每种设备类型将执行不同的初始化函数。

但是有一点可以看出的是，qemu设备类型有一个树形结构。或者说是一种面向对象的编程模型，需要初始化父类后，再初始化自己。

# 接口类

在看了一段时间代码后，发现一个类型还能定义其相关的接口类型。

比如我们看e1000设备的定义可以看到在最后有定义一个interfaces成员。

```
#define TYPE_E1000_BASE "e1000-base"

static const TypeInfo e1000_base_info = {
    .name          = TYPE_E1000_BASE,
    .parent        = TYPE_PCI_DEVICE,
    ...
    .interfaces = (InterfaceInfo[]) {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    },
};
```

那么我们来看看这个成员如何初始化，如何使用。

在 type_initialize()函数中有一段

```
for (i = 0; i < ti->num_interfaces; i++) {
  TypeImpl *t = type_get_by_name(ti->interfaces[i].typename);
  ...
  type_initialize_interface(ti, t, t);
}
```

这个就是在初始化ti这个类型的接口类。好在这个函数type_initialize_interface()不长，我们直接打开看看。

```
static void type_initialize_interface(TypeImpl *ti, TypeImpl *interface_type,
                                      TypeImpl *parent_type)
{
    InterfaceClass *new_iface;
    TypeInfo info = { };
    TypeImpl *iface_impl;

    info.parent = parent_type->name;
    info.name = g_strdup_printf("%s::%s", ti->name, interface_type->name);
    info.abstract = true;

    iface_impl = type_new(&info);
    iface_impl->parent_type = parent_type;
    type_initialize(iface_impl);
    g_free((char *)info.name);

    new_iface = (InterfaceClass *)iface_impl->class;
    new_iface->concrete_class = ti->class;
    new_iface->interface_type = interface_type;

    ti->class->interfaces = g_slist_append(ti->class->interfaces,
                                           iface_impl->class);
}
```

说实话，这个东西有点绕。因为在这个过程中又注册了一个新的类型并做了初始化。

```
                                                  TypeImpl
                                                  +------------------------------------+
                                                  |name                                |
                                                  |   "conventional-pci-device"        |
                                                  |class (struct InterfaceClass)       |
                                                  |                                    |
                                                  +------------------------------------+
                                                                      ^
                                                                      |
                                                                      |
    TypeImpl                                      TypeImpl            |
                                                  +------------------------------------+
    +--------------------------+                  |parent          ---+                |
    |name                      |                  |name                                |
    |   "e1000-base"           |                  |   "e1000::conventional-pci-device" |
    |class (E1000BaseClass)    |     +----------->|class (struct InterfaceClass)       |
    |   ^  interfaces     -----|-----+            |       concrete_class  ----+        |
    |   |                      |                  +------------------------------------+
    +--------------------------+                                              |
        |                                                                     |
        +---------------------------------------------------------------------+
```

最后的结果我尝试用图来展示。

  * 创建了一个INTERFACE_CONVENTIONAL_PCI_DEVICE类型的子类，也是一个接口类型
  * E1000BaseClass中的interfaces会指向新建的接口类
  * 而接口类中的concrete_class会指向E1000BaseClass
