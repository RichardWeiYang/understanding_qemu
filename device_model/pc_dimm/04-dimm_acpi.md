在x86带有acpi支持的系统上，如果去要加入内存numa的支持，除了上述的步骤之外好要添加相应的acpi表。这样才能在内核中获取相关的numa信息。

# 内核需要什么acpi信息

在看Qemu中干了什么之前我们先看看内核中是需要什么acpi的信息。

```
int __init acpi_numa_init(void)
{
	int cnt = 0;

	if (acpi_disabled)
		return -EINVAL;

	/*
	 * Should not limit number with cpu num that is from NR_CPUS or nr_cpus=
	 * SRAT cpu entries could have different order with that in MADT.
	 * So go over all cpu entries in SRAT to get apicid to node mapping.
	 */

	/* SRAT: System Resource Affinity Table */
	if (!acpi_table_parse(ACPI_SIG_SRAT, acpi_parse_srat)) {
		struct acpi_subtable_proc srat_proc[3];

    ...
		cnt = acpi_table_parse_srat(ACPI_SRAT_TYPE_MEMORY_AFFINITY,
					    acpi_parse_memory_affinity, 0);
	}

	/* SLIT: System Locality Information Table */
	acpi_table_parse(ACPI_SIG_SLIT, acpi_parse_slit);

  ...
	return 0;
}
```

简化后的代码如上，主要就是解析了两张表：

  * SRAT: System Resource Affinity Table
  * SLIT: System Locality Information Table

你要问我这是啥我也不懂，反正就是他们俩没错了～

# Qemu中模拟ACPI表

这个工作在下面的函数中执行。

```
acpi_build
  if (pcms->numa_nodes) {
    acpi_add_table(table_offsets, tables_blob);
    build_srat(tables_blob, tables->linker, machine);
    if (have_numa_distance) {
        acpi_add_table(table_offsets, tables_blob);
        build_slit(tables_blob, tables->linker);
    }
  }
```

你看，是不是qemu也是创建了两张表？

* 至于这个acpi_build是什么时候调用的，请看下一节。

那这个表是不是就是内核中解析的表呢？我们来看看两个数据结构是不是能够对上就好。

Qemu中有 AcpiSratMemoryAffinity

```
struct AcpiSratMemoryAffinity {
    ACPI_SUB_HEADER_DEF
    uint32_t    proximity;
    uint16_t    reserved1;
    uint64_t    base_addr;
    uint64_t    range_length;
    uint32_t    reserved2;
    uint32_t    flags;
    uint32_t    reserved3[2];
} QEMU_PACKED;
```

而内核中有 acpi_srat_mem_affinity

```
struct acpi_srat_mem_affinity {
	struct acpi_subtable_header header;
	u32 proximity_domain;
	u16 reserved;		/* Reserved, must be zero */
	u64 base_address;
	u64 length;
	u32 reserved1;
	u32 flags;
	u64 reserved2;		/* Reserved, must be zero */
};
```

这两个正好一摸一样。这下放心了。

# 附带一个ACPI System Description Tables

多次需要对ACPI Table进行操作，总是要去找什么表在什么地方。很费时间，干脆做个总结。没有单独开一章节，就先放在这里了。

下图的内容在ACPI SPEC 5.2 ACPI System Description Tables中可以找到细节。

```
     RSDP                      
     +--------------+               RSDT / XSDT
     |RSDT Addr  ---|-------+------>+--------------+
     |              |       |       |Head          |
     +--------------+       |       |              |
     |XSDT Addr  ---|-------+       |              |
     |              |               |              |             
     +--------------+         Entry +--------------+           FADT
                                    |FADT Addr  ---|---------->+--------------+
                                    |              |           |Head          |
                                    +--------------+           |              |
                                    |              |           |              |
                                    |              |           |              |
                                    +--------------+           +--------------+
                                    |MADT Addr     |           |FACS Addr     |
                                    |              |           |              |
                                    +--------------+           +--------------+
                                    |MCFG Addr     |           |DSDT Addr     |
                                    |              |           |              |
                                    +--------------+           +--------------+
                                    |              |           |X_DSDT Addr   |
                                    |              |           |              |
                                    +--------------+           +--------------+
                                    |SSDT Addr     |                       
                                    |              |                       
                                    +--------------+                                                         
```
