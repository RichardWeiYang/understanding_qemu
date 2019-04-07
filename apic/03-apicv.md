在qemu/kernel混合模拟的基础上，为了进一步优化中断模拟，硬件提出了APICV。

# 相关的VM-execution controls

The following are the VM-execution controls relevant to APIC virtualization and virtual interrupts (see Section 24.6 for information about the locations of these controls):
* **Virtual-interrupt delivery**. This controls enables the evaluation and delivery of pending virtual interrupts (Section 29.2). It also enables the emulation of writes (memory-mapped or MSR-based, as enabled) to the APIC registers that control interrupt prioritization.
* **Use TPR shadow**. This control enables emulation of accesses to the APIC’s task-priority register (TPR) via CR8 (Section 29.3) and, if enabled, via the memory-mapped or MSR-based interfaces.
* **Virtualize APIC accesses**. This control enables virtualization of memory-mapped accesses to the APIC (Section 29.4) by causing VM exits on accesses to a VMM-specified APIC-access page. Some of the other controls, if set, may cause some of these accesses to be emulated rather than causing VM exits.
* **Virtualize x2APIC mode**. This control enables virtualization of MSR-based accesses to the APIC (Section 29.5).
* **APIC-register virtualization**. This control allows memory-mapped and MSR-based reads of most APIC registers (as enabled) by satisfying them from the virtual-APIC page. It directs memory-mapped writes to the APIC-access page to the virtual-APIC page, following them by VM exits for VMM emulation.
* **Process posted interrupts**. This control allows software to post virtual interrupts in a data structure and send a notification to another logical processor; upon receipt of the notification, the target processor will process the posted interrupts by copying them into the virtual-APIC page (Section 29.6).

# 相关的寄存器

* **APIC-access address (64 bits)**. This field contains the physical address of the 4-KByte APIC-access page
* **Virtual-APIC address (64 bits)**. This field contains the physical address of the 4-KByte virtual-APIC page

Certain VM-execution controls enable the processor to virtualize certain accesses to the APIC-access page without a VM exit. In general, this virtualization causes these accesses to be made to the virtual-APIC page instead of the APIC-access page.

* **TPR threshold (32 bits)**. Bits 3:0 of this field determine the threshold below which bits 7:4 of VTPR (see Section 29.1.1) cannot fall.
* **EOI-exit bitmap (4 fields; 64 bits each)**. These fields are supported only on processors that support the 1- setting of the “virtual-interrupt delivery” VM-execution control. They are used to determine which virtualized writes to the APIC’s EOI register cause VM exits


* **Posted-interrupt notification vector (16 bits)**. Its low 8 bits contain the interrupt vector that is used to notify a logical processor that virtual interrupts have been posted. See Section 29.6 for more information on the use of this field.
* **Posted-interrupt descriptor address (64 bits)**. 
