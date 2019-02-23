看完了代码才发现有这个[规范文档][1]。不过也好，正好印证一下我对代码的理解。

按照我的理解，讲清楚两个东西就可以理解这个规范：

  * 接口
  * 数据结构

# 接口

在文档的开头就描述了两个东西：

  * Selector (Control) Register
  * Data Register

说白了就是有两个寄存器/通道，分别对应了控制面和数据面。

扯多了，直白的解释就是：

> 用户使用时，先通过Selector Register选定需要访问的单元，再通过Data Register来获取实际的数据。

好了，接口这部分其实就这么些，没多大花头。

在规范中还定义了这两个接口的地址。

```
  === x86, x86_64 Register Locations ===

  Selector Register IOport: 0x510
  Data Register IOport:     0x511
  DMA Address IOport:       0x514

  === ARM Register Locations ===

  Selector Register address: Base + 8 (2 bytes)
  Data Register address:     Base + 0 (8 bytes)
  DMA Address address:       Base + 16 (8 bytes)
```

我猜大家到这里一定感觉啥都不知道，不着急，等看到数据结构就清楚了。

# 数据结构

好了，这里要上一个巨大的数据结构了。嗯，其实呢和某些结构相比也不算大，主要是这里面有数组，所以显得大了。

```
                 FWCfgIoState
                 +------------------------------------------+
                 |machine_ready                             |
                 |   notify                                 | = fw_cfg_machine_ready
                 +------------------------------------------+
                 |dma_iomem                                 |
                 |   (MemoryRegion)                         |
                 |   +--------------------------------------+
                 |   |addr                                  |  = FW_CFG_IO_BASE + 4(0x514)
                 |   |name                                  |  = "fwcfg.dma"
                 |   |ops                                   |  fw_cfg_dma_mem_ops
                 |   |opaque                                |  FWCfgState itself
                 |   |size                                  |  = 8
                 +---+--------------------------------------+
                 |comb_iomem                                |
                 |   (MemoryRegion)                         |
                 |   +--------------------------------------+
                 |   |addr                                  |  = FW_CFG_IO_BASE(0x510)
                 |   |name                                  |  = "fwcfg"
                 |   |ops                                   |  fw_cfg_comb_mem_ops
                 |   |opaque                                |  FWCfgState itself
                 |   |size                                  |  = 0x02
                 +---+--------------------------------------+
                 |cur_entry                                 | fw_cfg_select(key)
                 |cur_offset                                |
                 |   (uint)                                 |
                 +------------------------------------------+
                 |file_slots                                |
                 |   (uint16)                               |
                 +------------------------------------------+
                 |files                                     |
                 |   (FWCfgFiles)                           |
                 |   +--------------------------------------+
                 |   |count                                 |
                 |   |   (uint32_t)                         |
                 |   +--------------------------------------+
                 |   |f[]                                   |                                                                                             etc/acpi/tables                                         etc/table-loader
                 |   |   (FWCfgFile)                        | ------------------------------------------------------------------------------------------->+------------------------------+----------------------->+------------------------------+
                 |   |   +----------------------------------+                                                                                             |name                          |                        |name                          |
                 |   |   |name                              |                                                                                             | (char [FW_CFG_MAX_FILE_PATH])|                        | (char [FW_CFG_MAX_FILE_PATH])|
                 |   |   |   (char [FW_CFG_MAX_FILE_PATH])  |                                                                                             |size                          |                        |size                          |
                 |   |   |size                              |                                                                                             |select                        |                        |select                        |
                 |   |   |select                            |                                                                                             |reserved                      |                        |reserved                      |
                 |   |   |reserved                          |                                                                                             +------------------------------+                        +------------------------------+
                 +---+---+----------------------------------+                                                                                                   ^                                                       ^
                 |entry_order                               | [FW_CFG_FILE_FIRST + file_slots]                                                                  |                                                       |
                 |   (int*)                                 |                                                                                                   |                                                       |
                 +------------------------------------------+                                                                                                   |                                                       |
                 |entries[0]                                | [FW_CFG_FILE_FIRST + file_slots]                                                                  |                                                       |
                 |   (FWCfgEntry*)                          |                                                                                                   |                                                       |
                 |                                          |                                                                                                   |                                                       |
                 |                                          |                                                                                                   v                                                       v
                 |                                          |     [FW_CFG_SIGNATURE]                         [FW_CFG_FILE_DIR]                            [FW_CFG_FILE_FIRST+1]                                   [FW_CFG_FILE_FIRST+2]
                 |                                          |     +-------------------+                      +-----------------+                          +------------------+                                    +------------------+
                 |                                          |     |data               | = "QEMU"             |data             | = s->files               |data              | = AcpiBuildTables.table_data       |data              | = AcpiBuildTables.linker->cmd_blob
                 |                                          |     |   (uint8_t *)     |                      |   (uint8_t *)   |                          |   (uint8_t *)    |                                    |   (uint8_t *)    |
                 |                                          |     |len                | = 4                  |len              | = size of s->files       |len               | = sizeof (table_data)              |len               | = sizeof (cmd_blob)
                 |                                          |     |   (uint32_t)      |            . . .     |   (uint32_t)    |                          |   (uint32_t)     |                                    |   (uint32_t)     |
                 |                                          |     |callback_opaque    | = NULL               |callback_opaque  | = NULL                   |callback_opaque   | = AcpiBuildState                   |callback_opaque   | = AcpiBuildState
                 |                                          |     |   (void*)         |                      |   (void*)       |                          |   (void*)        |                                    |   (void*)        |
                 |                                          |     |select_cb          | = NULL               |select_cb        | = NULL                   |select_cb         | = acpi_build_update                |select_cb         | = acpi_build_update
                 |                                          |     |write_cb           | = NULL               |write_cb         | = NULL                   |write_cb          | = NULL                             |write_cb          | = NULL
                 |                                          |     +-------------------+                      +-----------------+                          +------------------+                                    +------------------+
                 |                                          |
                 |                                          |     [FW_CFG_SIGNATURE...FW_CFG_FILE_DIR] static key                                         [FW_CFG_FILE_FIRST... ] related to f[]
                 |                                          |
                 |                                          |
                 +------------------------------------------+
                 |entries[1]                                | [FW_CFG_FILE_FIRST + file_slots]
                 |   (FWCfgEntry*)                          |
                 |   +--------------------------------------+
                 |   |data                                  |
                 |   |   (uint8_t *)                        |
                 |   |callback_opaque                       |
                 |   |   (void*)                            |
                 |   |select_cb                             |
                 |   |write_cb                              |
                 +---+--------------------------------------+
```

让偶来稍做解释，估计你可以看懂一些：

  * comb_iomem:  这个就是规范中定义的接口。看这个地址FW_CFG_IO_BASE就是0x510
  * entries:     这是整个数据的核心。fw_cfg用编号(key)作为关键字访问各种配置。而每个配置都有一个FWCfgEntry数据结构表示，并保存在entries数组上。
  * files:       fw_cfg的配置可以是简单的数据类型，如int, double。也可以是一个文件。所以在entries上可以看到分成了两端。从FW_CFG_FILE_FIRST开始保存的是文件信息。而且都有一个对应的FWCfgFile记录文件信息，并保存在files数组上。
  * cur_entry/offset:   因为接口简单，在每次访问配置之前先要指定访问的是哪个entry，且每次读写的位移取决于上一次的操作。所以这两个分别保存了当前的entry和偏移。

好了，其实展开就这么点内容了。除了长得丑，别的也没啥了。

# 相关代码

大的概念讲完了，还是要落到实际代码。

先看看这个高级货在哪里创建的。

```
pc_memory_init
    bochs_bios_init
        fw_cfg_init_io_dma
	    qdev_create(NULL, TYPE_FW_CFG_IO);
	    qdev_init_nofail(dev);
	        fw_cfg_io_realize
		    fw_cfg_common_realize
    rom_set_fw();
```

创建完了之后，分别有两大类添加配置的方式：

  * fw_cfg_add_bytes()
  * fw_cfg_add_file_callback()

前者就是添加普通配置的，而后者就是添加带有文件的配置的。

[1]: https://github.com/qemu/qemu/blob/master/docs/specs/fw_cfg.txt
