+++
title = "GDB 步进调试内核时遇到中断的解决办法"
date = 2023-05-22T21:23:11+08:00
author = "NKID00"
tags = ["GNU/Linux", "Kernel", "GDB"]
hideComments = false
draft = false
+++

GDB 步进调试内核时常会遇到中断，此时执行流会到中断处理程序中而难以调试，比如：

```
(gdb) n
__sysvec_apic_timer_interrupt (regs=<optimised out>) at ../arch/x86/kernel/apic/apic.c:1111                                                      │portal on /run/user/1000/doc type fuse.portal (rw,nosuid,nodev,re
1111            trace_local_timer_entry(LOCAL_TIMER_VECTOR);
```

## 笨办法

x86_64 的 Linux 内核的一般中断处理程序最后会在 `native_irq_return_iret` 符号处执行 `iretq` 指令返回到中断前的代码：

```
(gdb) x/i native_irq_return_iret
   0xffffffff81a0150a <common_interrupt_return+90>:     iretq
```

因此我们可以在这里下一个一次性断点：

```
(gdb) b native_irq_return_iret
Breakpoint 1 at 0xffffffff81a0150a: file ../arch/x86/entry/entry_64.S, line 712.
(gdb) en br once 1
(gdb) i b
Num     Type           Disp Enb Address            What
1       breakpoint     dis  y   0xffffffff81a0150a ../arch/x86/entry/entry_64.S:712
```

继续到这个断点再执行下一条指令：

```
(gdb) c
Continuing.
Breakpoint 1, common_interrupt_return () at ../arch/x86/entry/entry_64.S:712
712             iretq
(gdb) ni
```

然后就返回到中断前的代码了，可能要再执行几次 `fin` 才能返回到最开始步进的地方。再次遇到中断可以再启用这个断点。

## 聪明办法

后来发现直接在下一条语句下个临时断点就行了：

```
(gdb) c
Continuing.

Breakpoint 1, fsnotify (mask=mask@entry=131072, data=data@entry=0xffff88800621ef50, data_type=data_type@entry=1, dir=dir@entry=0xffff888003ea86f0, file_name=file_name@entry=0x0, inode=0xffff88800502b080, cookie=0) at ../fs/notify/fsnotify.c:483
483     {
(gdb) n
484             const struct path *path = fsnotify_data_path(data, data_type);
(gdb)
__sysvec_apic_timer_interrupt (regs=<optimised out>) at ../arch/x86/kernel/apic/apic.c:1111
1111            trace_local_timer_entry(LOCAL_TIMER_VECTOR);
(gdb) tb fsnotify.c:485
Temporary breakpoint 2 at 0xffffffff81293bf0: file ../fs/notify/fsnotify.c, line 486.
(gdb) c
Continuing.

Temporary breakpoint 2, fsnotify (mask=mask@entry=131072, data=data@entry=0xffff88800621ef50, data_type=data_type@entry=1, dir=dir@entry=0xffff88
8003ea86f0, file_name=file_name@entry=0x0, inode=0xffff88800502b080, cookie=0) at ../fs/notify/fsnotify.c:486
486             struct fsnotify_iter_info iter_info = {};
```

好简单。
