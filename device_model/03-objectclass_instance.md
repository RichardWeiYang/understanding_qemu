有了设备类型，接着就要实例化设备了。这个理解起来就好像我们可以在一台机器上安装多个同种类的网卡。

实例化完成后会产生如下的对应关系：

```
  ObjectClass                    Object
  +---------------+              +----------------------+
  |               | <------------|class                 |
  |               |              |    (ObjectClass*)    |
  +---------------+              +----------------------+
```

这个过程由object_initialize()->object_initialize_with_type()实现。其过程实在有点简单：

  * 建立obj和ObjectClass之间的关联
  * 递归调用父类型的instance_init
  * 调用自己的instance_init

怎么样，是不是有点超级简单了？

> 太单纯了!
