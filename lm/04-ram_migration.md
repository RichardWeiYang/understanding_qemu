内存是整个虚拟机中重要的部分，也是迁移过程中迁移量最大的部分。

首先我们将从迁移框架的流程上看看整个过程都有哪些函数参与其中，了解内存迁移的流程结构。

接着我们要从数据结构层面看看我们在这些流程中究竟是要处理哪些数据。

# 内存迁移流程

要说流程，还是要先回到总体架构那一节中的总体流程。

```
migration_thread()

  qemu_savevm_state_header
      qemu_put_be32(f, QEMU_VM_FILE_MAGIC);
      qemu_put_be32(f, QEMU_VM_FILE_VERSION);
      qemu_put_be32(f, QEMU_VM_CONFIGURATION);
      vmstate_save_state(f, &vmstate_configuration, &savevm_state, 0);
  qemu_savevm_send_open_return_path(s->to_dst_file);
  qemu_savevm_send_ping(s->to_dst_file, 1);
      qemu_savevm_command_send(f, MIG_CMD_PING, , (uint8_t *)&buf)

  ; iterate savevm_state and call save_setup
  qemu_savevm_state_setup(s->to_dst_file);
      save_section_header(f, se, QEMU_VM_SECTION_START)
      se->ops->save_setup(f, se->opaque)
      save_section_footer(f, se)
      precopy_notify(PRECOPY_NOTIFY_SETUP, &local_err)

  migrate_set_state(&s->state, MIGRATION_STATUS_SETUP, MIGRATION_STATUS_ACTIVE);
  migration_iteration_run

      ; iterate savevm_state and call save_live_pending
      qemu_savevm_state_pending(pend_pre/compat/post)
          se->ops->save_live_pending()

      ; iterate savevm_state and call save_live_iterate
      qemu_savevm_state_iterate()
          save_section_header(f, se, QEMU_VM_SECTION_PART)
          se->ops->save_live_iterate(f, se->opaque)
          save_section_footer(f, se)
      migration_completion()
          qemu_system_wakeup_request(QEMU_WAKEUP_REASON_OTHER, NULL);
          vm_stop_force_state(RUN_STATE_FINISH_MIGRATE)

          qemu_savevm_state_complete_precopy(s->to_dst_file, false, inactivate);
              ; iterate savevm_state and call save_live_complete_precopy
              cpu_synchronize_all_states();
              save_section_header(f, se, QEMU_VM_SECTION_END);
              se->ops->save_live_complete_precopy(f, se->opaque)
              save_section_footer(f, se);

              ; iterate savevm_state and call vmstate_save
              save_section_header(f, se, QEMU_VM_SECTION_FULL);
              vmstate_save(f, se, vmdesc)
              save_section_footer(f, se);

  migration_detect_error
  migration_update_counters
  migration_iteration_finish
```

在这个基础上我们进一步抽取和简化，能看到整个过程中主要是这么几个回调函数在起作用。

  * se->ops->save_setup
  * se->ops->save_live_pending
  * se->ops->save_live_iterate
  * se->ops->save_live_complete_precopy

那对应到内存，这个se就是savevm_ram_handlers，其对应的函数们就是

  * ram_save_setup
  * ram_save_pending
  * ram_save_iterate
  * ram_save_complete

虽然里面做了很多事儿，也有很多细节上的优化，不过总的流程可以总结成两句话：

  * 用bitmap跟踪脏页
  * 将脏页传送到对端

# 相关数据结构

了解了上述内存迁移的总体流程后，我们可以猜到在内存迁移过程中重要的数据结构就是跟踪脏页的bitmap了。

其中一共用到了两个bitmap：

  * RAMBlock->bmap
  * ram_list.dirty_memory[DIRTY_MEMORY_MIGRATION]

如果要画一个图来解释的画，那么这个可能会像一点。

```
    ram_list.dirty_memory[]
    +----------------------+---------------+--------------+----------------+
    |                      |               |              |                |
    +----------------------+---------------+--------------+----------------+
    ^                      ^               ^              ^
    |                      |               |              |
    RAMBlock.bmap                          RAMBlock.bmap
    +----------------------+               +--------------+
    |                      |               |              |
    +----------------------+               +--------------+
```

ram_list.dirty_memroy[]是一整个虚拟机地址空间的bitmap。虽然虚拟机的地址空间可能有空洞，但是这个bitmap是连续的。
RAMBlock.bmap是每个RAMBlock表示的地址空间的bitmap。

所以同步的时候分成两步：

  * 虚拟机对内存发生读写时会更新ram_list.dirty_memroy[]
  * 每次内存迁移迭代开始，将ram_list.dirty_memory[]更新到RAMBlock.bmap

这样两个bitmap各司其职可以同步进行。
