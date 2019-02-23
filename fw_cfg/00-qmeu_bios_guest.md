FW_CFG 提供了guest一种获得额外信息的方式，好像我也很难讲清楚，直接上原文

> This hardware interface allows the guest to retrieve various data items
(blobs) that can influence how the firmware configures itself, or may
contain tables to be installed for the guest OS. Examples include device
boot order, ACPI and SMBIOS tables, virtual machine UUID, SMP and NUMA
information, kernel/initrd images for direct (Linux) kernel booting, etc.

在Qemu代码中有对FW_CFG的[规范][1]，大家可以自行前往。

这一章，我们就来解读一下这个东西的作用，以及它的用户们。

所以首先我们来解读一下这个规范，看看这个东西的样子以及工作的原理。

[规范解读][2]

接着我们来看看都有谁，会怎么用这个FW_CFG。

现在我能看到的分别有两个使用者：

  * [linux 虚拟机][3]

  * [SeaBios][4]

[1]: https://github.com/qemu/qemu/blob/master/docs/specs/fw_cfg.txt
[2]: /fw_cfg/01-spec.md
[3]: /fw_cfg/02-linux_guest.md
[4]: /fw_cfg/03-seabios.md
