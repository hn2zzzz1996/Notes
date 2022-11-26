# Commit message For Upstream

## Fix a bug

We should add `Fixes` identifier in our commit message to tell other people this commit fixes which commit that import the bug.

Here is an example:

```C
commit 3d6a3d3a2a7a3a60a824e7c04e95fd50dec57812
Author: Alain Volmat <alain.volmat@foss.st.com>
Date:   Fri Feb 5 09:51:40 2021 +0100

    i2c: stm32f7: fix configuration of the digital filter

    The digital filter related computation are present in the driver
    however the programming of the filter within the IP is missing.
    The maximum value for the DNF is wrong and should be 15 instead of 16.

    Fixes: aeb068c57214 ("i2c: i2c-stm32f7: add driver")

    Signed-off-by: Alain Volmat <alain.volmat@foss.st.com>
    Signed-off-by: Pierre-Yves MORDRET <pierre-yves.mordret@foss.st.com>
    Signed-off-by: Wolfram Sang <wsa@kernel.org>
```

Here is an question? **How to get the format commit message that followed at Fixes tag?**

We can use the `git log --pretty` command:

```Shell
git log --pretty='%h ("%s")'
```

Will shows like below:

```C
9f13db705fb0 ("pkvm: Make the nested_vmexit() can skip_instruction if it handled vmcall")
dcb4e7315221 ("pkvm: Using vmx to replace vcpu to make the name consistant")
3c086a965455 ("pkvm: Add share/unshare memory hypercalls and handler")
9976dddf84a7 ("pkvm: Fix the missing prot init in __pkvm_hyp_donate_host()")
2667c2cf5aef ("pkvm: Check multiple page state when host_undonate_guest()")
b9eb32cccc06 ("pkvm: Implement __pkvm_guest_unshare_host()")
5907b0b9330c ("pkvm: Implement __pkvm_guest_share_host()")
```

What if we want to get one commit message?

```C
git show <commit-hash> --pretty='%h ("%s")'
```

And then we can simply copy it.