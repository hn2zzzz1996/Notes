# QEMU 初始化cpuid的feature list

在QEMU中预先定义好了非常多的CPU模型的基本信息，这些信息用一个结构体`X86CPUDefinition`来表示，该结构体中包含了CPU特性的数组`features`。QEMU中定义了各种各样的CPU，都在数组`builtin_x86_defs`中，每一项表示一种模拟的CPU模型，其`features`以及各种其他数据都是事先定义好的。

如果启用了KVM，那么QEMU就要采用与host相同的CPU定义，这时会使用一个名为`max_x86_cpu_type_info`的CPU类型，其定义如下：

```C
static const TypeInfo max_x86_cpu_type_info = {
    .name = X86_CPU_TYPE_NAME("max"),
    // parent is x86_cpu_type_info
    .parent = TYPE_X86_CPU,
    .instance_init = max_x86_cpu_initfn,
    .class_init = max_x86_cpu_class_init,
};
```

`max_x86_cpu`就是为了"**Enables all features supported by the accelerator in the current host**".

在`max_x86_cpu_initfn`中会设置cpu->max_features为true。这在后面x86CPU realize的时候会用到。

`x86_cpu_realizefn`是x86CPU的具象化函数，在其中会获取host的feature并且扩充或减少feature根据命令行指定的参数，这里重点关注如何获取host的feature：

```C
// target/i386/cpu.c
void x86_cpu_realizefn(DeviceState *dev, Error **errp)
{
    X86CPU *cpu = X86_CPU(dev);
    
    // 在这个函数里会获取host的feature
    x86_cpu_expand_features(cpu, &local_err);
}

void x86_cpu_expand_feature(X86CPU *cpu, Error **errp)
{
    // 只有是开启了KVM的情况下，也就是当前CPU是max_x86_cpu的情况下，才会获取
    // host的cpu feature
    if (cpu->max_feature) {
        for (w = 0; w < FEATURE_WORDS; w++) {
            env->features[w] |=
                x86_cpu_get_supported_feature_word(w, cpu->migratable, false) &
                   ~env->user_minus_features[w] &
                   ~feature_word_info[w].no_autoenable_flags;
        }
    }
}
```

`FeatureWord`是一个枚举的结构，里面的每一项都是一个相应的feature，通过调用`x86_cpu_get_supported_feature_word`来获取host的feature信息。

```C
uint64_t x86_cpu_get_supported_feature_word(FeatureWord w,
                                            bool migratable_only,
                                            bool is_tdx)
{
    // feature_word_info中定义了该feature所属的type，以及各个位上feature的名字，
    // 所对应的cpuid的input和output，
    FeatureWordInfo *wi = &feature_word_info[w];
    uint64_t r = 0;

    if (kvm_enabled()) {
        switch (wi->type) {
        // 对于不同类型的feature，使用不同的函数获取
        case CPUID_FEATURE_WORD:
            if (is_tdx && kvm_tdx_enabled()) {
                r = tdx_get_supported_cpuid(wi->cpuid.eax, wi->cpuid.ecx,
                                            wi->cpuid.reg);
            } else {
                // eax,ecx is the index of cpuid
                // The reg contains the output
                r = kvm_arch_get_supported_cpuid(kvm_state, wi->cpuid.eax,
                                                 wi->cpuid.ecx,
                                                 wi->cpuid.reg);
            }
            break;
        case MSR_FEATURE_WORD:
            r = kvm_arch_get_supported_msr_feature(kvm_state,
                        wi->msr.index);
            break;
        }
    }
    return r;
}
```

这里有两种类型：

* CPUID_FEATURE_WORD, call `kvm_arch_get_supported_cpuid`.
* MSR_FEATURE_WORD, call `kvm_arch_get_supported_cpuid.`

这里首先看CPUID_FEATURE_WORD类型：

```C
uint32_t kvm_arch_get_supported_cpuid(KVMState *s, uint32_t function,
                                      uint32_t index, int reg)
{
    struct kvm_cpuid2 *cpuid;
    
    // 首先从host获取所有的host支持的cpuid的列表
    cpuid = get_supported_cpuid(s);
    
    // 从cpuid的列表中找出当前要查询的cpuid
	struct kvm_cpuid_entry2 *entry = cpuid_find_entry(cpuid, function, index);
    // 获得返回值
    if (entry) {
        ret = cpuid_entry_get_reg(entry, reg);
    }
    
    // Fixups for the data returned by KVM
    // 这里会对KVM返回的值做一些修改
    ...
        
    return ret;
}

/* Run KVM_GET_SUPPORTED_CPUID ioctl(), allocating a buffer large enough
 * for all entries.
 */
static struct kvm_cpuid2 *get_supported_cpuid(KVMState *s)
{
    struct kvm_cpuid2 *cpuid;
    int max = 1;
	
    // 如果已经查询过了，直接返回缓存
    if (cpuid_cache != NULL) {
        return cpuid_cache;
    }
    // 在try_get_cpuid中分配一个max大小的缓冲区，然后调用KVM的api
	// KVM_GET_SUPPORTED_CPUID来获取host支持的cpuid的列表.
    while ((cpuid = try_get_cpuid(s, max)) == NULL) {
        max *= 2;
    }
    cpuid_cache = cpuid;
    return cpuid;
}
```

然后再来看MSR_FEATURE_WORD类型：

```C
uint64_t kvm_arch_get_supported_msr_feature(KVMState *s, uint32_t index)
{
    // 这个数据结构是在kvm_init()的时候被初始化的，里面包含了可以访问的MSR的index，
    // 但是没有data，data在后面被获取
    if (kvm_feature_msrs == NULL) { /* Host doesn't support feature MSRs */
        return 0;
    }
    
    /* Check if requested MSR is supported feature MSR */
    int i;
    for (i = 0; i < kvm_feature_msrs->nmsrs; i++)
        if (kvm_feature_msrs->indices[i] == index) {
            break;
        }
    if (i == kvm_feature_msrs->nmsrs) {
        return 0; /* if the feature MSR is not supported, simply return 0 */
    }
    
    msr_data.info.nmsrs = 1;
    msr_data.entries[0].index = index;

    // 获取该MSR寄存器的值
    ret = kvm_ioctl(s, KVM_GET_MSRS, &msr_data);
    value = msr_data.entries[0].data;
    
    // do some fix
    return value;
}
```

在将相应的feature信息填充到`env->feature`数组中之后，QEMU在`kvm_arch_init_vcpu()`中填充cpuid的项：

```C
int kvm_arch_init_vcpu(CPUState *cs)
{
    struct {
        struct kvm_cpuid2 cpuid;
        struct kvm_cpuid_entry2 entries[KVM_MAX_CPUID_ENTRIES];
    } cpuid_data;
    
    // 如果暴露KVM给Guest的话，填充相应的cpuid，让guest可以通过这个探测到自己是虚拟机
    if (cpu->expose_kvm) {
        memcpy(signature, "KVMKVMKVM\0\0\0", 12);
        c = &cpuid_data.entries[cpuid_i++];
        c->function = KVM_CPUID_SIGNATURE | kvm_base;
        c->eax = KVM_CPUID_FEATURES | kvm_base;
        c->ebx = signature[0];
        c->ecx = signature[1];
        c->edx = signature[2];

        c = &cpuid_data.entries[cpuid_i++];
        c->function = KVM_CPUID_FEATURES | kvm_base;
        c->eax = env->features[FEAT_KVM];
        c->edx = env->features[FEAT_KVM_HINTS];
    }
    
    // 填充每一项cpuid
    cpuid_i = kvm_x86_arch_cpuid(env, cpuid_data.entries, cpuid_i);
    cpuid_data.cpuid.nent = cpuid_i;
    
    // 最后将QEMU填充好的cpuid设置给KVM
    // KVM将这些cpuid拷贝到内核里
    kvm_vcpu_ioctl(cs, KVM_SET_CPUID2, &cpuid_data);
}

uint32_t kvm_x86_arch_cpuid(CPUX86State *env, struct kvm_cpuid_entry2 *entries,
                            uint32_t cpuid_i)
{
    // cpu_x86_cpuid()有多种方式返回结果:
    // 1. 一种是直接执行cpuid返回结果
    // 2. 一种是返回之前在x86_cpu_expand_feature()函数中填充的env->features中的值
    
    // 这里获取最大的cpuid的编号limit
    cpu_x86_cpuid(env, 0, 0, &limit, &unused, &unused, &unused);
    
    // 填充entries
    ...
}
```

填充好之后，通过`KVM_SET_CPUID2`将这些cpuid设置给KVM，之后KVM就可以用这些cpuid给guest做模拟了。