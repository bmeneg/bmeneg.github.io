# x86-64 architecture: stack, memory, ..., internals: part 1 #

(yeah, it'll be a series of posts... I hope)

The idea behind this series of posts is to present to anyone whom may care about 
the  low-level stuff related to the x86-64 architecture, without any kind of
order or specific subject. Yeah, I really like to learn new things and right now
the subject that seems to be most convenient to be learned (considering my job
and perhaps my future master degree) is about the beloved Intel's architecure.
Anything I found really interesting, or at least allowed me to have enjoyable
moments I'll share with you. But like I said, without any specific order, I'm
sorry for that.

## Post's epilogue ##

I never touched x86 CPU-family's internals, because of that, be aware that my
presentations might have not the most precise information you can find in the
internet. But I promise to try to stay only with the correct ones. Any wrong or
misunderstood infos, please, feel free to get in touch!

Something I recommend you to do during these posts about "low-level" stuff is to
try to follow the examples by writing your own code, since it might be a little
bit different from my results or something like that. Whenever you can doubt
about what I've said. I think this is the best way to be confident about your
own knowlegde.

Another thing I suggest is to use the 
"GDB Dashboard":https://github.com/cyrus-and/gdb-dashboard peace of code, in
case you use GDB as your debugger. I never had used it as I should, but nowadays
is a common practice. 

## Ok, let's finally start ##

I started to read about x86 assembly once I start to play around SGX security
mechanism, which is available only on Intel i7 6th generation and onwards, the
idea behind it was to study it before actually start a master degree over this
subject, which I really would like to start as soon as possible.

At the time of writing this post SGX mechanism wasn't have being accepted on
Kernel mainstream, but some noise was being made on LKML, take a look at 
https://lkml.org/lkml/2016/4/25/717, and was there that I started to face so
many letter and acronyms that I saw I had so many things I should learn even
before start to try to understand SGX terminology, well not just for SGX, but
for everything around it, in other words, any CPU mechanisms and the Operating
System itself.

Yes, I decided to start from its beginning, Assembly language, since every time
you try to read something on memory, when some Kernel oops/panic just pop up,
you should understand a bit about it, so why not?

I'll present you the steps I made to "pretend" I learned.

## The x86(-64?) assembly ##

There are so many material around it, and so much more around x86 (32 bits) that
I decided to write the x86-64 way, with its registers naming convation, data
type standard and everything I can explain through a 64bits system. In my case
I'm using an Linux SO (Fedora to be more specific, but with no reason) and GCC
6.3.1 version, which would change a lot the underneath generated code for some
program we write if changed to something like CLANG, but the assembly language
itself is exactly the same, then again, it's your call.

When writing code to x86-64 you might face some 32bits instructions and/or
naming convention, like working with 32b registers instead of 64b ones. That's
because 64b is somewhat compatible to its predecessor and when 64bits values
aren't used or the action you want to perform just works using 32b instructions
then is will be used (less bytes to handle? maybe).

### Basic standards ###

IA (Intel Archtecture) has some basic registers that is a nice idea to present
before go ahead:

(naming will be presented as 32bits(64bits))

-   general purpose registers: EAX (RAX), EBX (RBX), ECX (RCX), EDX (RDX), ESI
    (RSI), EDI (RDI), and some 64 bits only: R8~R15
-   program/instruction counter: EIP (RIP), yeah, I know you made the joke about
    RIP like everyone does.
-   stack pointer: ESP (RSP)
-   stack base frame pointer: EBP (RBP)
-   segment selectors: CS, DS, SS stands for Code and Data and Stack segment,
    ES, FS and GS are all used as data segment like DS, but there are some
    differences I'll mention when necessary
-   float pointer enabled registers: ST0-ST7
-   flag control register: EFLAGS (CFLAGS)

There are a bunch of others, but for now let's stick around these, as I said
before.

Note: ESP (RSP) and EBP (RBP) are considered general purpose registers by the
architecture, but notably they have specific meaning and thus aren't really used
as general ones.

Beyond that there is the standard regarding data types which isn't directly
related to the architecture itself, but with the operating system and compiler.
This standard defines how many bytes data types will use in memory, for example:
Linux x86 used ILP32 standard which stands for _int_, _long_ and  _pointer_
types as 32 bits each, but when on Linux x86-64 the standard used is LP64, in
other words, _long_ and _pointer_ types are 64 bits long while _int_ still is 32
bits long. 

This is important to know because if you are reading this post over an Windows
operating system the standard changes, thus the code generated by the same GCC
compiler (for Windows, of course) is also different from the one generated to
Linux. Windows x86-64 follows LLP64, which creates the _long long_ type, being
this one and _pointer_ 64 bits long, while  either _int_ and _long_ continues as
32 bits. This was used to maintain full compatibility with 32 bits old
applications. There isn't a war about "which one is better", the idea is to be
consistent to your target operating system conventions.

For more info about this convention take a loot [here][lp64-convention].

### Endianess ###

A small but really important thing I must say is about the archtecture
endianess, and what is that?

Well, short answer: endianess is the order that bytes are arranged into memory.
There are two type of endianess: _big endian_ and _little endian_, that said
let's present some examples:

Consider a 2 bytes word: 0x03 0x02 0x01 0x00 and that our imaginary archtecture
grows its memory address from left to right.

When in big endian the most significant byte (MSB) will be stored in the lowest
memory address:

                                  (grows <--)
        memory address:    4     3     2     1     0  
        memory slots:   | ... | x00 | x01 | x02 | x03 |

If we consider an archtecture that stores 2 bytes of data in each memory slot it
would be: 

                                  (grows <--)
        memory address:            1        0  
        memory slots:   | ... | x01x00 | x03x02 |

When in little endian the least significant byte (LSB) is stored in the lowest
memory address:

                                  (grows <--)
        memory address:    4     3     2     1     0  
        memory slots:   | ... | x03 | x02 | x01 | x00 |

And for 2 bytes memory data:

                                  (grows <--)
        memory address:            1        0  
        memory slots:   | ... | x03x02 | x01x00 |

Well, IMHO little endian seems to be easier to read when the memory address
grows from left to right, right? Fortunately our imaginary architecture follows
this standard that x86(-64) ALSO follows. In short, IA32(64) is little endian
and its memory addresses grows from left to right. 

The table below was extracted from Intel Software Development Manual 1st Volume,
which defines exactly what we discussed so far about endianess.

PICK THE TABLE AS IMAGE!!!!! 

## x86-64 ABI ##

From now on things start to become a bit interesting, even some code might be
written if you wish. We are about to discuss the Application Binary Interface
(ABI) to the x86 architecture, which is really important you understand to
continue thoughout this post's goal.

!!! STUDY MORE ABOUT STACK BEFORE CONTINUE !!!!


!!! INTERESTING REFERENCES TO POINT TO !!!
[lp64-convention]: <http://www.unix.org/whitepapers/64bit.html>
[x86-64 abi]: <https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf>
[stack-frame]: <http://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64/#id3>
[write-some-asm]: <http://nickdesaulniers.github.io/blog/2014/04/18/lets-write-some-x86-64/>
[different-abi]: <http://nelhagedebugsshit.tumblr.com/post/84342207533/things-i-learned-writing-a-jit-in-go>
