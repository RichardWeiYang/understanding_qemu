本小节主要讲清楚type_table这个hash table的由来。

为了比较清楚的解释这个流程，在本节中以e1000这种设备为例。其代码主要集中在hw/net/e1000.c这个文件中。

# TypeInfo定义设备

qemu中注册的每个设备都由一个TypeInfo类型来定义。这个定义的内容不太多，我就直接上代码了。

```
struct TypeInfo
{
    const char *name;
    const char *parent;

    size_t instance_size;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;
    size_t class_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void (*class_finalize)(ObjectClass *klass, void *data);
    void *class_data;

    InterfaceInfo *interfaces;
};
```

那对于一个e1000设备，这个类型是什么样子的呢？

```
#define TYPE_E1000_BASE "e1000-base"

static const TypeInfo e1000_base_info = {
    .name          = TYPE_E1000_BASE,
    .parent        = TYPE_PCI_DEVICE,
    .instance_size = sizeof(E1000State),
    .instance_init = e1000_instance_init,
    .class_size    = sizeof(E1000BaseClass),
    .abstract      = true,
    .interfaces = (InterfaceInfo[]) {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    },
};
```

暂时我们不关系其他内容，只看到定义时赋值的头两个值是：

  * name：  本设备类型的名称
  * parent：父设备类型的名称

这里可以看出在qemu设备模型中使用**名称作为类型的唯一标识**的，并且还存在了父子关系（这点我们在后面再讲）。

# 类型注册函数type_register()

定义了设备类型后，需要做的是注册这个类型。这样qemu才知道现在可以支持这种设备了。

注册的函数就是type_register()，做的工作也很简单：

  * 通过type_new()生成一个TypeInfo对应的TypeImpl类型
  * 并以name为关键字添加到名为type_table的一个hash table中

假如我们用一个图来描述，大概可以画成这样。

```
       type_table(GHashTable)  ; this is a hash table with name as the key
       +-----------------------+
       |                       |
       +-----------------------+
                |
                v
       +------------------+                   +--------------------+
       |TypeImpl*         | <--- type_new()   | TypeInfo           |
       +------------------+                   +--------------------+
                |
                v
       +------------------+                   +--------------------+
       |TypeImpl*         | <--- type_new()   | TypeInfo           |
       +------------------+                   +--------------------+
```

# 何时注册设备类型

不过这些都不难，着重我想说的是type_register调用的时机。而这个过程又分了两步走：

  * “**注册**”设备注册函数
  * “**执行**”设备类型注册

这个东西是有点绕，那就以e1000为例来看。

## 注册设备注册函数

在e1000的实现中，我们可以看到如下的过程：

```
static void e1000_register_types(void)
{
    ...
    type_register(&type_info);
}

type_init(e1000_register_types)
```

神奇的地方就在这个type_init()，使用了一个我以前不知道的gcc方法__attribute__((constructor))。

来看看函数的定义先：

```
#define type_init(function) module_init(function, MODULE_INIT_QOM)

#define module_init(function, type)                                         \
static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
{                                                                           \
    register_module_init(function, type);                                   \
}
```

因为使用了__attribute__((constructor))修饰，所以这个函数将会在main函数执行前被执行。那这个register_module_init()函数又干了啥呢？

```
void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    ModuleTypeList *l;

    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;

    l = find_type(type);

    QTAILQ_INSERT_TAIL(l, e, node);
}
```

所以说qemu有时候有点蛋疼。又整了一个链表，把设备类型注册函数添加到里面。在这个e1000的例子中就是e1000_register_types这个函数。

细心的朋友可能还注意到了，这个链表还分了类型。对type_init()而言，这个类型是**MODULE_INIT_QOM**。记住这个，后面我们将会用到。

## 执行设备类型注册

刚才忙活了一堆，其实这个类型注册函数e1000_register_types还没有被执行到。那究竟什么时候执行真正的类型注册呢？

让偶来揭示谜团：

```
  main()
    module_call_init(MODULE_INIT_QOM);
    {
      ModuleTypeList *l;
      ModuleEntry *e;

      l = find_type(type);

      QTAILQ_FOREACH(e, l, node) {
      e->init();
      }
    }
```

看到上一节中的MODULE_INIT_QOM了没有？其作用就是找到MODULE_INIT_QOM对应的链表，执行其中的init函数。也就是我们刚通过register_module_init添加进去的e1000_register_types了。

到这，终于算是把type_table这个hash table的由来说清楚了。
