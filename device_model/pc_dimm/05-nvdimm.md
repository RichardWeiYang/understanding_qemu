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

                acpi_add_rom_blob(build_state, tables.table_data,
                                  "etc/acpi/tables", 0x200000);
                acpi_add_rom_blob(build_state, tables.linker->cmd_blob,
                                  "etc/table-loader", 0);
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

比如我们就看nvdimm中创建的NFIT表，那么再执行

```
# iasl NFIT.dat
```

就能解析成可阅读的文件了。在我的虚拟机上显示如下，而且正好就是spa, memdev, dcr三个部分。

```
[000h 0000   4]                    Signature : "NFIT"    [NVDIMM Firmware Interface Table]
[004h 0004   4]                 Table Length : 000000E0
[008h 0008   1]                     Revision : 01
[009h 0009   1]                     Checksum : BF
[00Ah 0010   6]                       Oem ID : "BOCHS "
[010h 0016   8]                 Oem Table ID : "BXPCNFIT"
[018h 0024   4]                 Oem Revision : 00000001
[01Ch 0028   4]              Asl Compiler ID : "BXPC"
[020h 0032   4]        Asl Compiler Revision : 00000001

[024h 0036   4]                     Reserved : 00000000

[028h 0040   2]                Subtable Type : 0000 [System Physical Address Range]
[02Ah 0042   2]                       Length : 0038

[02Ch 0044   2]                  Range Index : 0002
[02Eh 0046   2]        Flags (decoded below) : 0003
                   Add/Online Operation Only : 1
                      Proximity Domain Valid : 1
[030h 0048   4]                     Reserved : 00000000
[034h 0052   4]             Proximity Domain : 00000000
[038h 0056  16]           Address Range GUID : 66F0D379-B4F3-4074-AC43-0D3318B78CDB
[048h 0072   8]           Address Range Base : 00000001C0000000
[050h 0080   8]         Address Range Length : 0000000278000000
[058h 0088   8]         Memory Map Attribute : 0000000000008008

[060h 0096   2]                Subtable Type : 0001 [Memory Range Map]
[062h 0098   2]                       Length : 0030

[064h 0100   4]                Device Handle : 00000001
[068h 0104   2]                  Physical Id : 0000
[06Ah 0106   2]                    Region Id : 0000
[06Ch 0108   2]                  Range Index : 0002
[06Eh 0110   2]         Control Region Index : 0003
[070h 0112   8]                  Region Size : 0000000278000000
[078h 0120   8]                Region Offset : 0000000000000000
[080h 0128   8]          Address Region Base : 0000000000000000
[088h 0136   2]             Interleave Index : 0000
[08Ah 0138   2]              Interleave Ways : 0001
[08Ch 0140   2]                        Flags : 0000
                       Save to device failed : 0
                  Restore from device failed : 0
                       Platform flush failed : 0
                            Device not armed : 0
                      Health events observed : 0
                       Health events enabled : 0
                              Mapping failed : 0
[08Eh 0142   2]                     Reserved : 0000

[090h 0144   2]                Subtable Type : 0004 [NVDIMM Control Region]
[092h 0146   2]                       Length : 0050

[094h 0148   2]                 Region Index : 0003
[096h 0150   2]                    Vendor Id : 8086
[098h 0152   2]                    Device Id : 0001
[09Ah 0154   2]                  Revision Id : 0001
[09Ch 0156   2]          Subsystem Vendor Id : 0000
[09Eh 0158   2]          Subsystem Device Id : 0000
[0A0h 0160   2]        Subsystem Revision Id : 0000
[0A2h 0162   1]                 Valid Fields : 00
[0A3h 0163   1]       Manufacturing Location : 00
[0A4h 0164   2]           Manufacturing Date : 0000
[0A6h 0166   2]                     Reserved : 0000
[0A8h 0168   4]                Serial Number : 00123456
[0ACh 0172   2]                         Code : 0301
[0AEh 0174   2]                 Window Count : 0000
[0B0h 0176   8]                  Window Size : 0000000000000000
[0B8h 0184   8]               Command Offset : 0000000000000000
[0C0h 0192   8]                 Command Size : 0000000000000000
[0C8h 0200   8]                Status Offset : 0000000000000000
[0D0h 0208   8]                  Status Size : 0000000000000000
[0D8h 0216   2]                        Flags : 0000
                            Windows buffered : 0
[0DAh 0218   6]                    Reserved1 : 000000000000
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

## 模拟nvdimm的_DSM

好了，绕了这么大一圈，终于可以回到正题看看nvdimm的_DSM是如何模拟的。

所谓模拟，也就是在guest执行_DSM方法时截获该操作，由qemu进行模拟。

### guest内核

所以正式开始前，我们还得看一眼内核中执行_DSM时做了点什么。

下面的代码截取自acpi_evaluate_dsm()

```
	params[0].type = ACPI_TYPE_BUFFER;
	params[0].buffer.length = 16;
	params[0].buffer.pointer = (u8 *)guid;
	params[1].type = ACPI_TYPE_INTEGER;
	params[1].integer.value = rev;
	params[2].type = ACPI_TYPE_INTEGER;
	params[2].integer.value = func;
	if (argv4) {
		params[3] = *argv4;
	} else {
		params[3].type = ACPI_TYPE_PACKAGE;
		params[3].package.count = 0;
		params[3].package.elements = NULL;
	}

	ret = acpi_evaluate_object(handle, "_DSM", &input, &buf);
```

别的不管，主要看一共传进来了四个参数。比如有版本号和函数。这里先记着，留着后面再来看。

### AML

回到之前看得acpi表总的来说，SSDT表在地址空间上构建了这么两段：

```
                      0xA18                 MEMA
   +------------------+--+------------------+------+-------------+
   |                  |  |                  |      |             |
   +------------------+--+------------------+------+-------------+
                                          /          \
                                        /              \
                                        +--------------+
                                        |HDLE          |
                                        |REVS          |
                                        |FUNC          |
                                        |FARG          |
                                        +--------------+
```

其目的就是创建了一块共享内存，用来在guest和qemu之间传递模拟_DSM的参数和返回值。

比如我们可以在aml文件中看到

```
REVS = Arg1
FUNC = Arg2
```

而其中Arg1就是内核传递下来的参数rev，Arg2是内核传递参数的func。

### Qemu

当AML执行了 NFIT = Local6 的时候，接下去的操作就被Qemu截获了。而这个Local6参数的值是什么呢？

> Local6 = MEMA

所以，实际上就是告诉了Qemu一块共享内存的空间。这下是不是明白了？

那Qemu截获后是如何操作？这还要看nvdimm_dsm_ops了。因为是写操作，所以只看write方法。

其中有一个变量很引人注目：NvdimmDsmIn。

```
NvdimmDsmIn *in;

in->revision = le32_to_cpu(in->revision);
in->function = le32_to_cpu(in->function);
in->handle = le32_to_cpu(in->handle);
```

而这个变量类型定义为：

```
struct NvdimmDsmIn {
    uint32_t handle;
    uint32_t revision;
    uint32_t function;
    /* the remaining size in the page is used by arg3. */
    union {
        uint8_t arg3[4084];
    };
} QEMU_PACKED;
```

这下是不是和AML中共享内存的结构对应起来了？

好了，再详细的细节就和具体的函数实现相关了。留给大家自己去探索把～

# 最后的疑问

如果大家仔细看，比较qemu中aml的代码和guest中SSDT生成的文件，其中还有一个问题没有解决。

那就是 Local6 = MEMA 的这个MEMA的地址是如何得到的。

这个就牵扯到了另外两个超纲的知识点，将在[FW_CFG][2]章节中解开这个谜团。

[1]: https://gist.github.com/RichardWeiYang/aea8e71f5c9ff71499d19e77eb8a777e
[2]: /fw_cfg/00-qmeu_bios_guest.md
