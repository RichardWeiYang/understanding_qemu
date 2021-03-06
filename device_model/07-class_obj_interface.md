因为用c语言实现了一个类面向对象的设备模型，在类型和实例之间的转换都需要手工实现。所以在代码中有几个重要的宏定义来完成这样的工作。

# obj -> class

对应的obj找到它的class。

```
#define OBJECT_GET_CLASS(class, obj, name) \
    OBJECT_CLASS_CHECK(class, object_get_class(OBJECT(obj)), name)
```

obj就是我们传入的对象实例， object_get_class就是获得obj->class这个成员，也就是对应的class。
class就是这个对象对应的class的类型，name呢就是这个class的名字。
对于没有intefaces的类型来说，这就是一个强制类型转换。也就是得到了obj对应的类型。

# ObjectClass -> 子Class

在上面的图中我们也看到，类型之间也有着父子关系。比如PCIDeviceClass就是DeviceClass的子类。

所以有时候在代码中我们需要从父类获得具体的子类。这个工作交给了：

```
#define OBJECT_CLASS_CHECK(class_type, class, name) \
    ((class_type *)object_class_dynamic_cast_assert(OBJECT_CLASS(class), (name), \
                                               __FILE__, __LINE__, __func__))
```

这个宏其实就是上面的那个宏，如果没有interfaces成员这也是一个强制类型转换。把class类型转换成class_type。

但是因为Qemu自身的面向对象的模型，子类和父类的基地址相同所以能够通过强制转换来得到子类。

其实这么看，反过来转换也是可以的。

# Object -> 子Object

同上面的类型一样，object也有父子关系。所以当代码中需要从父对象转换到一个具体的子对象时就要用：

```
#define OBJECT_CHECK(type, obj, name) \
    ((type *)object_dynamic_cast_assert(OBJECT(obj), (name), \
                                        __FILE__, __LINE__, __func__))
```

对obj来讲就没有类型Class这么复杂了，在生产环境中纯粹就是强制类型转换，把obj类型转换到type类型。
因为按照Qemu的面向对象模型，父子obj的基地址是一样的。


# Class -> InterfaceClass

这个转换和Class到子Class的方式一样，也是

```
#define OBJECT_CLASS_CHECK(class_type, class, name) \
    ((class_type *)object_class_dynamic_cast_assert(OBJECT_CLASS(class), (name), \
                                               __FILE__, __LINE__, __func__))
```

但是因为这个Class有interfaces成员，所以会在interfaces链表上查找名字为name的接口。

如果有则返回对应name的接口。

# Object -> Interface

从上面的转换可以看出，如果想要从一个obj得到interface可以分成两步：

  * obj -> class
  * class -> interface

但是因为当Class有interface成员，且传入的name就是接口类型的名字时，obj可以直接转换成接口。

也就是用OBJECT_CLASS_CHECK可以直接转换得到。

# obj -> 虚obj

所以虚obj就是obj对象对应的Class类型的接口类型的对象。这么说还真有点绕，估计我自己过段时间来看也得花点时间回忆回忆。

之所以我叫它虚obj，是因为接口类型本身是没有对应的对象的。所以这个东西还挺神奇，没想到代码里还会这么用。

大家也一定觉得不信，那先举个例子：

```
#define MEMORY_DEVICE(obj) \
     INTERFACE_CHECK(MemoryDeviceState, (obj), TYPE_MEMORY_DEVICE)

typedef struct MemoryDeviceState MemoryDeviceState;
```

这个MemoryDeviceState就定义了这么一个类型，没有任何的成员。而它对应的Class类型是TYPE_MEMORY_DEVICE的接口类型。

而在代码中通常的用法还是要将这个虚obj再转换到接口类型。通常用法如下：

```
const MemoryDeviceState *md = MEMORY_DEVICE(obj);
const MemoryDeviceClass *mdc = MEMORY_DEVICE_GET_CLASS(obj);
```

这个obj就是一个真实的obj对象，而md则是对应的虚obj。其实他们俩的指是一样的，都指向了同一个地址。

还是觉得很神奇，为什么代码要这么写呢？就不能直接传obj么？

我能想到的一个作用就是在debug选项打开时能够检测类型，而不至于传错参数。
