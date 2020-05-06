---
title: "Kernel Debugging with QEMU"
author: "Bruno Meneguele"
date: 2020-05-06
---

## Intro

Ok, this is a real quick overview on my kernel development environment setup,
that some has asked about. To be honest, I'm not sure how helpful it can be to
every single people working on each part of the kernel, but for sure it'll
help to speed things up a lot for those starting on kernel space or actually
working over some core subsystem that doesn't really require a specific
hardware to test that.

If you search around the web you're going to found plenty of materials about
this very same subject, and to be honest, I've based great part of my setup on
lots of them. At the same time, it's important to open a note here: that's not
the only option to actually develop and debug your kernel, it really depends
on your own requirements, if you don't need a rootfs to be in place, so you're
fine following the steps I'm going to present, on the other hand you can try
something more sophisticated like
[virtme](https://git.kernel.org/pub/scm/utils/kernel/virtme/virtme.git), which
I don't have much experience, but seems pretty helpful as well.

## My environment

Nowadays I've been using Fedora as my Linux distribution for daily work and,
consequently, one of the tools I'm going to pinpoint here are specific to
Fedora-like distros, like CentOS and RHEL - although I've seen OpenSuSE and
Gentoo people commenting on them, so perhaps it's getting attention?

The core utilities are _QEMU_ for virtualization and _dracut_ for creating the
ramdisk we're going to use as filesystem. Other than that, there are some
kernel CONFIG options and arguments I really recommend you enable to get
better debugging information embedded with the kernel image, avoiding too many
information being removed during debugging session due to compilation
optimizations.

## Installation step

First of all, I'm going to assume you're building a kernel module in-tree, so
the commands being used doesn't really help for getting your out-of-tree
modules built properly, make sure to check how to build out-of-tree modules
before getting here.

_dracut_ probably is going to be already installed in your system, so you can
skip that from _dnf_ call, otherwise:

```bash
$ sudo dnf install qemu-system-x86 elfutils-libelf-devel dracut
```

It's also worth checking if you system has any virtualization technology
enabled, it'll improve your experience by a great amount:

```bash
$ lscpu | grep Virt
Virtualization:                  AMD-V
... or ...
Virtualization:                  VT-x
```

If so, make sure you have **kvm** module enabled in your system and also to
install **qemu-kvm** package:

```bash
$ lsmod | grep kvm
kvm_amd               114688  0
kvm                   802816  1 kvm_amd
... or ...
kvm_intel             114688  0
kvm                   802816  1 kvm_intel

$ sudo dnf install qemu-kvm
```

## Compilation step

Compiling the kernel can be somewhat tricky to get it right the first time.
You can use a brand new configuration file or the exact same one your local
installation is running.

If you wish to run the exact same kernel configuration you have running:

```bash
$ cp /boot/config-$(uname -r) <kernel-src>/.config
$ make olddefconfig
$ make -j$(nproc)
$ make -j$(nproc) modules_install
```

On the other hand, you can try something completely new:

```bash
$ make x86_64_defconfig
$ make kvmconfig
$ make -j$(nproc)
$ make -j$(nproc) modules_install
```

But in both cases something I recommend you to try is to enable the debugging
options you have at disposal (they are many!), which I'm going to present some
of them I suggest:

```bash
$ ./scripts/config -e DEBUG_INFO -e DEBUG_KERNEL -e DEBUG_INFO_DWARF4
$ ./scripts/config -e DEBUG_SECTION_MISMATCH -e DEBUG_OBJECTS -e DEBUG_OBJECTS_WORK -e DEBUG_VM
$ ./scripts/config -e GDB_SCRIPTS -e HEADERS_INSTALL
```
The first line of **DEBUG_** options are the standard ones I suggest you have
enabled regardless the portion of the kernel you're touching at. The second
line is basically things useful for me, that makes sense for the type of
feature I'm currently working on, which may significantly change to your case.
And the third one is also for everyone, both options will provide you more
in-depth information during gdb debugging session.

Something good to remember is the way **make** works with relation to its
incremental build process: if you don't call something cleaning option of
**make**, like _distclean_, anytime you re-execute **make** it'll check what
actually have changed since the last time you ran that and only recompile the
source codes that got updated. In this way the amount of time you spend
recompiling code is decreased by some orders of magnitude, at least.

## Getting RAM disk ready

In short, a RAM disk is a filesystem dynamically placed in you RAM memory in
boot time that contains the basic stuff needed to get your real filesystem
running with the first processes needed to get your whole system running as
expected, like the _init_ process.

_dracut_ is the tool used by Fedora, and its relatives, to create the RAM disk
known as _initramfs_. Whenever this ram disk gets executed to and doesn't find
the real rootfs to continue the boot process, or if it finds an issue during
it's own code path, it'll hand off to the user a terminal where some basic
tools are available, allowing basic, but useful, recovering and or
modifications to the system in the hope the user can get the system running
manually or at least gather the information he needs to seek some help.

That's the goal of our mechanism here, not to but completely until we reach
the real rootfs image, but to the _maintenance terminal_ we have at disposal
after _initramfs_ is fully booted.

To create this ramdisk you first need to have in mind the way your modules was
built: was it set as a built-in module or as a loadable module? If it's an
out-of-tree module so you know the it's a loadable module, while if it's an
in-tree module you had to choose between "=y" or "=m" in the configuration
file. This information is important because you need to decide if you need to
import or not you module inside the ramdisk. Built-in modules are linked
directly inside kernel image, known as _vmlinux_, while loadable modules are
separate files with **.ko** extensions that must be moved into the ramdisk
we're going to create before we actually run it.

If you've built your module as built-in, the only thing you need is to create
the ramdisk pointing to the kernel image generated by the compilation step of
the kernel in question:

```bash
$ sudo dracut -v -f ramdisk.img <kernel-name>
```

Where the _kernel-name_ is the name of the folder that was created during the
`make modules-install` step.

On the other hand, if your module was built as a loadable one, you need to
import this driver file into ramdisk, it's done by passing folder path where
the **.ko** is located. For instance, assuming the path is
`/home/myuser/my-great-module/`

```bash
$ sudo dracut -v -f --include "/home/myuser/my-great-module" "/my-module" ramdisk.img <kernel-name>
```

In this way you're going to have your folder module copied into the ramdisk at
`/my-module`.

## Execution step

Ok, now it's time to get our compiled kernel plus the ramdisk we just created
and bundle them together to launch the virtualization machinery with that.
Just to make things clear, lets assume some "variables":

```
kernel_path = /home/myuser/my-kernel/
ramdisk_path = /home/myuser/my-ramdisk/
```

Now:

```bash
$ qemu-system-x86_64 \
	-kernel $kernel_path/vmlinux \
	-initrd $ramdisk_path/ramdisk.img \
	-append "console=ttyS0 nokaslr" \
	-nographic -enable-kvm -cpu host -m 512
```

The above command will launch `qemu-system-x86_64` using the our kernel and
ramdisk plus some additional parameters that you can fine the meaning pretty
easily checking `qemu-system-x86_64` manpage. But you don't need to worry
about these additional parameters for now; their usage there is basically to
improve the debugging experience and also to allow better performance.

As soon as you hit enter you're going to see that the whole kernel boot starts
popping up on the console screen, that's your custom kernel and ramdisk
booting. Whatever changes you've done to that, is going to be there. Once the
ramdisk finished its boot you're going to see something like:

```bash
Press Enter for maintenance
(or press Control-D to continue):
```

Hit enter and then load your module (if built as a loadable module):

```bash
:/root# insmod my-module/test.ko
```

## Debugging step

Moving forward, the debugging step is somewhat straightforward if you're
already used to GDB. The only thing that changes is the way you attach the
running kernel on `qemu` to the GDB session.

The first thing is to actually add a couple arguments to the `qemu` call:

```bash
$ qemu-system-x86_64 \
	-kernel $kernel_path/vmlinux \
	-initrd $ramdisk_path/ramdisk.img \
	-append "console=ttyS0 nokaslr" \
	-nographic -enable-kvm -cpu host -m 512 \
	-s -S
```

Yes, these last two `-s -S`, which make the execution wait on startup and also
open a connection port `:1234`. With that, now we can attach GDB to this
port and debug our kernel. But, before that lets add a special file that was
created during kernel compilation, due to the _GDB\_SCRIPTS_ option, to our
`~/.gdbinit` file:

```bash
$ echo "add-auto-load-safe-path $kernel_path/scripts/gdb/vmlinux-gdb.py" >> ~/.gdbinit
```

and finally start the debug:

```bash
$ gdb $kernel_path/vmlinux
...
(gdb) tagert remote localhost:1234
Remote debugging using localhost:1234
0x000000000000fff0 in exception_stacks ()
(gdb) lx-
lx-clk-summary        lx-device-list-bus    lx-fdtdump            lx-list-check         lx-symbols            
lx-cmdline            lx-device-list-class  lx-genpd-summary      lx-lsmod              lx-timerlist          
lx-configdump         lx-device-list-tree   lx-iomem              lx-mounts             lx-version            
lx-cpus               lx-dmesg              lx-ioports            lx-ps                 
(gdb) hbreak kernel_init
Hardware assisted breakpoint 1 at 0xffffffff81b07d8a: file init/main.c, line 1358.
(gdb)
```

All these `lx-*` commands were added by the `vmlinux-gdb.py` helper created in
compilation time. And from now on, you can follow the normal GDB commands to
debug the kernel, side by side with the console running the boot process.

I've plans to create a post related to the debug process itself, with some
tips and tricks I've been learning every single day. The Linux Kernel itself
has quite some complex constructs and optimizations, sometimes it's really
hard to follow along, but the best way I've found to understand it was by
debugging it.

HTH.
Many thanks.
