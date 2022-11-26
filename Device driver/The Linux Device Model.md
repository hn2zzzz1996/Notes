# The Linux Device Model

The 2.6 device model provides a general abstraction describing the structure of the system. It is used within the kernel to support a wide variety of tasks:

* Power management and system shutdown

* Communications with user space

  The implementation of the **sysfs** virtual filesystem is tightly **tied into the device module** and **exposes the structure** represented by it.

* Hotpluggable devices

* Device classes

  The device model includes a mechanism for `assigning devices to classes,`  which describe those device at a `higher, functional level` and allow them to `be discovered from user space`.

* Object lifecycles

## Kobjects, Ksets, and Subsystems

The `kobject` is the fundamental structure of the device model. It include:

* Reference counting of objects

  Used to tracking the lifecycle of objects.

* Sysfs representation

  Every object that shows up in sysfs has, a kobject that interacts with the kernel to create its visible representation.  

* Data structure glue

* Hotplug event handling

### Kobject Basics

`struct kboject` ot represent kobject. It's defined in `<linux/kobject.h>`.

#### Embedding kobjects

A kobject only exist to tie a higher-level object into the device model.

For example, let's look at `struct cdev`:

```C
struct cdev {
    struct kobject kobj;
    struct module *owner;
    struct file_operations *ops;
    struct list_head list;
    dev_t dev;
    unsigned int count;
};
```

#### kobject initialization

The first is to `set the entire kobject to 0`, usually with a call to `memset`. Failure to zero out a kobject often leads to very strange crashes further down the line.

The next step is to set up some of the internal fields with a call to `kobject_init()`:

```C
void kobject_init(struct kobject *kobj);
```

Also, `kobject_init` sets the kobjects's reference count to one. And Kobject users **must set the name of the kboject**, this is the name that is used in sysfs entries.

Using this function to set the name:

```C
int kobject_set_name(struct kobject *kobj, const char *format, ...);
```

Notice: this function may be fail due to it may try to allocate memory.

#### Reference count manipulation

```C
struct kobject *kobject_get(struct kobject *kobj);
void kobject_put(struct kobject *kobj);
```

If the kobject is already in the process of being destroyed, the `kobject_get` failed and returns NULL.