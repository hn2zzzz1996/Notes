```C
struct kvm_vcpu {
    struct {
		u32 queued;		/* How many page are waiting to handle asychronously. */
		struct list_head queue;
		struct list_head done;
		spinlock_t lock;
	} async_pf;
};
```

