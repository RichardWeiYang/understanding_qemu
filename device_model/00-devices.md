qemu作为一个虚拟机的软件，其重要功能之一就是模拟设备。说实话，这个设备模拟的模型还挺复杂的。

看过好几次都没有记清楚，这次重新梳理一遍，并记录在此。

# 设备类型注册

qemu中模拟的每一种设备都在代码中对应了一个类型，这个类型在使用之前需要注册到系统中。这样的好处是后续增添设备的流程变得简单化了。

这一节就来看看设备类型注册的流程。

[设备类型注册][1]

# 设备类型初始化

设备类型注册后，在需要使用之前得初始化该类型，并生成对应得ObjectClass对象。

[设备类型初始化][2]

# 设备实例化

接着就是实例化设备类型，也就是真的生成一个设备给虚拟机使用。

[设备实例化][3]

# device实例的隐藏套路

对于qemu中一个"device"设备，除了实例化中instance_init函数之外，还有一个非常重要的实例化套路。

[device实例的隐藏套路][4]

# 面向对象的设备模型

[面向对象的设备模型][]

[1]: /device_model/01-type_register.md
[2]: /device_model/02-register_objectclass.md
[3]: /device_model/03-objectclass_instance.md
[4]: /device_model/04-device_hidden_part.md
