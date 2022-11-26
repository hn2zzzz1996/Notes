有关oops信息的解释在内核文档里`/Documentation/admin-guide/bug-hunting.rst`

**栈回溯信息的解释**

```bash
[<c001a6f4>] (s3c2410fb_probe+0x0/0x560) from [<c01bf4e8>] (platform_drv_
probe+0x20/0x24)
```

如上所示，这行信息分为几部分：

1. platform_drv_probe调用了fb_probe函数。
2. * "c001a6f4"是fb_probe函数的首地址偏移0的地址，这个函数大小为0x560;
   * "c01bf4e8"是platform_drv_probe函数首地址偏移0x20的地址，这个函数大小为0x24.
3. 后半部的"c01bf4e8"表示fb_probe函数执行后的返回地址。

**出错指令附近的机器码**

Code: e24cb004 e24dd010 e59f34e0 e3a07000 (e5873000)

