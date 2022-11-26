# QEMU options

This post introduce the QemuOpts API inside QEMU. QemuOpts was [introduced in 2009](https://github.com/qemu/qemu/commit/e27c88fe9eb26648e4fb282cb3761c41f06ff18a):

```Shell
QemuOpts: framework for storing and parsing options.
This stores device parameters in a better way than unparsed strings.

New types:
  QemuOpt       -  one key-value pair.
  QemuOpts      -  group of key-value pairs, belonging to one
                   device, i.e. one drive.
  QemuOptsList  -  list of some kind of devices, i.e. all drives.

Functions are provided to work with these types.  The plan is that some
day we will pass around QemuOpts pointers instead of strings filled with
"key1=value1,key2=value2".

Check out the next patch to see all this in action ;)

Signed-off-by: Gerd Hoffmann <kraxel@redhat.com>
Signed-off-by: Anthony Liguori <aliguori@us.ibm.com>
```

It's a simple abstraction that handles two tasks:

1. Parsing of config files and command-link options
2. Storage the configuration options

## Data structures

The QemuOpts data model is pretty simple:

* `QemuOptsList` carries the list of all options belonging to a given `config group`. Each entity is represented by a `QemuOpts` struct. (Why designed like this way? Because qemu may have many devices, like have two disk, but every disk may have different properties, so they will have different QemuOpts, thus need a QemuOptsList to include all disk's config options).
* `QemuOpts` represents a set of key-value pairs which belongs to one option.
* `QemuOpt` is a single key=value pair.

Some config groups have mulitple `QemuOpts` structs (e.g. "drive", "object", "device" that represent multiple drives, multiple objects, and multiple devices, respectively), while others always have only one `QemuOpts` struct (e.g. the "machine" config group).

For example, the following command-line options:

```Shell
-drive id=disk1,file=disk.raw,format=raw,if=ide \
-drive id=disk2,file=disk.qcow2,format=qcow2,if=virtio \
-machine usb=on -machine accel=kvm
```

are represented internally as:

![qemuopts-example.mmd](.\pictures\qemuopts-example.mmd.png)

QEMU using the `QEMUOption` to describe the supported command line options.

The definition of the `QEMUOption` is below:

```C
// softmmu/vl.c
// The only flags will be set in QEMUOption. Means if this option has arguments.
868 #define HAS_ARG 0x0001

870 typedef struct QEMUOption {
       // The option name.
   871     const char *name;
   872     int flags;
   873     int index;
    // The architecture it supported.
   874     uint32_t arch_mask;
   875 } QEMUOption;
```

And there is a `qemu_options` array which stores the entire supported options. It's showed below:

```C
   877 static const QEMUOption qemu_options[] = {
   878     { "h", 0, QEMU_OPTION_h, QEMU_ARCH_ALL },
   879
✗  880 #define DEF(option, opt_arg, opt_enum, opt_help, arch_mask)     \
   881     { option, opt_arg, opt_enum, arch_mask },
✗  882 #define DEFHEADING(text)
✗  883 #define ARCHHEADING(text, arch_mask)
   884
   885 #include "qemu-options.def"
   886     { NULL },
   887 };
```

The definitions are all in the `qemu-options.def` file, this file is generated from `qemu-options.hx`.

```C
// include/qemu/qemu-optins.h
31 enum {
   32
   33 #define DEF(option, opt_arg, opt_enum, opt_help, arch_mask)     \
   34     opt_enum,
   35 #define DEFHEADING(text)
   36 #define ARCHHEADING(text, arch_mask)
   37
   38 #include "qemu-options.def"
   39 };
```

The `qemu_options` only provide which command options qemu supported. But how to resolve these options? The qemu using `QemuOptsList` to solve the problem.

The `QemuOptsList` is defined as below:

```C
// include/qemu/option.h
struct QemuOptsList {
      const char *name;
      const char *implied_opt_name;
      bool merge_lists;  /* Merge multiple uses of option into a single list? */
      QTAILQ_HEAD(, QemuOpts) head;
      QemuOptDesc desc[];
  };
```

One `QemuOptsList` tells qemu how to resolve a option, because a option can have many properties, qemu should know how to resolve them and know if it's a supported option.

Here is an example of the `qemu_mem_opts`:

```C
   402 static QemuOptsList qemu_mem_opts = {
   403     .name = "memory",
   404     .implied_opt_name = "size",
   405     .head = QTAILQ_HEAD_INITIALIZER(qemu_mem_opts.head),
   406     .merge_lists = true,
   407     .desc = {
   408         {
   409         ¦   .name = "size",
   410         ¦   .type = QEMU_OPT_SIZE,
   411         },
   412         {
   413         ¦   .name = "slots",
   414         ¦   .type = QEMU_OPT_NUMBER,
   415         },
   416         {
   417         ¦   .name = "maxmem",
   418         ¦   .type = QEMU_OPT_SIZE,
   419         },
   420         { /* end of list */ }
   421     },
   422 };
```

## Reference

https://habkost.net/posts/2016/12/qemu-apis-qemuopts.html