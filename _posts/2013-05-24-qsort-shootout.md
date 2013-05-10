---
layout: post
title: libc qsort() shootout
location: New York
draft: true
---

The C Standard Library (libc) provides a generic sort function called
[qsort()](http://linux.die.net/man/3/qsort). While the *q* prefix
suggests that [quicksort](http://en.wikipedia.org/wiki/Quicksort)
might be a good choice of algorithm, the standard does not dictate
what algorithm an implementation should use.

In this post we'll take a look at a few libc implementations and try
to contrast their qsort() performance.

What to measure?
----------------

Evaluating an implementations performance by measuring runtime won't
work. Some of the implementations we're interested are only
distributed for certain types of hardware. Other target specific types
of hardware, e.g. embedded devices.

A nice property of qsort() is that it is a generic sort function,
meaning it can sort any type of data. The order and how to compare
elements is defined by a comparison function which the caller must
pass to qsort(). Since we control the comparison function, we can also
count how many comparisons an implementation performs.

We claim that this is a pretty good metric. Invoking the comparison
function is a fairly expensive operation, so the less it is invoked
the better. This is especially true when comparisons are expensive,
e.g. when sorting long strings or complex data structures.

It could also be that this is a stupid metric, but whatever.

The combatants
--------------

We've a total of 14 different sort functions to go through. The
not-very-interested reader can [skip ahead](#results) to the results
and fancy graphs below.

### GLIBC

The [GNU project](http://www.gnu.org/)'s libc implementation,
[GLIBC](http://www.gnu.org/software/libc/), powers most GNU/Linux
distributions (sometimes in the shape of
[EGLIBC](http://www.eglibc.org/home)) and several other operating
systems. This qsort() is interesting in that it shuns quicksort in
favour of [merge
sort](http://en.wikipedia.org/wiki/Merge_sort). Unlike quicksort,
merge sort is not in-place and requires at least \\(O(n)\\) additional
memory. If that memory is not available, or if it exceeds a fourth of
the machine's physical RAM, then GLIBC will fall back to an in-place
quicksort. The quicksort in turn resorts to [insertion
sort](http://en.wikipedia.org/wiki/Insertion_sort) for arrays of 4 or
fewer elements.

In the graphs below, the merge sort and quicksort implementations have
been separated; *glibc-2.17_merge* and *glibc-2.7_quick*
respectively. Our humble gut feeling is that the merge sort is what
most commonly runs in "production" and that it is the most
representative of the two for GLIBC. YMMV.

### BSD

[FreeBSD](http://www.freebsd.org/) qsort() is a quicksort based on
[Engineering a Sort
Function](http://www.cs.fit.edu/~pkc/classes/writing/samples/bentley93engineering.pdf)
by Bentley and McIlroy. A significant deviation from the paper is that
the BSD qsort() doesn't only resort to insertion sort for small
arrays, but also does so whenever a partitioning round is completed
without having moved any elements. The latter is intended to capture
nearly sorted inputs, since insertion sort will handle these cases in
linear time.

The implementation appears to have undergone very little change since
[the days of 4.4BSD
Lite](http://www.freebsd.org/cgi/cvsweb.cgi/src/lib/libc/stdlib/qsort.c?rev=1.1.1.1;content-type=text/x-cvsweb-markup)
and is essentially identical to that of
[OpenBSD](http://www.openbsd.org/) and
[DragonflyBSD](http://www.dragonflybsd.org/). It is also appears in
[newlib](http://sourceware.org/newlib/) and probably also in
[OSX](http://www.apple.com/macosx/).

### NetBSD

[NetBSD](http://www.netbsd.org/)'s qsort() used to be identical to
that of the other BSDs but as of version 6.0 the aforementioned
deviation from Bentley and McIlroy has been removed. In other words,
NetBSD qsort() does *not* switch to insertion sort when a partitioning
round finishes without moving elements.

The motivation for this change seems to be the possibility of
particular inputs triggering insertion sort in spite of the data being
highly unsorted. This in turn means quadratic complexity, which can be
devastating for large inputs. The issue has been
[discussed](http://www.mail-archive.com/freebsd-hackers@freebsd.org/msg155127.html)
by FreeBSD hackers earlier, but does not appear to have been fixed in
their repository.

### klibc

When a machine running Linux is booting and the kernel is about to be
brought up,
[klibc](http://git.kernel.org/?p=libs/klibc/klibc.git;a=summary) is
made available to allow certain user space programs to operate in
spite of not yet having access to a "real" C library. Klibc is
designed to be as small and as correct as possible and is sometimes
mentioned as a good fit for embedded systems. At roughly 40 lines of
C, [the
code](http://git.kernel.org/cgit/libs/klibc/klibc.git/tree/usr/klibc/qsort.c?id=HEAD)
is very compact and easy to grok.

The klibc qsort() implements the [comb
sort](http://en.wikipedia.org/wiki/Comb_sort) algorithm; a bubble sort
on steroids. While the source code claims a time complexity of
\\(O(n\log n)\\), others,
e.g. [Vitányi](http://www.cwi.nl/~paulv/papers/sorting.pdf), put the
average complexity at \\(\Omega(n^2/2^p)\\), where \\(p\\) is the
number of passes. For klibc, using a shrinking factor of 1.3, we
should be able to expect \\(p\ge\lfloor\log_{1.3}(n)\rfloor\\). Since
\\(n^2/2^{\log_{1.3}(n)}\\) is ridiculously small for larger values of
\\(n\\), it probably makes sense to ignore the quadratic term for most
practical applications.

### uClibc

Originally created to support [uClinux](http://www.uclinux.org), a
linux distribution targeting embedded devices and microcontrollers,
the [uClibc](http://www.uclibc.org/) is designed to "provide as much
functionality as possible in a small amount of space".

This qsort() implements
[shellsort](http://en.wikipedia.org/wiki/Shellsort) which, similarly
to the comb sort of klibc, is an insertion sort on juice. The time
complexity depends on the "gap sequence" used, which in uClibc's case
is the \\((3^k-1)/2\\) (for \\(k=1,2,\ldots\\)) sequence suggested by
Knuth. This puts the algorithm at \\(\Theta(n^{3/2})\\). There are
other more efficient sequences out there but Knuth's sequence is both
compact and simple in implementation.

### dietlibc

The [diet libc](http://www.fefe.de/dietlibc/) is designed to be as
small as possible. Its qsort() is a compact and straightforward
quicksort without many bells and whistles. There is no switch to
e.g. insertion sort and only a single element is sampled when choosing
pivot.

It is worth mentioning that dietlibc 0.32 used to always select the
last element as pivot. This results in qsort() using \\(O(n)\\) stack
frames for sorted inputs, effectively crashing due to stack overflow
on anything but tiny numbers of elements. This has been fixed in
version 0.33 by choosing random pivots, and this is also the version
we consider in this article.

### musl

smoothsoort. have to figure out how it works./

### illumos

OpenSolaris ceased to exist in 2010 after Oracle bought Sun
Microsystems. A fork of the project lives on in the shape of
[illumos](http://www.illumos.org) and
[OpenIndiana](http://www.openindiana.org/). The distinction between
the two is a tad fuzzy but the C library appears to be part of
illumos.

This qsort() is very well documented and should serve nicely as a
starting point for anyone interested in studying the quicksort
algorithm. It samples multiple elementes for pivot and switches to
insertion sort for small partitions.

### plan9

Last but not least among the free software libraries, we have [Plan 9
from Bell Labs](http://plan9.bell-labs.com/plan9/index.html). This
research OS was/is develeoped as a successor to Unix. It's worth
[reading up on](http://plan9.bell-labs.com/sys/doc/).

Plan9's qsort() is a very [clean
looking](http://plan9.bell-labs.com/sources/plan9/sys/src/libc/port/qsort.c)
quicksort, although it wouldn't have hurt to throw in a comment or two
in the code. It does not switch to a low overhead algorithm for small
inputs/partitions, but it does sample multiple elements when choosing
it's pivot.

### proprietary C libraries

We'll also have a look at the performance of a couple of proprietary
qsort() implementations. The source code for some of these is floating
around on the intrawebs, but since they're non-free we won't discuss
them in detail.

* IRIX 6.5 on an Origin 300
* OpenVMS 8.3 on a DEC Alphastation 200 4/166
* Visual Studio 2005 on x86 running Windows XP
* Solaris 10 on x86

Many thanks to [andoma](http://www.lonelycoder.com/) for providing the
VS2005 data. Many many thanks to
[njansson](http://www.csc.kth.se/~njansson/) for providing access to
his SGI machine, VAX cluster and x86 Solaris server.

Results
-------

### Random data

We'll start by looking at the number of comparisons when sorting
random data.

![Number of comparisons per qsort() implementation when sorting 2^16
 random elements.](/img/min_rand.png)

The slightly less orthodox algorithms - musl's smoothsort, klibc's
comb sort and uClibc's shellsort - can be found on the left hand
side. The same goes for dietlibc's somewhat naive quicksort
implementation.

Further right we find the C libraries of, or associated with, more or
less major operating systems. It may be pointless to try to draw any
conclusions about how these fair against eachother - they're all doing
pretty well - but we can still discern three clusters.

The most well performing one holds glibc's merge sort and the
proprietary qsort() of solaris 10. To the least well performing we
count OpenVMS, VS2005, IRIX, plan9 and glibc's quicksort. The BSD
quicksorts, both with and without the heuristic for switching to
insertion sort for nearly sorted data, and opensolaris reside in the
middle.

What happens when we bump up the input size by a factor of 64?

![Number of comparisons per qsort() implementation when sorting 2^22
 random elements.](/img/max_rand.png)

We see a fairly distinct improvement for VS2005 and especially for
Solaris 10. They are definitely doing something right, but being
non-free software, it's hard to say exactly what. uClibc, on the other
hand, doesn't appear to scale very well at all. This trend is perhaps
more clear in the next plot, which is otherwise pretty useless.

![Number of comparisons per qsort() of random data for several
 sizes](/img/count_rand.png)

### Ordered data

While random data may be the most important test case, it is also
interesting to consider nearly or completely sorted inputs. We can
e.g. expect nice behaviour from FreeBSD, with its switch to insertion
sort for nearly sorted data, but not so much from NetBSD which removes
said switch. Musl libc's smoothsort should also behave nicely for
these inputs.

![Number of comparisons per qsort() implementation when sorting 2^22
 increasing elements.](/img/max_inc.png)

In addition to confirming our suspicions, we also see that uClibc's
shellsort is doing quite well. Given the shellsort algorithm's basis
in insertion sort, which is great for sorted inputs, this is not too
surprising. The merge sort of glibc also deserves mention; it appears
to provide excellent behaviour for most inputs.

![Number of comparisons per qsort() implementation when sorting 2^22
 decreasing elements.](/img/max_dec.png "max dec")

In this final diagram we consider strictly decreasing input, i.e. the
reverse of a sorted input. Here we see a drop in performance for musl,
uClibc and NetBSD. FreeBSD however remains the same.

As it turns out, FreeBSD's performance stems from how the quicksort
partitioning round is implemented. It mostly reverses the order of the
elements in each partition, which in turn allows the heuristic for
nearly sorted data to kick in early. Is this intentional or just a
happy accident? No idea.

Does this matter?
-----------------

Previously we argued that the number of comparisons is a useful
metric. If nothing else, it allows us to reason about the relative
performance of proprietary implementations running on archaic/obscure
hardware (no offence to fans of SGI or VAX). But how strong is the
correlation between this metric and actual performance?

![Runtime and number of comparisons per qsort() when sorting random
 integers.](/img/runtime_rand.png)




Summary
-------

GLIBC!!!! WOOOOOOOOOOOOO!