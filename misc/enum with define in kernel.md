# enum with define in kernel

Sometimes we want to list some of the flags in kernel. And we have many ways to do it, here is the two ways to do it. One it normal way, and the other is the most good way now in kernel.

The first is use the enum, that's the common way, for example:

```C
enum {
    RMAP_NONE = 0;
    RMAP_EXCLUSIVE = BIT(0);
    RMAP_COMPOUND = BIT(1);
}
```

And the another good way is, using the `__bitwise` and `define`, for example:

```C
/* These are extracted from include/linux/rmap.h, and we can imitate it. */
/* RMAP flags, currently only relevant for some anon rmap operations. */
typedef int __bitwise rmap_t;

/*
 * No special request: if the page is a subpage of a compound page, it is
 * mapped via a PTE. The mapped (sub)page is possibly shared between processes.
 */
#define RMAP_NONE		((__force rmap_t)0)

/* The (sub)page is exclusive to a single process. */
#define RMAP_EXCLUSIVE		((__force rmap_t)BIT(0))

/*
 * The compound page is not mapped via PTEs, but instead via a single PMD and
 * should be accounted accordingly.
 */
#define RMAP_COMPOUND		((__force rmap_t)BIT(1))
```

