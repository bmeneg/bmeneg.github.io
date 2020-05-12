---
title: "Kernel debugging - memory case"
author: "Bruno Meneguele"
date: 2020-05-11
---

In the [last post](/post/kernel-debugging-with-qemu) I showed the first steps
to setup a development/debugging environment for working with Linux Kernel,
which has been the "perfect" (read it as: the one that have worked) setup so
far to me. But something was lacking at the end of the post... a bit of the
real taste of that. Because of that, in this post I'm going to show you a
quick walkthrough using one of my recent projects into kernel.

The project itself doesn't really matter, one day I'm going to share it with
you guys, but today the context is somewhat pretty simple: I have a userspace
application, that got launched by a mechanism called _User Mode Helper_ (UMH)
from within the kernel, which shares memory with the kernel directly; in other
words, I have a shared memory region that is shared between the kernel and a
user program running independently.

This region is first requested by the user program by opening and mmap'ing a
character device file that I called `/dev/umg_shm`, then the kernel allocates
a reserved region (that doesn't get swapped) within its own virtual memory
space and gives read and write access to whomever requested that. Albeit the
region is fully shared (read and write access) for both sides, a
synchronization protocol must be applied to avoid any data race among them.

To start with, we need to make sure both kernel and user processes are indeed
writing to the very same region and perform a quick "healthy check" to make
sure both can read and write data to the region without any surprises.

> NOTE: we are not too concerned about security issues right now, that's
> for another discussion.

> NOTE²: the code here being presented aren't fully implemented and you
> won't find some parts of it in upstream kernel, since it's part of a
> personal project. Whenever some specific information can't be find in
> upstream I'm going to give you guys the context around that.

Lets start using the GDB session we created in the [previous
post](/post/kernel-debugging-with-qemu/):

```shell
$ gdb vmlinux
...
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x000000000000fff0 in exception_stacks ()
(gdb) hbreak drivers/char/umh_shmem.c:51
Hardware assisted breakpoint 1 at 0xffffffff814d01b0: file drivers/char/umh_shmem.c, line 52.
(gdb) continue
Continuing.

Breakpoint 1, shmem_mmap (file=0xffff88801d7f8b00, vma=0xffff88801f98f540) at drivers/char/umh_shmem.c:52
warning: Source file is more recent than executable.
52      {
(gdb) l
47              .close = vma_remap_close,
48              .fault = vma_remap_fault,
49      };
50
51      static int shmem_mmap(struct file *file, struct vm_area_struct *vma)
52      {
53              int err;
54              struct umh_shmem *shm_info = &(current->umh_info->ipc.shm_info);
55              char *mem = (char *)kzalloc(UMH_SHM_LEN, GFP_KERNEL);
56              phys_addr_t data;
(gdb)
```

As I said before, the shared memory is being exposed via a char device in
`/dev/umh_shm`, which is created in the `_init` function on
`drivers/char/umh_shmem.c`. Besides that, this file also holds all file
operations available for the char dev, including the `open()` and `mmap()` and
others. A way to check the memory being actually returned by the mmap() call
is by setting a breakpoint in there.

## Side note about UMH

You might be wondering the reason for setting a breakpoint at kernel side
instead of user program, I mean, why would I debug kernel image rather than
user program? The reason for that is related to how UMH works, which deserves
a whole post for itself, but just to give you a better context: UMH processes
are created through a direct kernel call to the executable object. In my case,
the object is actually compiled withing kernel image, written as a userspace
program statically linked to glibc. Then an internal kernel module calls the
UMH and passes the start and end addresses, where this executable is placed
within kernel image, and the length of this object. Thus UMH creates a user
space thread, via a call to `kernel_thread()` and `do_execve()` directly to
it.

Of course, there are other action in-between my module call to UMH and the
actual `execve()` into my userspace program's `main()` function, a whole
environment setup is required, but the program is then launched as a child of
the kernel's _init_ process.

Could I pass to GDB the exact place within kernel image to start the debug
process? Yes, probably, but considering the workflow is really tied to the
kernel internal behavior I think it's easier to understand and solve issues
looking from lower layers perspective.

## Getting back to the debugging session

First of all, we can see that both arguments of the function `shmem_mmap` are:

```shell
file=0xffff88801d7f8b00
vma=0xffff88801f98f540
```

If you know the code, or you're following it, you know that any `mmap()` call
has the arguments of type `struct file *` and `struct vm_area_struct *`
respectively. With that and, possibly, with a small step further we can
retrieve some interesting information. For instance:

```shell
(gdb) p *(struct dentry *)file.f_path.dentry
$7 = {d_flags = 5242888, d_seq = {sequence = 2}, d_hash = {next =
0xffff88801d132a88, pprev = 0xffff88801ddd4ad0}, d_parent = 0xffff88801d004300,
d_name = {{{hash = 2927770382, len = 7}, hash_len = 32992541454}, 
name = 0xffff88801cec8578 "umh_shm"}, d_inode = 0xffff88801cb4b630, d_iname = "umh_shm", 
'\000' <repeats 24 times>, d_lockref = {{lock_count = 8589934592, {lock =
{{rlock = {raw_lock = {{val = {counter = 0}, { ...
...
(gdb) p $7.d_name.name
$8 = (const unsigned char *) 0xffff88801cec8578 "umh_shm"
(gdb)
```

For each object feedback GDB gives it generates a variable to which you can
refer later, like the `$7` there. And with output `$8` we know the file being
passed as the argument of `mmap()` is indeed the one we were expecting
`umh_shm`. If you're still not sure about that, you can dig even further to
check if the file is indeed referring to a char device:

```shell
(gdb) p /x *(struct inode *)file.f_inode
$9 = {i_mode = 0x2180, i_opflags = 0xd, i_uid = {val = 0x0}, i_gid = {val = 0x0},
i_flags = 0x0, i_acl = 0x0, i_default_acl = 0x0, i_op = 0xffffffff82015f80,
i_sb = 0xffff88801d4c9000, i_mapping = 0xffff88801cb4b798, i_security = 0xffff88801f8ccd20,
i_ino = 0x275f, { ...
...
(gdb)
```

That's indeed a huge amount of information that we could actually gather about
the file's `inode`, but for now, the only thing that really interests
us is the `i_mode` number: `0x2180`. Checking on kernel's documentation we can
see that this number means:

* 0x80
  - S\_IWUSR (Owner may write)
* 0x100
  - S\_IRUSR (Owner may read)
* 0x2000
  - S\_IFCHR (Character device)

So it seems our file has been setup correctly.

Now lets investigate the memory region we are sharing between kernel and
userspace:

```shell
(gdb) p /x *(struct vm_area_struct *)vma
$14 = {vm_start = 0x7fe6c24bc000, vm_end = 0x7fe6c24bd000, vm_next = 0x0,
vm_prev = 0x0, vm_rb = {__rb_parent_color = 0x0, rb_right = 0x0, rb_left =
0x0}, rb_subtree_gap = 0x0, vm_mm = 0xffff88801d4cc400, vm_page_prot = {pgprot
= 0x8000000000000025}, vm_flags = 0xd1, shared = {rb = { __rb_parent_color =
0x0, rb_right = 0x0, rb_left = 0x0}, rb_subtree_last = 0x0}, anon_vma_chain =
{next = 0xffff88801f98f5b8, prev = 0xffff88801f98f5b8}, anon_vma = 0x0, vm_ops
= 0xffffffff8200ab20, vm_pgoff = 0x0, vm_file = 0xffff88801d7f8b00,
vm_private_data = 0x0, swap_readahead_info = {counter = 0x0}, vm_policy = 0x0,
vm_userfaultfd_ctx = {<No data fields>}}
(gdb)
```

First of all, we need to remember that we only will understand what these
attributes means if we understand the code it actually covers, and the first
thing about `struct vm_area_struct` we need to know is that it is the
information about a specific _virtual memory area_, but not a conjunction of
areas or pages or physical memory or ... it's solely about a single piece of
virtual memory, with a specific _start_ (`vm_start`) and an _end_ (`vm_end`).
Another important point to know is that the `struct vm_area_struct` passed to
a `mmap()` call refers to a userspace area within the program's virtual
memory. So, what do this all mean?

* it means the user's virtual address `0x7fe6c24bc000` and `0x7fe6c24bd000`
  are the inferior and superior limits for mapping the shared memory,
  respectively.
* if we subtract theses addresses we have the length of the memory being
  mapped in userspace program's memory: `0x1000`, which is equivalent to 4096
  bytes, which is also equivalent to the size of a _memory page_ (that was
  defined by myself, that's not always the case.

These are the most important things to be known for now, considering the
memory was actually mapped yet; these are the information passed by the user
application in the `mmap()` call.

So, moving the code forward, we now expects the kernel dynamically allocates a
fresh memory on its own virtual space to then hand off to the user. Usually
this operation looks like (considering we're speaking about a page size
allocation):

```C
void *mem = kzalloc(PAGE_SIZE, GFP_KERNEL);
```

Unfortunately this call is usually optimized by the compiler, forcing us to
check for processors registers directly:

```shell
...
(gdb) n
55         char *mem = (char *)kzalloc(UMH_SHM_LEN, GFP_KERNEL);
(gdb) ni
...
(gdb) disassemble
...
   0xffffffff814d01ec <+60>:    callq  0xffffffff811d3d30 <kmem_cache_alloc_trace>
=> 0xffffffff814d01f1 <+65>:    mov    $0x80000000,%edx
   0xffffffff814d01f6 <+70>:    add    %rdx,%rax
...
```

To know which register is used for which purpose is completely architecture
dependent. So it's completely normal to not remember some definitions if
you're not used to work directly with assembly or to the ABI of the system.
The whole example I'm presenting here is performed in a x86\_64 architecture,
thus the return value of a `call` instruction is kept in the `%rax` register.

```shell
(gdb) p /x $rax
$18 = 0xffff88801f973000
(gdb) x/12 $rax
0xffff88801f973000:     0x00000000      0x00000000      0x00000000      0x00000000
0xffff88801f973010:     0x00000000      0x00000000      0x00000000      0x00000000
0xffff88801f973020:     0x00000000      0x00000000      0x00000000      0x00000000
(gdb)
```

All bytes are zero because the allocation was performed with `kzalloc()`,
which zerofy the entire allocated memory before handing it off to the caller.
And, by calling `virt_to_phys()` we know where in physical memory this virtual
address resides: `0x1f973000`.

This memory has the exact same size as the one requested by the user (although
only 12 words were shown in the command) and will be mapped into user's
virtual memory through the `remap_pfn_range()` function, which adds the new
memory area to the list of other memory areas owned by this specific process,
known as `struct mm_struct`. The process of mapping goes down to the point
where _page table entries_ are manipulated, which is **somewhat** out of the
scope of this post, but I would love to bring more content about it soon.

## Watching the communication

Now that we have established the shared memory between kernel and userspace we
need to keep watching the communication between these two parts. To do that we
can setup a _watchpoint_ to the memory shared for any modification there
happening. To know what memory address to use you basically need to think
that: you're debugging the kernel process as a whole, so you have direct
access to its virtual memory, but not to userspace memory nor physical
address. With that said, we can easily setup a _watchpoint_:

```shell

(gdb) awatch *0xffff88801f973000
Hardware access (read/write) watchpoint 2: *0xffff88801f973000
(gdb) c
Continuing.

Program received signal SIGTRAP, Trace/breakpoint trap.
__umh_health_check (arg=<optimized out>) at security/rztee/rztee_core.c:34
34              strncpy(kreq->payload, "hello", 6);
(gdb) l
30
31              pr_info("rztee: healthy checking userspace program\n");
32              kreq = (struct rztee_kreq *)shm_info->mm_map;
33              kreq->id = 2;
34              strncpy(kreq->payload, "hello", 6);
35              strncpy(kreq->pl_hash, "world", 6);
36              return 0;
37      }
38
(gdb)
```

This specific point of code isn't upstream, it's part of a personal project.
But `shm_info->mm_map` is a pointer to the memory we're sharing, so any
modification to that will be visible to our user program. And `struct
rztee_kreq` has the following layout:

```C
struct rztee_kreq {
	int id;
	char *payload[1024];
	char *pl_hash;
};
```

Right after the assignment `kreq->id = 2` the watchpoint is hit and thus we
can inspect what have changed in the memory:

```shell
(gdb) x/gx 0xffff88801f973000
0xffff88801f973000:     0x0000000000000002
(gdb)
```

If we let the next two lines to be executed, the resulting memory is:

```shell
(gdb) x/2gx 0xffff88801f973000
0xffff88801f973000:     0x6c6c656800000002      0x000000000000006f
(gdb) x/2gx (0xffff88801f973000+1024)
0xffff88801f973400:     0x6c726f7700000000      0x0000000000000064
```

And considering we know the data structure behind this data, plus some ASCII
encoding:

```C
int id = 2
char *payload[1024] = "hello"
char *pl_hash = "world"
```

## Switching to userspace

Now, jumping directly to userspace, you only need a couple of calls to get the
shared information:

```C
int main(void)
{
	int shm_fd;
	void *shm_mm;
	struct rztee_kreq *kreq;

	shm_fd = open("/dev/umh_shm", O_RDONLY);
	if (shm_fd == -1) {
		// ... check error
	}
	shm_mm = mmap(NULL, 4096, PROT_READ, MAP_SHARED, shm_fd, 0);
	if (shm_mm == MAP_FAILED) {
		// ... check error
	}

	kreq = (struct rztee_kreq *)shm_mm;
	printf("%d %s %s\n", kreq->id, kreq->payload, kreq->pl_hash);

	return 0;
}
```

And as a result: `2 hello world`.

## Conclusion

Well, this post doesn't show you how to setup a shared memory between user and
kernel space, of course, but it gives you a general idea on how to, somewhat,
track the memory manipulation on kernel side in a real case using **qemu**
plus **gdb**. It a pretty specific case, but if you take a few minutes to
think about you're going to realize there're more situations that can be
actually debugged in the exact same way to find _null pointer dereference_,
_double free_, _page faults_, and other issues that are commonly seen in
kernel issues and which are generally not related to the memory management
subsystem, but with another piece of code screwing things up while
manipulating their own memory mess.

Kernel development are sometimes seen as a huge beast, but with some basic
debugging knowledge you can actually walkthrough parts you would like to
better understand. Kernel debugging can also seem some "beast", but once you
start digging into it you automatically start learning assembly language,
understand memory management and usage, basic data structures used within
kernel, and many other things. There are no magics... **_cough cough_**...
sometimes though.

I really hope this guidelines, for such specific case, helps you somehow. Feel
free to ping me on IRC or Twitter!

Many thanks!
