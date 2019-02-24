第二个用户是seabios，就现在了解，seabios所做的工作比linux上的qemu_fw_cfg做的工作要多些。

# 基础工作

所谓基础工作就是和linux qemu_fw_cfg模块所做的工作类似

> 读取fw_cfg相关的内容

说白了就是按照规范把文件的读写接口给实现了。

这些实现都在文件 src/fw/paravirt.c中。

  * qemu_cfg_select
  * qemu_cfg_read_file
  * qemu_cfg_write_file

等。

# linker/loader

接下来要讲的这个东西就有点意思了，说实话研究fw_cfg的目的就是想了解一下这个东西。

一切还得从qemu说起。

## Qemu的BIOSLinker

在Qemu的代码中有一个神奇的结构名字叫BIOSLinker。而且貌似它有一个作用是能够动态填写某些地址。

比如在[nvdimm][1]小节中留下的最后疑问，MEMA地址的动态绑定好像就是这个家伙完成的。

那究竟是怎么绑定的呢？ 开始我们的探索之旅吧～

先来看看这个结构的样子。

```
    AcpiBuildTables
    +-------------------------------------------+
    |table_data                                 |
    |     (GArray*)                             |
    |     +-------------------------------------+ ------------  facs
    |     |AcpiFacsDescriptorRev1               |
    |     |   signature                         | = "FACS"
    |     |                                     |               64bytes
    |     |                                     |
    |     |                                     |
    |     +-------------------------------------+ ------------  dsdt
    |     |AcpiTableHeader                      |
    |     |   signature                         | = "DSDT"
    |     |                                     |
    |     |                                     |
    |     |                                     |
    |     |                                     |
    |     +-------------------------------------+ ------------  
    |                                           |
    +-------------------------------------------+
    |linker                                     |
    |     (BIOSLinker*)                         |
    |     +-------------------------------------+
    |     |cmd_blob                             | list of BiosLinkerLoaderEntry
    |     |    (GArray*)                        |
    |     |    +--------------------------------+
    |     |    |command                         | = BIOS_LINKER_LOADER_COMMAND_ALLOCATE
    |     |    |   alloc.file                   | = "etc/acpi/tables"
    |     |    |   alloc.align                  |
    |     |    |   alloc.zone                   |
    |     |    |                                |
    |     |    +--------------------------------+
    |     |    |command                         | = BIOS_LINKER_LOADER_COMMAND_ALLOCATE
    |     |    |   alloc.file                   | = "etc/acpi/nvdimm-mem"
    |     |    |   alloc.align                  |
    |     |    |   alloc.zone                   |
    |     |    |                                |
    |     |    +--------------------------------+
    |     |    |command                         | = BIOS_LINKER_LOADER_COMMAND_ADD_CHECKSUM
    |     |    |   file                         | = "etc/acpi/tables"
    |     |    |   offset                       |
    |     |    |   start_offset                 |
    |     |    |                                |
    |     |    +--------------------------------+
    |     |    |command                         | = BIOS_LINKER_LOADER_COMMAND_ADD_POINTER
    |     |    |   dest_file                    | = "etc/acpi/tables"
    |     |    |   src_file                     | = "etc/acpi/nvdimm-mem"
    |     |    |   offset                       | = mem_addr_offset
    |     |    |   size                         | = 4
    |     |    +--------------------------------+
    |     |                                     |
    |     |file_list                            | list of BiosLinkerFileEntry
    |     |    (GArray*)                        |
    |     |    +--------------------------------+
    |     |    |name                            | = "etc/acpi/tables"
    |     |    |bolb                            | = table_data
    |     |    +--------------------------------+
    |     |    |name                            | = "etc/acpi/nvdimm-mem"
    |     |    |bolb                            | = AcpiNVDIMMState.dsm_mem
    |     |    +--------------------------------+
    |     |    |                                |
    |     |    |                                |
    +-----+----+--------------------------------+
```

在这里我把AcpiBuildTables这个结构也列出，以便看清结构中和acpi table的对应关系。

这个结构展开其中就两个重要的部分：

  * cmd_blob
  * file_list

file_list是一个“文件”列表，其实是这个结构以文件名字的方式组织了对应的内存。
cmd_blob则是定义了“命令”列表。

怎么样，看到命令是不是感觉有点意思了？

定义命令的结构体叫做BiosLinkerLoaderEntry，它太大了我们就不在这里展开。不过我们可以看看一共有几种命令。

```
enum {
    BIOS_LINKER_LOADER_COMMAND_ALLOCATE          = 0x1,
    BIOS_LINKER_LOADER_COMMAND_ADD_POINTER       = 0x2,
    BIOS_LINKER_LOADER_COMMAND_ADD_CHECKSUM      = 0x3,
    BIOS_LINKER_LOADER_COMMAND_WRITE_POINTER     = 0x4,
};
```

也就四种，不算太多。

## Seabios的romfile_loader_entry_s

Seabios中好像没有找到BIOSLinker对应的结构，但是能找到BiosLinkerLoaderEntry对应的是romfile_loader_entry_s。

而且可以看到Seabios中定义的命令类型是：

```
enum {
    ROMFILE_LOADER_COMMAND_ALLOCATE      = 0x1,
    ROMFILE_LOADER_COMMAND_ADD_POINTER   = 0x2,
    ROMFILE_LOADER_COMMAND_ADD_CHECKSUM  = 0x3,
    ROMFILE_LOADER_COMMAND_WRITE_POINTER = 0x4,
};
```

这样正好好qemu中的命令对上。

好了，到了这里我想你基本已经知道他们之间的关系以及是如何运作的了。

最后我只用提示一点，那就是[nvdimm][1]中那个存疑的MEMA地址是通过COMMAND_ADD_POINTER来实现的。

[1]: /device_model/pc_dimm/05-nvdimm.md
