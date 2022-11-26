# What is 0-settings and 1-settings that often used in intel spec

The 0 and 1 setting are always been used to identify If a bit can be set to 0 or 1. It's used to control some setting by bits.

To simplify, the x-settings means the bit == x can be set.

* 0-setting
  * if bit_a[i] is 0. means bit_b[i] can be 0 or 1.
  * if bit_a[i] is 1. means bit_b[i] must be 1.
* 1-setting
  * if bit_a[i] is 0. means bit_b[i] must be 0.
  * if bit_a[i] is 1. means bit_b[i] can be 0 or 1.

Here is am example that widely been used in kvm:

```C
// The ctl is what we want to set
u32 ctl = ...;

// And we read msr, it returns:
// vmx_msr_low is the 0-setting
// vmx_msr_high is the 1-setting
rdmsr(msr, vmx_msr_low, vmx_msr_high);

ctl &= vmx_msr_high; // These bit is 1 in vmx_msr_high means we can set these bit
ctl |= vmx_msr_low;  // These bit is 0 in vmx_msr_low means we can set it, bit is 1 means we must set it to 1.
```

