# MSR in KVM

首先需要注意的是在x86.c里面定义了三个大数组，分别是`msrs_to_save_all`, `emulated_msrs_all` and `msr_based_features_all`，在KVM初始化的时候，会根据这几个数组里的内容，过滤掉不支持的MSR，然后将可以支持的MSR List保存在`msrs_to_save`,`emulated_msrs` and `msr_based_features`三个数组中。

为什么要这样定义一个数组，然后在初始化的时候再拷贝一份呢？这是因为在之前的实现中，这几个数组都定义在x86.c中，并且只有一份，也就是在KVM模块中，当加载kvm-intel.ko模块的时候，才会去初始化整个kvm模块，而在初始化的时候，kvm-intel模块可能会修改这些数组当中的项，当卸载了kvm-intel模块之后，被其所修改过的内容仍然保留在了预定义的数组当中，这样在下一次加载kvm-intel.ko去初始化的时候，用的就是上一次遗留的数据，可能会导致问题。所以必须用一份const的，然后在运行时做一份拷贝。

`msrs_to_save`中保存的是host cpu所支持的msr.

`emulated_msrs`中保存的是KVM特有的msr.

`msr_based_features`中保存的是与cpu feature有关的msr.

## KVM_GET_MSR_INDEX_LIST

这个ioctl用来从KVM获取所有支持的MSR的index的list，可以在x86.c里面找到实现的代码，可以看到在这里只是简单的把`msrs_to_save`...三个数组拷贝给了用户空间，也就是完成了返回工作。

那这三个数组是在哪里完成初始化的呢？是在`kvm_arch_hardware_setup`中调用的`kvm_init_msr_list`中完成的.

QEMU如何使用？

QEMU在`kvm_get_supported_msrs`中调用`ioctl(KVM_GET_MSR_INDEX_LIST)`，然后使用这些返回的msr list作为kvm是否支持某特性的依据。

## KVM_GET_MSRS

QEMU首先使用`KVM_GET_MSR_FEATURE_INDEX_LIST`获取KVM支持的MSR_FEATURE的列表，然后把该返回的列表保存到一个全局的缓存变量`kvm_feature_msrs`中。

在之后调用`kvm_arch_get_supported_msr_feature`的时候，会判断当前要get的msr是否在`kvm_feature_msrs`中，只有在其中，才去调用`KVM_GET_MSRS`去获取其实际的data。

```C
uint64_t kvm_arch_get_supported_msr_feature(KVMState *s, uint32_t index)
{
    /* Check if requested MSR is supported feature MSR */
        int i;
        for (i = 0; i < kvm_feature_msrs->nmsrs; i++)
            if (kvm_feature_msrs->indices[i] == index) {
            ¦   break;
            }
        if (i == kvm_feature_msrs->nmsrs) {
            return 0; /* if the feature MSR is not supported, simply return 0 */
        }

}
```

## get/set msr flow

下面以get_msr为例，说明KVM中实现的流程。

首先需要知道有两个API是QEMU与KVM交互MSR的，分别是`KVM_GET_MSRS`和`KVM_SET_MSRS`。

这两个API的处理都在x86.c里面：

```C
long kvm_arch_vcpu_ioctl(struct file *filp, unsigned int ioctl, unsigned long arg)
{
    switch (ioctl) {
        case KVM_GET_MSRS:
            int idx = srcu_read_lock(&vcpu->kvm->srcu);
            r = msr_io(vcpu, argp, do_get_msr, 1);
            srcu_read_unlock(&vcpu->kvm->srcu, idx);
            break;
        case KVM_SET_MSRS:
			int idx = srcu_read_lock(&vcpu->kvm->srcu);
            r = msr_io(vcpu, argp, do_set_msr, 0);
            srcu_read_unlock(&vcpu->kvm->srcu, idx);
            break;
    }
}


```

首先看下函数`kvm_get_msr_ignored_check`，这个函数里有一个特殊的变量叫`host_initiated`，如果是true，则是qemu调用的get_msr；如果是false，则是KVM自己调用的get_msr。

```C
static int kvm_get_msr_ignored_check(struct kvm_vcpu *vcpu,
                                  ¦    u32 index, u64 *data, bool host_initiated)
{
    // __kvm_get_msr will call static_call(kvm_x86_get_msr)
    // and it is vmx_get_msr to get the value
    int ret = __kvm_get_msr(vcpu, index, data, host_initiated);
}

int kvm_get_msr(struct kvm_vcpu *vcpu, u32 index, u64 *data)
{
    return kvm_get_msr_ignored_check(vcpu, index, data, false);
}

/*
 * Adapt set_msr() to msr_io()'s calling convention
 */
static int do_get_msr(struct kvm_vcpu *vcpu, unsigned index, u64 *data)
{
    return kvm_get_msr_ignored_check(vcpu, index, data, true);
}
```

可以看到两个函数对于`host_initiated`设置的不同：

* do_get_msr, called in msr_io, and it's used for qemu ioctl.
* kvm_get_msr, called in emulator_get_msr(), and it's called by kvm itself.

不管是get还是set msr，最终都会调用到`vmx_get/set_msr`函数中，调用流程如下所示：

```C
kvm_arch_vcpu_ioctl
    -> do_set_msr
    	-> kvm_set_msr_ignored_check
    		-> __kvm_set_msr
    			-> vmx_set_msr
```

在`vmx_set_msr`中，会将msr的值设置到具体的内容中去，可供guest读取。

```C
// Writes msr value into the appropariate "register".
int vmx_set_msr(struct kvm_vcpu *vcpu, struct msr_data *msr_info)
{
    switch (msr_info->index) {
        case MSR_GS_BASE:
            vmcs_writel(GUEST_GS_BASE, data);
            break;
        case MSR_IA32_FEAT_CTL:
            vmx->msr_ia32_feature_control = data;
            break;
        ...
    }
}
```

上面展示了处理MSR的两种方式，一种是直接写入到vmcs里面，让CPU自动切换；另一种是记录到vmx数据结构中，当guest需要访问该msr的时候，通过模拟的方式将保存到vmx数据结构中的值返回给guest。



如果在PKVM里面实现MSR模拟的话，首先与vmcs相关的项都不能够让kvm去设置，得由pkvm自己去设置，包括初始化的过程，这里可以参考tdx的实现，加上一个`is_emulated_msr`函数的判断，能够让KVM处理的就交给KVM处理，剩下的都由pkvm去处理。对于初始化MSR，能够让KVM去处理的就交给KVM去处理吧，对于那些不应该让KVM去访问的MSR，应该有一个合适的时机让PKVM去初始化这些MSR，这个初始化应该在初始化VCPU的时候进行，想到两种方法：

1. 在`vcpu_create`函数中hypercall到pkvm中进行初始化，然后在pkvm中将需要初始化的MSR都初始化好。
2. 在QEMU中的vcpu_init中调用一个ioctl下来，然后再调用到pkvm中进行初始化。（这里又有一个问题就是ioctl是自己新定义一个，还是复用其他的ioctl的api）。

如何初始化MSR？如何确定哪些部分的MSR确实需要PKVM去初始化。我想PKVM首先初始化和模拟一小部分MSR，比如`MSR_GS_BASE`、`MSR_FS_BASE`、`SYSENTER_EIP`等等，剩下的MSR暂时都留给KVM去处理，可以仿照TDX的做法，在vmx的get/set_msr中加一个过滤的函数，可以被KVM的处理才处理，不能处理的先返回错误。

对于uret的msr，这一部分msr因为切换的开销比较大，所以采用了延迟切换的方法优化效率，可以暂时忽略这部分msr，还是统一切换。

对于运行时来说，如果有get/set msr，那么会首先trap到pkvm，由pkvm保护的那部分由pkvm自己emulate，剩下的可以返回给kvm，如果要返回给kvm去模拟，需要考虑几个问题:

1. 当从pkvm返回到kvm的时候，会调用ops->handle_exit，在这个ops里面，可以加上一个pkvm的hook，如果是SecureVM的vcpu，则调用pkvm的handler去处理exit，考虑如下问题。

   * VM的exit_reason如何获取？
   * 要get的msr的index如何获取？
   * get到的msr的值如何返回给pkvm？

   这里使用vcpu中的寄存器传递参数，对于SecureVM来说，寄存器的信息对于上面的KVM来说应该是不可见的，所以可以直接将某些信息填充到vcpu的寄存器中去，然后通过定义一种协议，比如PKVM传递给KVM的参数p1,p2分别保存在r10,r11寄存器中，KVM给PKVM的返回值保存在RAX寄存器中。

2. 如果pkvm想要kvm去处理这部分的msr，应该给kvm返回什么exit_reason，这里有两个选择，一个是直接返回`EXIT_REASON_MSR_READ`这种，然后KVM直接根据这个exit_reason去进行处理；另一种是先返回一个`EXIT_REASON_NEED_KVM`，说明退出原因是想要KVM去处理，然后调用到一个专门的函数中再去细分原因处理。

