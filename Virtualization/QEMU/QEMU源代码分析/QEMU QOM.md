# QEMU QOM

QEMU QOM is the abbreviation of the `QEMU Object Model`. This is a abstract level for these objects in QEMU. The QEMU uses QOM to makes the code has the capability like Class, Inherency etc. So we should better understand the QEMU QOM, so we can know the QEMU easily.

## Why QOM

All device creation, device configuration, backend creation and backend configuration done through a single interface.

## deepin QEMU QOM

First we have a glance at the QOM. The QOM has three parts:

1. Type register
   * type_init.
   * register_module_init
   * type_register
2. Type initialization
   * type_initalize
3. Object initialization
   * object_new
   * object_intialize
   * object_initialize_with_type

We will use the edu device which in `/hw/misc/edu.c`, this is an edu device, so it's very easy to understand. We will use it to illustrate the QOM in QEMU.

### Type register

```C
static void pci_edu_register_types(void)
  {
      static InterfaceInfo interfaces[] = {
          { INTERFACE_CONVENTIONAL_PCI_DEVICE },
          { },
      };
      static const TypeInfo edu_info = {
          .name          = TYPE_PCI_EDU_DEVICE,
          .parent        = TYPE_PCI_DEVICE,
          .instance_size = sizeof(EduState),
          .instance_init = edu_instance_init,
          .class_init    = edu_class_init,
          .interfaces = interfaces,
      };

      type_register_static(&edu_info);
  }
  type_init(pci_edu_register_types)
```

The `type_init` is a macro, and we can see it's definition:

```C
#define type_init(function) module_init(function, MODULE_INIT_QOM) 
```

```C
#define module_init(function, type)                                         \
  static void __attribute__((constructor)) do_qemu_init_ ## function(void)    \
  {                                                                           \
      register_module_init(function, type);                                   \
  }
```

Eventually call `register_module_init()`. This function will link the function into a list, one type has one link list. The link list name is `init_type_list[MODULE_INIT_MAX]`.

We should notice there has a attribute `constructor`, with this attribute. All of the function will be executed before the main function. So before main execute, all of the `Type` in QEMU has been registered.

Then when main execute, the `module_call_init` function will be called:

```C
  void module_call_init(module_init_type type)
  {
      ModuleTypeList *l;
      ModuleEntry *e;

      if (modules_init_done[type]) {
          return;
      }

      l = find_type(type);

      QTAILQ_FOREACH(e, l, node) {
          e->init();
      }

      modules_init_done[type] = true;
  }
```

It will iterate the function that registered with type. So `pci_edu_register_types` that registered by edu device will be called.

```C
static void pci_edu_register_types(void)
{
    static InterfaceInfo interfaces[] = {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    };
    static const TypeInfo edu_info = {
        .name          = TYPE_PCI_EDU_DEVICE,
        .parent        = TYPE_PCI_DEVICE,
        .instance_size = sizeof(EduState),
        .instance_init = edu_instance_init,
        .class_init    = edu_class_init,
        .interfaces = interfaces,
    };

    type_register_static(&edu_info);
}
```

The `type_register_static` will pass the `edu_info` to the `type_register_internal`, and all of the work are implemented at here.

```C
static TypeImpl *type_register_internal(const TypeInfo *info)
{
    TypeImpl *ti;
    ti = type_new(info);

    type_table_add(ti);
    return ti;
}

    92 static void type_table_add(TypeImpl *ti)
    93 {
    94     assert(!enumerating_types);
    95     g_hash_table_insert(type_table_get(), (void *)ti->name, ti);
    96 }
```

The `type_new` using the `TypeInfo` which is `edu_info` to construct the `TypeImpl`. And then add the `TypeImpl` into the `type_table`, it's a hash table. And the `name of the type` will be used as the hash key.

The `TypeImpl` definition is show at below:

```C
  struct TypeImpl
  {
      const char *name; // "edu"

      size_t class_size;

      size_t instance_size;
      size_t instance_align;

      // This seems like the constructor in C++
      void (*class_init)(ObjectClass *klass, void *data);
      void (*class_base_init)(ObjectClass *klass, void *data);

      void *class_data;	// This is point to the class_data provided by TypeInfo

      void (*instance_init)(Object *obj);
      void (*instance_post_init)(Object *obj);
      void (*instance_finalize)(Object *obj);

      bool abstract; // If true, can't create instance. Sames like C++ class

      const char *parent;	// parent name
      TypeImpl *parent_type;

      // This stores the class information, like the definition of the Class in C++
      ObjectClass *class;	// The basic information of this class

      int num_interfaces;
      InterfaceImpl interfaces[MAX_INTERFACES];
  };
```

After all type have been registered into the type hash_table. We should initialize the class for these types. As we can see from the `TypeImpl`, there are some fields related to class, like `class_size`, `class_init` and `class`. The `class_init` is copied from `TypeInfo`.

To make a type available, we should initialize the class of it. That's the step of type initialize.

### Type Initialize

The type will be initialized at `type_initialize()` function. And the first time it has been called is at `qemu_create_machine`. The flow is at below:

```Shell
#0  type_initialize (ti=0x555556885990) at ../qom/object.c:290
#1  0x0000555555da28c9 in object_class_foreach_tramp (key=0x555556885b10, value=0x555556885990, opaque=0x7fffffffe000) at ../qom/object.c:1081
#2  0x00007ffff7bb2698 in g_hash_table_foreach () at /lib/x86_64-linux-gnu/libglib-2.0.so.0
#3  0x0000555555da29af in object_class_foreach
     (fn=0x555555da2b18 <object_class_get_list_tramp>, implements_type=0x5555560aa392 "machine", include_abstract=false, opaque=0x7fffffffe050) at ../qom/object.c:1103
#4  0x0000555555da2b9e in object_class_get_list (implements_type=0x5555560aa392 "machine", include_abstract=false) at ../qom/object.c:1160
#5  0x0000555555ac0886 in select_machine (qdict=0x5555568d6500, errp=0x55555685e4a0 <error_fatal>) at ../softmmu/vl.c:1580
#6  0x0000555555ac1a0b in qemu_create_machine (qdict=0x5555568d6500) at ../softmmu/vl.c:2015
#7  0x0000555555ac5895 in qemu_init (argc=19, argv=0x7fffffffe3b8) at ../softmmu/vl.c:3533
#8  0x000055555582e90b in main (argc=19, argv=0x7fffffffe3b8) at ../softmmu/main.c:47
```

In the `type_initialize` function, it will allocate the `class` for this `TypeImpl`. And initialize it.

```C
// qom/object.c
static void type_initialize(TypeImpl *ti)
{
    if (ti->class) {
        return;
    }
    
    // Get the class and instance size, if it's not assigned by the TypeInfo, it will be 0. So the size will be the parent size.
    ti->class_size = type_class_get_size(ti);
    ti->instance_size = type_object_get_size(ti);
    
    // Allocate the ObjectClass
    ti->class = g_malloc0(ti->class_size);
    
    ti->class->properties = g_hash_table_new_full(g_str_hash, g_str_equal, NULL,
                                                  object_property_free);
    
    ti->class->type = ti;
    
    if (ti->class_init) {
        ti->class_init(ti->class, ti->class_data);
    }
}
```

We can see the parameter is `TypeImpl`. And if `ti->class` is not NULL. The TypeImpl has been initialized. At here, the `ti->class` will point to `PCIDeviceClass`.

The other thing `type_initalize` do is to initialize all of it's parent TypeImpl.

```C
parent = type_get_parent(ti);
if (parent) {
    type_initialize(parent);
}
```

In the last, it will initialize the information of the class. First initialize all of it's parent class, and last it will initialize itself.

```C
while (parent) {
    if (parent->class_base_init) {
        parent->class_base_init(ti->class, ti->class_data);
    }
    parent = type_get_parent(parent);
}

if (ti->class_init) {
    ti->class_init(ti->class, ti->class_data);
}
```

There the `ti->class_init` will call `edu_class_init`:

```C
static void edu_class_init(ObjectClass *class, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(class);
    PCIDeviceClass *k = PCI_DEVICE_CLASS(class);

    k->realize = pci_edu_realize;	// The function which used to create instance
    k->exit = pci_edu_uninit;		// Destroy instance
    k->vendor_id = PCI_VENDOR_ID_QEMU;
    k->device_id = 0x11e8;
    k->revision = 0x10;
    k->class_id = PCI_CLASS_OTHERS;
    set_bit(DEVICE_CATEGORY_MISC, dc->categories);
}
```

We have mentioned before the class is the pointer of the `PCIDeviceClass`, so it's easy to translate it to any parent class. Just simply `(PCIDeviceClass *)class`  is ok. Same like DeviceClass due to it's the parent of PCIDeviceClass, and it occupy the first position in PCIDeviceClass.

The last question? Where do the QEMU to call the `type_initialize`? Here is the flow:

```C
#0  edu_class_init (class=0x555556bdcd40, data=0x0) at ../hw/misc/edu.c:413
#1  0x0000555555e671f5 in type_initialize (ti=0x555556a7f9a0) at ../qom/object.c:365
#2  0x0000555555e689ff in object_class_foreach_tramp (key=0x555556a7fb20, value=0x555556a7f9a0, opaque=0x7fffffffdcd0) at ../qom/object.c:1070
#3  0x00007ffff7d7c508 in g_hash_table_foreach () at /lib/x86_64-linux-gnu/libglib-2.0.so.0
#4  0x0000555555e68ae2 in object_class_foreach (fn=0x555555e68c4b <object_class_get_list_tramp>, implements_type=0x555556274062 "machine", include_abstract=false, opaque=0x7fffffffdd20)
    at ../qom/object.c:1092
#5  0x0000555555e68cce in object_class_get_list (implements_type=0x555556274062 "machine", include_abstract=false) at ../qom/object.c:1149
#6  0x0000555555dda453 in select_machine (qdict=0x555556ac1c60, errp=0x555556a4def8 <error_fatal>) at ../softmmu/vl.c:1623
#7  0x0000555555ddb6c5 in qemu_create_machine (qdict=0x555556ac1c60) at ../softmmu/vl.c:2108
#8  0x0000555555ddf28f in qemu_init (argc=31, argv=0x7fffffffe038, envp=0x7fffffffe138) at ../softmmu/vl.c:3646
#9  0x0000555555954435 in main (argc=31, argv=0x7fffffffe038, envp=0x7fffffffe138) at ../softmmu/main.c:49
```

### Object Initialize

Now we have all of the class (TypeImpl) information and the data of these class has been initialized. The QEMU will create the instance according to the parameter that passed by user.

We can see how QEMU initialize the instance:

```C
// softmmu/vl.c
void qemu_init(int argc, char **argv, char **envp)
{
    qemu_opts_foreach(qemu_find_opts("device"),
                      device_init_func, NULL, &error_fatal);
}
```

This means for each `-device` that specified by user, QEMU will call `device_init_func` to handle it.

```C
static int device_init_func(void *opaque, QemuOpts *opts, Error **errp)
{
    DeviceState *dev;

    dev = qdev_device_add(opts, errp);
}

// qdev-monitor.c
DeviceState *qdev_device_add(QemuOpts *opts, Error **errp) 
{
    /* create device */
    dev = DEVICE(object_new(driver));
    
    // !!!!!!!!!!!!Notice here, it's important
    object_property_set_bool(OBJECT(dev), true, "realized", &err);
}

Object *object_new(const char *typename)
{
    TypeImpl *ti = type_get_by_name(typename);

    return object_new_with_type(ti);
}

static Object *object_new_with_type(Type type)
{
    Object *obj;

    g_assert(type != NULL);
    type_initialize(type);

    // At here, allocate the memory for the instance. For edu device, this allocate sizeof(EduState), and this object will be used as Edustate.
    obj = g_malloc(type->instance_size);
    object_initialize_with_type(obj, type->instance_size, type);
    obj->free = g_free;

    return obj;
}

static void object_initialize_with_type(void *data, size_t size, TypeImpl *type)
 {
     Object *obj = data;

     type_initialize(type);

     memset(obj, 0, type->instance_size);
     // This combined the object and the class. So the object can know I belongs to which class.
     obj->class = type->class;
     object_ref(obj);
     object_class_property_init_all(obj);
    
     object_init_with_type(obj, type); // This is important
     object_post_init_with_type(obj, type);
 }
```

With a long call list, we arrived at `object_init_with_type`. And in this function, we call the instance init.

```C
static void object_init_with_type(Object *obj, TypeImpl *ti)
{
    if (type_has_parent(ti)) {
        object_init_with_type(obj, type_get_parent(ti));
    }

    if (ti->instance_init) {
        ti->instance_init(obj);
    }
}
```

Let's go back to the edu device. The `instance_init` in edu device is `edu_instance_init`.

```C
static void edu_instance_init(Object *obj)
{
    EduState *edu = EDU(obj);

    edu->dma_mask = (1UL << 28) - 1;
    object_property_add_uint64_ptr(obj, "dma_mask",
                                   ¦       ¦       ¦       ¦  &edu->dma_mask, OBJ_PROP_FLAG_READWRITE,
                                   ¦       ¦       ¦       ¦  NULL);
}
```

Here just add the property, the more things like init struct member is in the `instance realize` step.

### Instance realize

After the instance has been init, we need to realize it. The `realize` step is like class constructor been called in C++. So we back to the `qdev_device_add`, the `realize` property will be added after the instance init.

```C
// qdev-monitor.c
DeviceState *qdev_device_add(QemuOpts *opts, Error **errp) 
{
    /* create device */
    dev = DEVICE(object_new(driver));
    
    // !!!!!!!!!!!!Notice here, it's important
    object_property_set_bool(OBJECT(dev), true, "realized", &err);
}
```

But we didn't see the `realized` property before, where is it comes from?

It is added by the `device_class_init`:

```C
static void device_class_init(ObjectClass *class, void *data)
{
    object_class_property_add_bool(class, "realized",
                                   device_get_realized, device_set_realized,
                                   &error_abort);
}
```

The property will be added into the `class->property`. When the `object_property_set_bool` been called, the `device_set_realized` will be called. What this function to do is to call the `realize` function pointer in the `DeviceClass`:

```C
static void device_set_realized(Object *obj, bool value, Error **errp)
{
    DeviceState *dev = DEVICE(obj);
    DeviceClass *dc = DEVICE_GET_CLASS(dev);
    
    if (value && !dev->realized) {
        if (dc->realize) {
            dc->realize(dev, &local_err);
		}
	}
}
```

What the `dc->realize` points to? It points to the `pci_qdev_realize`, where to set it? It's been set at `pci_device_class_init`:

```C
static void pci_device_class_init(ObjectClass *klass, void *data);
{
    DeviceClass *k = DEVICE_CLASS(klass);
    
    k->realize = pci_qdev_realize;
    k->unrealize = pci_qdev_unrealize;
    k->bus_type = TYPE_PCI_BUS;
}
```

Now let's take a look at `pci_qdev_realize`:

```C
static void pci_qdev_realize(DeviceState *qdev, Error **errp)
{
    PCIDevice *pci_dev = (PCIDevice *)qdev;
    PCIDeviceClass *pc = PCI_DEVICE_GET_CLASS(pci_dev);
    
    if (pc->realize) {
        pc->realize(pci_dev, &local_err);
    }
}
```

It's the same with above. `pc->realize` is point to `pci_edu_realize`, and it's set in `edu_class_init`:

```C
static void edu_class_init(ObjectClass *class, void *data)
{
    PCIDeviceClass *k = PCI_DEVICE_CLASS(class);
    
    k->realize = pci_edu_realize;
}
```

And in `pci_edu_realize`, the edu device do the most initiate work for itself:

```C
static void pci_edu_realize(PCIDevice *pdev, Error **errp)
{
    EduState *edu = EDU(pdev);
    uint8_t *pci_conf = pdev->config;

    pci_config_set_interrupt_pin(pci_conf, 1);

    if (msi_init(pdev, 0, 1, true, false, errp)) {
        return;
    }

    timer_init_ms(&edu->dma_timer, QEMU_CLOCK_VIRTUAL, edu_dma_timer, edu);

    qemu_mutex_init(&edu->thr_mutex);
    qemu_cond_init(&edu->thr_cond);
    qemu_thread_create(&edu->thread, "edu", edu_fact_thread,
                       ¦       ¦      edu, QEMU_THREAD_JOINABLE);

    memory_region_init_io(&edu->mmio, OBJECT(edu), &edu_mmio_ops, edu,
                          ¦       ¦   "edu-mmio", 1 * MiB);
    pci_register_bar(pdev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &edu->mmio);
}
```

For programmer who want to add new devices into QEMU, we most care about the realize, because at here we initialize the device function eventually.

### Property

## The hierarchy of the object in QEMU

```C
// hw/pci/pci.c
static const TypeInfo pci_device_type_info = {
    .name = TYPE_PCI_DEVICE,
    .parent = TYPE_DEVICE,
    .instance_size = sizeof(PCIDevice),
    .abstract = true,
    .class_size = sizeof(PCIDeviceClass),
    .class_init = pci_device_class_init,
    .class_base_init = pci_device_class_base_init,
};
```

```C
// hw/core/qdev.c
static const TypeInfo device_type_info = {
    .name = TYPE_DEVICE,
    .parent = TYPE_OBJECT,
    .instance_size = sizeof(DeviceState),
    .instance_init = device_initfn,
    .instance_post_init = device_post_init,
    .instance_finalize = device_finalize,
    .class_base_init = device_class_base_init,
    .class_init = device_class_init,
    .abstract = true,
    .class_size = sizeof(DeviceClass),
    .interfaces = (InterfaceInfo[]) {
        { TYPE_VMSTATE_IF },
        { TYPE_RESETTABLE_INTERFACE },
        { }
    }
};
```

```C
// qom/object.c
static TypeInfo object_info = {
    .name = TYPE_OBJECT,
    .instance_size = sizeof(Object),
    .class_init = object_class_init,
    .abstract = true,
};
```

## Summary

1. Every Class registered at QEMU boot time. And QEMU init every Class first (Written by TypeImpl).
2. Then allocate a object that is used to as an instance of a Class. And we can add property at the object initialize time.
3. Set the `realize` property to true. And this will call the realize function eventually. And we always implement most of the device initialize here.