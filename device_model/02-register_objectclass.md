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
