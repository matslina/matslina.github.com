---
layout: post
title: qsort() and its killer adversaries
location: New York
draft: true
---

Algorithmic complexity attacks are denial of service attacks that
operate by triggering algorithmic worst case behaviour. A piece of
code that performs well in the average case can perform very poorly on
certain inputs. The canonical example would be the widely published
[attacks against hash table
implementations](http://www.cs.rice.edu/~scrosby/hash/CrosbyWallach_UsenixSec2003/index.html),
where carefully crafted inputs made snappy \\(O(1)\\) operations
deteriorate into \\(O(n)\\) time sinks. Several major programming
language implementations and web frameworks were vulnerable.

Another algorithm commonly mentioned in this context is quicksort,
with it's average \\(O(n\log n)\\) and worst case \\(O(n^2)\\). When
we [previously looked](/2013/05/31/qsort-shootout.html) at libc
qsort() implementations, it became clear that while many different
sorting algorithms are in use, quicksort is by far the most common.
This is so for good reasons. In addition to the theoretical average
case, quicksort performs very well in implementation. It is cache
friendly and optimizes well. Even so, the quadratic worst case does
look a bit scary.

In this post we'll examine how these qsort() worst case inputs can be
created, what they look like and how they affect the performance of a
couple of real world libc qsort() implementations.

Breaking BSD
============

Inspecting a
[diff](http://svnweb.freebsd.org/base/stable/9/lib/libc/stdlib/qsort.c?view=diff&r1=225736&r2=1573&diff_format=h)
between the qsort() of 4.4BSD Lite and that of FreeBSD 9.1 reveals
that very little has changed since 1994. Some K&R syntax has been
removed, some macros have been introduced, support for the reentrant
qsort_r() has been added and a couple of variables have been
renamed. Other than that, the code is pretty much the same.

![Number of comparisons per qsort() implementation when sorting 2^22
 increasing elements.](/img/max_inc.png)

This isn't a cause for concern in and of itself. As we [saw in the
previous post](/2013/05/31/qsort-shootout.html#results), the
implementation performs very well and appears to have stood the test
of time. It is particularly good on partially sorted inputs, which are
commonly encountered in practice. 

In the chart above, the FreeBSD qsort() outperforms several other
major C libraries. We'll soon discuss why, but first a little
background on quicksort and how BSD has implemented it.

### Quicksort 101

The basic flow of quicksort is as follows:

1. Select a pivot element
2. Partition the data around the pivot; smaller elements to the left, larger to the right
3. Recursively quicksort each partition

As long as the partitions end up being of roughly the same size, we
can expect the algorithm to run in \\(O(n\log n)\\). An ideal pivot
selection would always pick the median element, since that produces
perfectly balanced partitions. The worst possible pivot selection is
that which results in highly skewed partitions; the goal of each round
is not to shave off a few elements, the goal is to split the problem
in half.

Selecting the true median in each round of partitioning is
unfortunately prohibitively expensive. It [can be done in linear
time](http://en.wikipedia.org/wiki/Median_of_medians) but the constant
factors are just too high for this to make any sense in practice. BSD
qsort() approximates the true median by sampling up to 9 elements,
like so:

    pivot = median(median(v[0],         v[n/8],       v[n/4]),
                   median(v[n/2 - n/8], v[n/2],       v[n/2 + n/8]),
                   median(v[n-1 - n/4], v[n-1 - n/8], v[n-1]))

Like most of the BSD quicksort, this pivot selection is based on on
Bentley and McIlroy's [Engineering a Sort
Function](http://www.cs.fit.edu/~pkc/classes/writing/samples/bentley93engineering.pdf). This
paper, which is a bit of a classic, covers many of the less obvious
aspects of how to implement quicksort. Well worth a read if you're
into the whole sorting thing.

When a partition or an original input is sufficiently small, it can
pay off to switch to a low overhead algorithm. In the BSD case, this
algorithm is [insertion
sort](http://en.wikipedia.org/wiki/Insertion_sort) and it is chosen
whenever \\(n\lt 7\\). While the time complexity of insertion sort is
quadratic, its low constant factors makes it really shine on such
small inputs. Now this is all well and good, but BSD goes one step
further.

### The BSD deviation

As mentioned before, the BSD qsort() outperforms its competition on
sorted and partially sorted inputs. This is due to a pretty simple
heuristic: whenever a partitioning round finishes without rearranging
any elements, we switch to insertion sort! In other words, whenever an
input is perfectly partitioned around the approximated median, BSD
qsort() assumes that the input is nearly sorted and pulls the
insertion sort trigger. Insertion sort in turn, while quadratic in the
worst case, is in fact linear on nearly sorted inputs, so excellent
performance can be expected.

As it turns out, this heuristic also opens up for a very nasty worst
case behaviour. Consider an input created by the following snippet of
code:

      for (i = 0;       i < n/2; i++) v[i] = n/2 - i;
      v[n/2] = n/2 + 1;
      for (i = n/2 + 1; i < n;   i++) v[i] = n  + n/2 + 1 - i;

The plot below visualizes this input for \\(n=64\\).

![BSD qsort() input which will trigger premature switch to insertion
 sort and significant performance
 degradation.](/img/anti_freebsd-8.1.0.png)

Feed this into the BSD pivot selection and the element at position
\\(n/2\\) will pop out. Since the data is already perfectly
partitioned around this entry, no elements will be rearranged, qsort()
will assume that it's facing a nearly sorted input and will happily
switch to insertion sort. The data is however far from sorted and the
algorithm will exhibit catastrophic quadratic behaviour.

![Number of comparisons performed by BSD qsort() on random and worst
 case inputs.](/img/anti_lines_freebsd-8.1.0.png)

Notice in the plot above how doubling the input size roughly
quadruples the number of comparisons performed. This is of course the
trademark of an \\(O(n^2)\\) algorithm.

### Only FreeBSD?

<!-- TODO check most recent openbsd and dragonfly, add version numbers
here -->

4\.4BSD Lite has many descendants. Both OpenBSD and DragonflyBSD seem
to behave exactly like FreeBSD on these inputs. Many other software
projects, both free and proprietary, have also incorporated this
implementation. But not NetBSD! A [2009
commit](http://cvsweb.netbsd.org/bsdweb.cgi/src/lib/libc/stdlib/qsort.c?rev=1.20&content-type=text/x-cvsweb-markup&only_with_tag=MAIN)
removed the switch to insertion sort, citing "*catastrophic
performance for certain inputs*". Similar modifications can be found
in
e.g. [PostgreSQL](http://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/port/qsort.c;h=2747df3c5a6507201d0ce6b899ea89eecc623e35;hb=HEAD#l4)
and
[OSX](http://www.opensource.apple.com/source/Libc/Libc-763.11/stdlib/qsort-fbsd.c).

It is important to not exaggerate the significance of this particular
behaviour. After all, it only produces quadratic complexity and that's
asymptotically no worse than the regular worst case. BSD qsort() is
still a very fine quicksort.

Breaking almost any quicksort
=============================

The same [McIlroy]((http://www.cs.dartmouth.edu/~doug/) who
co-authored *Engineering a sort function* (the article mentioned above;
the one you should read) also wrote [A Killer Adversary for
Quicksort](http://www.cs.dartmouth.edu/~doug/mdmspe.pdf). It's a very
nice read.

In a nutshell, this article describes a simple adversarial program,
*antiqsort*, that reduces almost any quicksort implementation to
quadratic performance. There's no point in rehashing the article - it
really is both well-written and accessible - so let's instead have a
look at what antiqsort can do to the NetBSD qsort().

<!-- TODO replace my antiqsort with the original and regenerate the
graphics -->

### McIlroy vs NetBSD

![Number of comparisons performed by NetBSD 6.0 qsort() when faced
 with antiqsort killer inputs.](/img/anti_lines_netbsd-6.0.png)

Blam! Quadratic complexity. The absolute numbers aren't quite as bad
as for the regular BSD qsort(), but it is quadratic and that's bad
enough.

The visualization of the BSD killer input was inspired by similar
graphics for Digital Unix in McIlroy's article. As it turns out, the
corresponding visualization for NetBSD is not even remotely as pretty.

![antiqsort killer input of size 64 for NetBSD 6.0
 qsort()](/img/anti_netbsd-6.0.png)

Good looks are however of no consequence here. McIlroy's adversary can
generate these inputs of any size for (almost) any quicksort just by
linking and doing a single round of sorting. The only caveat is that
if the implementation is randomized, then we lose the ability to
"replay" the input at a later time. Again, not a big problem in
practice as very few C libraries bother with randomization.

### McIlroy vs the world

We'll also have a look at how the McIlroy adversary fares against a
couple of other libc quicksort implementations. Bear in mind that
quicksort is not the default code path of glibc qsort(). It prefers
using mergesort and only falls back to quicksort when certain memory
limits come into play. More on that and on the other implementations
in the previous post.

![McIlroy antiqsort killer adversaries for several quicksort based
 libc qsort()](/img/anti_montage.png)

Some inputs are beautiful. Some aren't. They all trigger quadratic
complexity.

What does this mean in practice?
================================

<!-- this whole section must be transmogrified -->

Calling the BSD qsort() on a \\(2^16\\) killer input is about 1000
times slower than on a random input of the same size. Most benchmarks
are unlikely to test such edge cases, so it is not hard to imagine a
scenario where this vector is exploited in a denial of service attack.

Still, if this was such big deal then we would probably see it
happening all the time in the wild. Looking through the source code of
a handful of major free software projects suggests that we should not
lose too much sleep over this.

Most programmers would be
reluctant to call qsort() on user provided input. Especially so when
the input is not trusted.


### An example

One of the few potential issues we've found lies in how some software
implements directory listings. It is common to order directory entries
by name or timestamp and this ordering is sometimes created server
side. At first glance it might seem completely harmless. Consider for
instance this listing of \\(2^15\\) random entries on a FreeBSD 9.1
box:

    $ time ls -1 random/ > /dev/null
    
    real    0m0.065s
    user    0m0.022s
    sys     0m0.004s

Very snappy and unlikely to cause concern in a benchmark. Let's see
what happens when we feed ls a killer input? The filesystem used on
this particular (virtual) machine is UFS. One of its properties is
that readdir(3) returns directory entries in the same order as they
were created. This allows us to easily build a killer input.

    $ mkdir killer
    $ for i in $(seq -w 16384 1); do touch killer/$i; done
    $ touch killer/16385
    $ for i in $(seq -w 32768 16386); do touch killer/$i; done
    
    $ time ls -1 killer/ > /dev/null
    
    real    0m5.866s
    user    0m5.719s
    sys     0m0.006s

A pretty dramatic performance drop and almost exactly what we would
expect. If the \\(2^16\\) killer results in a factor 1000 drop, then
\\(2^15\\) should give about factor 250. We got factor 260 here, from
0.022s to 5.719 user time.

<!-- libexif qsorts on exif_data_save_data_content() -->

TL;DR
=====

Plain BSD qsort() can be tricked into running insertion sort on its
whole input with terrible performance as a consequence. There exists a
simple technique for triggering similar behaviour in almost any
quicksort implementation. This applies to a fair bit of software used
in the wild, including most major libc implementations.

Over and out.
