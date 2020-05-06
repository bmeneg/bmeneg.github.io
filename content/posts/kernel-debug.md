---
title: "Kernel Debugging"
author: "Bruno Meneguele"
date: 2020-02-20
draft: true
---

$ ./scripts/config -d COMPILE\_TEST -e DEBUG\_KERNEL -e DEBUG\_INFO
$ make vmlinux
$ gdb vmlinux

Capture the Code: line from crash dump output

```
$ cat crash-code.log | ./scripts/decodecode

Code: 67 8f 00 48 8b 3d 50 e6 34 01 48 81 ff c0 a3 44 94 74 69 48 8d 6f f8 eb 11 48 8b 7d 08 48 8d 6f f8 48 81 ff c0 a3 44 94 74 52 <39> 5d 40 75 ea e8 06 83 3f 00 84 c0 74 0f 48 8b 55 08 48 8b 45 10
All code
========
   0:	67 8f 00             	popq   (%eax)
   3:	48 8b 3d 50 e6 34 01 	mov    0x134e650(%rip),%rdi        # 0x134e65a
   a:	48 81 ff c0 a3 44 94 	cmp    $0xffffffff9444a3c0,%rdi
  11:	74 69                	je     0x7c
  13:	48 8d 6f f8          	lea    -0x8(%rdi),%rbp
  17:	eb 11                	jmp    0x2a
  19:	48 8b 7d 08          	mov    0x8(%rbp),%rdi
  1d:	48 8d 6f f8          	lea    -0x8(%rdi),%rbp
  21:	48 81 ff c0 a3 44 94 	cmp    $0xffffffff9444a3c0,%rdi
  28:	74 52                	je     0x7c
  2a:*	39 5d 40             	cmp    %ebx,0x40(%rbp)		<-- trapping instruction
  2d:	75 ea                	jne    0x19
  2f:	e8 06 83 3f 00       	callq  0x3f833a
  34:	84 c0                	test   %al,%al
  36:	74 0f                	je     0x47
  38:	48 8b 55 08          	mov    0x8(%rbp),%rdx
  3c:	48 8b 45 10          	mov    0x10(%rbp),%rax

Code starting with the faulting instruction
===========================================
   0:	39 5d 40             	cmp    %ebx,0x40(%rbp)
   3:	75 ea                	jne    0xffffffffffffffef
   5:	e8 06 83 3f 00       	callq  0x3f8310
   a:	84 c0                	test   %al,%al
   c:	74 0f                	je     0x1d
   e:	48 8b 55 08          	mov    0x8(%rbp),%rdx
  12:	48 8b 45 10          	mov    0x10(%rbp),%rax
```

and then:

```
$ (gdb) disassemble /s __exit_umh,+0x42
Dump of assembler code from 0xffffffff810fbd50 to 0xffffffff810fbd90:
kernel/umh.c:
warning: Source file is more recent than executable.
728
   0xffffffff810fbd50 <__exit_umh+0>:   callq  0xffffffff81a019c0 <__fentry__>

729     void __exit_umh(struct task_struct *tsk)
730     {
   0xffffffff810fbd55 <__exit_umh+5>:   push   %rbp
   0xffffffff810fbd56 <__exit_umh+6>:   push   %rbx
   0xffffffff810fbd57 <__exit_umh+7>:   mov    0x4b8(%rdi),%ebx

731             struct umh_info *info;
732             pid_t pid = tsk->pid;
   0xffffffff810fbd5d <__exit_umh+13>:  mov    $0xffffffff8244a3a0,%rdi
   0xffffffff810fbd64 <__exit_umh+20>:  callq  0xffffffff819f24b0 <mutex_lock>

733
   0xffffffff810fbd69 <__exit_umh+25>:  mov    0x134e650(%rip),%rdi        # 0xffffffff8244a3c0 <umh_list>
   0xffffffff810fbd70 <__exit_umh+32>:  cmp    $0xffffffff8244a3c0,%rdi
   0xffffffff810fbd77 <__exit_umh+39>:  je     0xffffffff810fbde2 <__exit_umh+146>
   0xffffffff810fbd79 <__exit_umh+41>:  lea    -0x8(%rdi),%rbp
   0xffffffff810fbd7d <__exit_umh+45>:  jmp    0xffffffff810fbd90 <__exit_umh+64>
   0xffffffff810fbd7f <__exit_umh+47>:  mov    0x8(%rbp),%rdi
   0xffffffff810fbd83 <__exit_umh+51>:  lea    -0x8(%rdi),%rbp
   0xffffffff810fbd87 <__exit_umh+55>:  cmp    $0xffffffff8244a3c0,%rdi
   0xffffffff810fbd8e <__exit_umh+62>:  je     0xffffffff810fbde2 <__exit_umh+146>

734             mutex_lock(&umh_list_lock);
   0xffffffff810fbd90 <__exit_umh+64>:  cmp    %ebx,0x40(%rbp)
End of assembler dump.
```

We know by the decoded crash dump code that the faulty instruction is the `cmp %ebx,0x40(%rbp)`, also it's possible to
note that `%ebx` was previously set to `0x4b8(%rdi)` in `<__exit_umh+7>`, where `%rdi` is the `tsk` parameter and
`0x4b8` is the offset to `pid` field within `struct task_struct`, as we can see as follows:

```
$ echo "ibase=16; 4B8" | bc
1208

$ pahole -C task_struct | grep -m 1 pid
        pid_t                      pid;                  /*  1208     4 */
```

Now, checking the `%ebx` value got from the crash dump output its value is `9e`, being `158` in decimal. And considering
the crash dump occurred during an exit code we still can see the initialization output of our module:

```
$ dmesg | grep rztee
[    2.522648] rztee: Ring 0 TEE module initialized                            
[    2.523257] rztee: user app called with pid: 158
```

Confirming the PID of our userspace app launched by UMH infrastructure.

With that in mind we know that the issue occured with the second argument of the `<__exit_umh+64> cmp
%ebx,0x40(%rbp)`, where the value on `0x40(%rbp)` is being dereferenced. Again, if we pick the value for `%rbp` from
crash dump output, `00000002001ffff8`, we can check the that summing the value there present with `0x40` results in
the value presented in the BUG line, stating it couldn't be dereferenced.

Well, it seems `umh_list` doesn't contain any `umh_info` structure there placed, which should have, since we've
created a valid userspace process with a valid PID. Why is that there then?
