---
title: "awk - possibly the most underrated Unix tool"
author: "Bruno Meneguele"
date: 2019-08-19
---

Yes, that's right... the title for this post might be one of the biggest truths
we see nowadays about a tool in Unix space. Although the name doesn't mean
anything special that describes the tool - it stands for the surname of its
creators: Alfred Aho, Peter J. Weinberger e Brian Kernighan, and guess what, at
Bell Labs - it really shines and *helps* in a niche that many of us work
everyday: _text processing_.

Every day we see more about "big data" processing, "data science", "business
intelligence", "machine learning", ... but at the very fundamental steps for
each of this big words there are a common source: _text_, a *lot* of text.

Unfortunately we have seen many good tools getting ignored throughout the time
on behalf of other options that we can arguably say they are "fancier" or, in
other words, misleadingly pointed as "better". _awk_ is a great example of such
tool that got forgotten and is very unlikely to be considered in the current
days by the new developers swamped in buzzwords.

> NOTE: don't get me wrong, I'm not stating these other tools are actually
> worse or not good enough, what I'm saying actually is that AWK shouldn't be
> ignored or considered "obsolete" compared to these tools (like I've heard
> countless times about C being obsolete because isn't object-oriented, ...
> wait, what?! That's a subject for another post).

My goal with this post is to give a *really quick overview* of that tool, not to
convince you to use it anywhere, but just to let you know it still exists, it's
still used and it's still viable to some extent. Perhaps what I'm trying to say
is: _stop for a sec and try to ignore any other new framework or tool about
text processing, try to use what you already have and didn't know_.

## Basic structure

As many other Unix tools, _awk_ handles _lines_ as it input source; this lines
are then matched against the following rule:

```
<pattern> { <action> }
```

being `pattern` the match that triggers the `action`.

Although `pattern` can assume "any" value, there are two special ones: `BEGIN`
and `END`.

```awk
BEGIN { <action> }
END { <action> }
```

`BEGIN` triggers an _action_ before the first line is handled, while `END`
triggers an _action_ after the last line was processed, see:

```shell
$ ls -l
total 0
-r--rw-r--. 1 bmeneg bmeneg 0 ago 19 19:43 file1
-r--r--r--. 1 bmeneg bmeneg 0 ago 19 19:44 file2
-rwxrw-r--. 1 bmeneg bmeneg 0 ago 19 19:45 file3
-rw-rw-r--. 1 bmeneg bmeneg 0 ago 19 19:45 file4
$ cat ../my-awk-program
BEGIN { print "- START -" }
{ print "processing new input line ..." }
END { print "- DONE -" }
$ ls -l | awk -f ../my-awk-program
- START -
processing new input line ...
processing new input line ...
processing new input line ...
processing new input line ...
processing new input line ...
- DONE -
$
```

## Use cases

You already can imagine a couple of use cases for a system administrator or an
automated task, right? Lets try to print the file name and the file owner of
our current folder.

```awk
BEGIN { print "name\towner" }
END { print "- done -"}
{ print $9, "\t", $3 }
```

Now let's run this _awk_ program with our `ls -l` output:

```shell
$ ls -l | awk -f ../my-awk-program
name    owner
         
file1    bmeneg
file2    bmeneg
file3    bmeneg
file4    bmeneg
- done -
```

> NOTE: feel free to try adding `#!/usr/bin/awk -f` to the first line of the
> program and set exec permission to that and then:
>
```
$ ls -l | ../my-awk-program
```
>
> _awk_ is also an interpreter.

First, yes, you can put your `BEGIN` and `END` anywhere in the program, every
pattern is checked for each input (line) processed. And second, _fields_, the
ones I used there, `$3` and `$9`, can be thought as Bash or Perl function
parameters, but the values they hold are actually the _fields_ present in the
input. Consider _fields_ like _columns_ of a row (line), each number represents
the column number of that specific input being processed. But, unlike other
programming languages, the dollar sign `$` doesn't mean it's a _variable_,
which is also present in _awk_, but without `$`. Also, _fields_ aren't expanded
within double quotes and that's the reason you see both being placed around
`"\t"`.

Now, to demonstrate the use of real variables let me introduce you a more
elaborated example:

```awk
#!//usr/bin/awk -f

BEGIN {
    # variables can be declared anywhere
    my_name = "awbot"
    # remember, awk was made to handle text input
    print "Hey user, tell me your name: "
}
{
    # NF = number of fields
    # here we using it as a variable
    if (NF > 1) {
        # and here we are using NF value to access the NFth (last) field
        # creating: 'first + last name' standard
        user_name = $1" "$NF
    } else {
        user_name = $1
    }

    print "Hi", user_name "! My name is", my_name
    exit
}
END {
    # variables are shared between patterns
    print "Bye", user_name
}
```

*Wait, what is that??* Yes, that's right, _awk_ has all those things you know
from other languages: conditions, loops, arithmetic operations, variables, ...
you can indeed play around with the string being used as input to your program,
you can change the processing step depending the size of the input, or you can
even change that based on the string per se. For instance, add the following
rule right after the `BEGIN`:

```awk
/^$/ || /^quit\s*/  {
    print "Ok, quiting..."
    exit
}
```

It will make the program quit whenever you enter an empty line or the word
`quit`. As you may have thought, if we're talking about text processing it's
almost mandatory to know something about *regular expressions*, this is the
core engine running behind this whole language.

> NOTE: regular expression is something really useful and should be known by
> every programmer to some extent, but at the same time it's not always
> suitable for the job: it can turns into a really expensive operation. Make
> sure you know when it should or not be used.

You might be wondering what else can be done with _awk_, well, many things! It
accepts functions, it has builtin operations for substitution, and so on. The
amount of things you can do manipulating your input to give you a meaningful or
a more desirable output is countless. You can find _awk_ being used in many
places of Linux ecosystem, for instance, within the Linux Kernel sources,
preparing file names based on file contents within Makefiles or a bunch of
projects packaging schemes for handling dynamic versioning, translation,
changelogs, an so on.

To finish, a final example, based on something I run in one of my daily task at
my job: consider the Linux kernel patch log, you need to retrieve some
information from patch headers and output it in a different way, not possible
using git's own `--pretty` feature. The input would be like this:

```shell
$ git format-patch --first-parent -k --stdout v5.2..v5.3-rc5
From 028db3e290f15ac509084c0fc3b9d021f668f877 Mon Sep 17 00:00:00 2001
From: Linus Torvalds <torvalds@linux-foundation.org>
Date: Wed, 10 Jul 2019 18:43:43 -0700
Subject: Revert "Merge tag 'keys-acl-20190703' of
 git://git.kernel.org/pub/scm/linux/kernel/git/dhowells/linux-fs"

...

From 9787aed57dd33ba5c15a713c2c50e78baeb5052d Mon Sep 17 00:00:00 2001
From: Nathan Chancellor <natechancellor@gmail.com>
Date: Mon, 1 Jul 2019 11:28:08 -0700
Subject: coresight: Make the coresight_device_fwnode_match declaration's
 fwnode parameter const

...
```

This input goes on for about 14 patches.

And the _awk_ program, with some comments to help you understand what is going
on and also possibly with some issues, is as follows:


```awk
BEGIN {
    total_patches = 0
    total_proc_patches = 0
}
/^From / {
    # store the second field from input
    commit_id = $2 
    total_patches++
    # get next input, ignoring next patterns
    next
}
/^From: / {
    # split the whole input using '<' as delimiter
    nf = split($0, a, "<")
    # get everything after 7th char
    name = substr(a[1], 7)
    next
}
/^Date: / {
    date_dmy = $3" "$4" "$5
    next
}
/^Subject: / {
    # you also can recheck for a substring with regex
    # ignore merges and tagging commits
    if ($0 ~ "([Rr]evert)?.*\"?[Mm]erge.*") {
        next
    } else if ($0 ~ "Linu[xs] [0-9].[0-9]") {
        next 
    }    
    subject = substr($0, 10)
    total_proc_patches++
    # NR = number of records (inputs)
    subj_nr = NR
    next
}
{
    if (NR == subj_nr+1) {
        if ($0 ~ "^$") {
            # awk offers some builtin helper functions
            r = sprintf("[%s] %s\n%s\nby %s\n", date_dmy, commit_id, subject,
              name)
        } else if ($0 ~ "^\\s") {
            r = sprintf("[%s] %s\n%s%s\nby %s\n", date_dmy, commit_id, subject,
              $0, name)
        }
        print r
    }
}
END {
    print "-----"
    print "Total patches found:", total_patches
    print "Total patches manipulated:", total_proc_patches
}

```

And the output:

```shell
$ git format-patch --first-parent -k --stdout v5.2..v5.3-rc5 | awk -f mail.awk
[1 Jul 2019] 9787aed57dd33ba5c15a713c2c50e78baeb5052d
coresight: Make the coresight_device_fwnode_match declaration's fwnode parameter const
by Nathan Chancellor 

[18 Jul 2019] 40ef768ab6eecc1b51461a034274350b31fc29d1
Remove references to dead website.
by Dave Jones 

[30 Apr 2019] 618381f09cc15592bf3afe846c6a94e9bfcd9ce4
hexagon: switch to generic version of pte allocation
by Mike Rapoport 

[11 Jul 2019] 8cf66504210d308a35cca35fe9c310b1241f9fa7
iommu/amd: fix a crash in iova_magazine_free_pfns
by Qian Cai 

[31 Jul 2019] 1b7e816fc80e668f0ccc8542cec20b9259abace1
mm: slub: Fix slab walking for init_on_free
by Laura Abbott 

[30 Jul 2019] b36a1552d7319bbfd5cf7f08726c23c5c66d4f73
Bluetooth: hci_uart: check for missing tty operations
by Vladis Dronov 

[5 Aug 2019] bfd77145f35c3deafe57e9eb67fff4ccffdaef6e
Makefile: Convert -Wimplicit-fallthrough=3 to just -Wimplicit-fallthrough for clang
by Joe Perches 

-----
Total patches found: 14
Total patches manipulated: 7
```

As you can see, with a not so complicated _awk_ program I can somewhat easily
filter the input and generate my desirable output. And because of that, I
insist, don't hesitate in consider using _awk_ for your daily basis tasks
around text processing.

## Final considerations

I know there are other text processing tools, for instance, _sed_ or even
_Perl_, which has a really powerful text processing engine, even faster than
_awk_, but sometimes the task to be performed doesn't require a complete
language to be used, or even you need to gather an specific column off that
specific line, why wouldn't I just call _awk_ on its simplest form? 

That's where I really like the idea of having more than one way to solve a
problem, or in this case, have more than a single tool.

Feel free to use whatever you want, but don't hesitate in consider other tools
you have in hand, tools that are very likely to be available by default in your
system, like _awk_.
