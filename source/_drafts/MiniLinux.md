---
title: Run a miniLinux in Qume
tags:
	- Linux
	- Kernel
	- Qemu
---



## 下载Kernel代码

```
wget https://mirrors.edge.kernel.org/pub/linux/kernel/v2.6/linux-2.6.34.tar.xz
```



## 编译内核



### 出现问题

### Q1

![](/media/monkey/DATA/Blog/blog/source/_drafts/MiniLinux/q1.png)

由于gcc 4.8不再支持linker-style架构，修改`arch/x86/vdso`

```makefile
DSO_LDFLAGS_vdso.lds = -m elf_x86_64 -Wl,-soname=linux-vdso.so.1
# 改成
DSO_LDFLAGS_vdso.lds = -m64 -Wl,-soname=linux-vdso.so.1


VDSO_LDFLAGS_vdso32.lds = -m elf_x86 -Wl,-soname=linux-gate.so.1
# 改成
VDSO_LDFLAGS_vdso32.lds = -m32 -Wl,-soname=linux-gate.so.1
```



### Q2

![](/media/monkey/DATA/Blog/blog/source/_drafts/MiniLinux/q2.png)

提示信息显示“你也许应该省略defined()，所以我们依照提示的，将373行的defined删除

```perl
if (!defined(@val)) { 
	@val = compute_values($hz); 
} 
```

改成

```perl
if (!@val) { 
	@val = compute_values($hz); 
} 
```


编译完成

```
Root device is (8, 7)
Setup is 13440 bytes (padded to 13824 bytes).
System is 4119 kB
CRC b577cbd4
Kernel: arch/x86/boot/bzImage is ready  (#1)

```



## Q3

https://blog.csdn.net/ymg2002abc/article/details/21652141



## 参考

https://leomos.me/compiling-and-emulating-linux-2-6-kernel-on-ubuntu-18-04-part-1



<https://blog.csdn.net/gouxf_0219/article/details/82018970>