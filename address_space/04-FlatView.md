FlatView，从字面上看意思就是 扁平视图。那是谁的偏平视图呢？

从大了说是地址空间的偏平视图，但是仔细想想它是对应MemoryRegion的。因为MemoryRegion是一颗树，那相应的就可以有一个偏平空间。

说到这儿，其实已经没有什么花头了。不就是一个偏平视图么？但是为了充数，还是要展开讲讲的。突然想到了《三体》世界中的低纬展开，这个MemoryRegion到FlatView的过程还真有点像。

这个过程在qemu中分成了两步：

  * generate_memory_topology()
  * address_space_set_flatview()

前者的工作是将MemoryRegion展开成一维，而后者是将AddressSpace和FlatView进行关联。

因为确实没啥多说的，还是用张图来表示AddressSpace和FlatView之间的关系。

```
    AddressSpace               
    +-------------------------+
    |name                     |
    |   (char *)              |          FlatView (An array of FlatRange)
    +-------------------------+          +----------------------+
    |current_map              | -------->|nr                    |
    |   (FlatView *)          |          |nr_allocated          |
    +-------------------------+          |   (unsigned)         |         FlatRange             FlatRange
    |                         |          +----------------------+         
    |                         |          |ranges                | ------> +---------------------+---------------------+
    |                         |          |   (FlatRange *)      |         |offset_in_region     |offset_in_region     |
    |                         |          +----------------------+         |    (hwaddr)         |    (hwaddr)         |
    |                         |                                           +---------------------+---------------------+
    |                         |                                           |addr(AddrRange)      |addr(AddrRange)      |
    |                         |                                           |    +----------------|    +----------------+
    |                         |                                           |    |start (Int128)  |    |start (Int128)  |
    |                         |                                           |    |size  (Int128)  |    |size  (Int128)  |
    |                         |                                           +----+----------------+----+----------------+
    |                         |                                           |mr                   |mr                   |
    |                         |                                           | (MemoryRegion *)    | (MemoryRegion *)    |
    |                         |                                           +---------------------+---------------------+
    |                         |
    |                         |
    |                         |
    |                         |          MemoryRegion(system_memory/system_io)
    +-------------------------+          +----------------------+
    |root                     |          |                      | root of a MemoryRegion
    |   (MemoryRegion *)      | -------->|                      | tree
    +-------------------------+          +----------------------+
```

在AddressSpace中，root和current_map分别指向了树状的地址空间和对应的一维展开。

至于为什么要使用这样两种数据结构，暂时还不知道这样做的好处。等我想明白了再回来解释。
