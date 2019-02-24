FW_CFG的一个用户就是linux中的qemu_fw_cfg模块了。

这个模块很简单，只有一个文件drivers/firmware/qemu_fw_cfg.c。其主要工作就是读取fw_cfg中的entry，并给每个entry创建一个sysfs。这样最终用户就可以通过sysfs来获取相关信息了。

# sysfs的结构

先来看一下模块加载后sysfs的样子。

一共有两个子目录：

  * 按照id区分的
  * 按照名字区分的

每个文件下面有

  * name
  * size
  * raw

前两个文件的含义比较明显，raw就是直接能读到fw的内容。

```
  /sys/firmware/qemu_fw_cfg/
      |
      +--rev
      |
      +--by_id
      |      |
      |      +-- 0
      |      |   |
      |      |   +--name
      |      |   |
      |      |   +--size
      |      |   |
      |      |   +--raw
      |      |
      |      +-- 1
      |          |
      |          +--name
      |          |
      |          +--size
      |          |
      |          +--raw
      |
      +--by_name
```

好了，我能想到的基本就这些了。也有可能我没有看到再上层的用户是如何利用sysfs来操作fw的。
