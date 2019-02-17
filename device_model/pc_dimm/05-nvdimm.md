NVDIMM是PCDIMM的子类型，所以整体的流程基本差不多，着重讲一下几个不同之处。

# 全局

我们先来看一下全局上，一个nvdimm设备初始化并加入系统有哪些步骤。

```
main()
    qemu_opts_foreach(qemu_find_opts("device"), device_init_func, NULL, NULL)
        ...
        device_set_realized()
            hotplug_handler_pre_plug
                ...
                memory_device_pre_plug
                    nvdimm_prepare_memory_region                (1)

            if (dc->realize) {
              dc->realize(dev, &local_err);
            }

            hotplug_handler_plug
                ...
                nvdimm_plug
                    nvdimm_build_fit_buffer                     (2)
    qemu_run_machine_init_done_notifiers()
        pc_machine_done
            acpi_setup
                acpi_build
                    nvdimm_build_acpi                           (3)
                        nvdimm_build_ssdt
                        nvdimm_build_nfit
```

从上面的图中可以看到NVDIMM设备初始化和PCDIMM设备类似，也是有这么几个步骤

  * 插入准备
  * 实例化
  * 插入善后
  * 添加ACPI表

其中大部分的工作和PCDIMM一样，只有标注了1，2，3的三个地方需要额外的工作。

分别是：

  1. 添加nvdimm label区域
  2. 创建NFIT表的内容
  3. 创建SSDT和NFIT并加入acpi

# 分配label区域

按照当前的实现，使用nvdimm设备时，在分配空间的最后挖出一块做label。这个工作在nvdimm_prepare_memory_region中完成。

这块label区域在虚拟机内核中通过DSM方法来操作，后面我们会看到DSM方法是如何虚拟化的。

# 填写nfit信息

nfit表是用来描述nvdimm设备上地址空间组织形式的，所以这部分需要在计算好了dimm地址后进行，有nvdimm_build_fit_buffer完成。

展开这个函数我们可以看到

```
for (; device_list; device_list = device_list->next) {
    DeviceState *dev = device_list->data;

    /* build System Physical Address Range Structure. */
    nvdimm_build_structure_spa(structures, dev);

    /*
     * build Memory Device to System Physical Address Range Mapping
     * Structure.
     */
    nvdimm_build_structure_memdev(structures, dev);

    /* build NVDIMM Control Region Structure. */
    nvdimm_build_structure_dcr(structures, dev);
}
```

也就是分别构建了 SPA, MEMDEV和DCR三个信息。而这些信息就在内核函数add_table中处理。

不看具体内容，我们只看add_table中类型的定义：

```
enum acpi_nfit_type {
	ACPI_NFIT_TYPE_SYSTEM_ADDRESS = 0,
	ACPI_NFIT_TYPE_MEMORY_MAP = 1,
	ACPI_NFIT_TYPE_INTERLEAVE = 2,
	ACPI_NFIT_TYPE_SMBIOS = 3,
	ACPI_NFIT_TYPE_CONTROL_REGION = 4,
	ACPI_NFIT_TYPE_DATA_REGION = 5,
	ACPI_NFIT_TYPE_FLUSH_ADDRESS = 6,
	ACPI_NFIT_TYPE_CAPABILITIES = 7,
	ACPI_NFIT_TYPE_RESERVED = 8	/* 8 and greater are reserved */
};
```

上面三个类型对应了这个枚举类型的0，1，4。

# 构建acpi

最终nvdimm的信息要填写到acpi中，一共有两张表：SSDT和NFIT。这部分工作在nvdimm_build_acpi中完成。

其中NFIT表在nvdimm_build_nfit函数中完成，其实没有太多内容。就是将上面创建好的表内容拷贝过来。

而SSDT这张表就讲究多了，这里面涉及了acpi的一些概念。

## ACPI Table

在进入代码之前，还是要先了解一下acpi的一些概念。这里并不是系统性介绍acpi，只是补充一些代码阅读中涉及的概念。

ACPI(Advanced Configuration and Power Management Interface)，简单理解就是用来描述硬件配置的。

为了做好管理，acpi当然提供了很多复杂的功能，其中我们关注的就是acpi table。而这些表的作用比较直观，那就是按照一定的规范来描述硬件设备的属性。

规范中定义了很多表，而这些表按照我的理解可以分成两类：

  * DSDT & SSDT
  * 其他

DSDT & SSDT 虽然也叫表，但是他们使用AML语言编写的，可以被解析执行的。也就是这两张表是可编程的。
而其他的表则是静态的，按照标准写好的数据结构。

## 查看Table

在linux上可以很方便得查看acpi table。因为他们都在sysfs上。

> /sys/firmware/acpi/tables/

文件的内容还需要使用工具查看。

```
# acpidump -b
```

这个命令将会把所有的acpi table的二进制形式导出到.dat文件，并保存在当前目录下。

比如你想查看FACS表，那么再执行

```
# iasl facs.dat
```

就能解析成可阅读的文件了。在我的虚拟机上显示如下。

```
[000h 0000   4]                    Signature : "FACS"
[004h 0004   4]                       Length : 00000040
[008h 0008   4]           Hardware Signature : 00000000
[00Ch 0012   4]    32 Firmware Waking Vector : 00000000
[010h 0016   4]                  Global Lock : 00000000
[014h 0020   4]        Flags (decoded below) : 00000000
                      S4BIOS Support Present : 0
                  64-bit Wake Supported (V2) : 0
[018h 0024   8]    64 Firmware Waking Vector : 0000000000000000
[020h 0032   1]                      Version : 00
[021h 0033   3]                     Reserved : 000000
[024h 0036   4]    OspmFlags (decoded below) : 00000000
               64-bit Wake Env Required (V2) : 0
```

## AML & ASL

开始我们也说了，acpi table分成两类。刚才的facs属于第二类，也就是静态的数据结构。

而对于第一种表，他们则复杂得多，是用AML编写的。

  * AML: ACPI Machine Language
  * ASL: ACPI source language

所以前者相当于机器语言，而后者相当于汇编。

解析这两张表的方式和刚才的一样也是通过iasl命令，但是样子就完全不一样了。我们就直接拿nvdimm构造ssdt来看。

[NVDIMM SSDT][1]

刚开始看确实有点头大，不过慢慢就好了。我稍微来解释一下。

**定义函数**

```
Method (NCAL, 5, Serialized)
```

所以看到这个表中定义了好些函数： _DSM, RFIT, _FIT 。

值得注意的是以_ 开头的函数是在规范中定义的，而其他的则是可以自己添加的。

**定义变量**

如果说函数定义还比较好理解，那么变量的定义就有点不那么直观了。

最简单的方式是赋值：

```
  Local6 = MEMA
```

当然这个变量Local6是自定义的，只要直接使用就好。

另外一种就是比较特殊的，我称之为**指针方式**。因为看着和c语言的指针访问很像。而且这种方式需要做两步。

  * 定义Region
  * 定义Field

先定义出一块空间，里面包含了这块空间对应的起始地址和长度。

```
OperationRegion (NPIO, SystemIO, 0x0A18, 0x04)
```

然后定义这块空间的field，这点上看和c语言的结构体很像。

```
Field (NPIO, DWordAcc, NoLock, Preserve)
{
    NTFI,   32
}
```

这样相当于得到了一个指针变量NFIT，这个地址是指向0x0A18的4个字节。

最后访问这个区域就是直接用赋值语句：

```
NTFI = Local6
```

就相当于往NTFI这个指针写了Local6的值。

[1]: https://gist.github.com/RichardWeiYang/aea8e71f5c9ff71499d19e77eb8a777e
