---
layout: post
title: qsort() killer adversaries
location: New York
draft: true
---

We [previously looked](/2013/05/31/qsort-shootout.html) at a couple of
libc qsort() implementations and tried to contrast their
performance. While several different algorithms were discussed, the
most common was clearly quicksort. This is so for good reasons;
quicksort is an excellent algorithm with very good performance both in
theory and in implementation. \\(O(n\log n)\\) comparisons on average,
cache friendly, etc.

Unfortunately, quicksort also comes with poor worst case
behaviour. \\(O(n^2)\\) to be specific. In this post we'll examine
what these worst case inputs can look like and how we can create
them. For fun and profit.

Breaking BSD
============

Inspecting a [diff between the qsort() of 4.4BSD Lite and that of
FreeBSD
9.1](http://svnweb.freebsd.org/base/stable/9/lib/libc/stdlib/qsort.c?view=diff&r1=225736&r2=1573&diff_format=h)
reveals that very few things have changed since 1994. Some K&R syntax
has been removed, some macros have been introduced, support for the
reentrant qsort_r() has been added and a couple of variables have been
renamed. Other than that, the code is pretty much the same.

This isn't a cause for concern in and of itself. As we saw in the
[previous post](/2013/05/31/qsort-shootout.html#results), the
implementation performs very well and it has definitely stood the test
of time. It is particularly good on partially sorted inputs, which are
commonly encountered in practice.

![Number of comparisons per qsort() implementation when sorting 2^22
 increasing elements.](/img/max_inc.png)

In the plot above, the FreeBSD qsort() outperforms several other major
C libraries. We'll soon discuss why, but first a little background on
quicksort and how BSD has implemented it.

### BSD Quicksort 101

The BSD qsort() is a quicksort based on Bentley and McIlroy's
[Engineering a Sort
Function](http://www.cs.fit.edu/~pkc/classes/writing/samples/bentley93engineering.pdf). This
paper is a bit of a classic and covers many less than obvious aspects
of how to implement quicksort. Very much worth reading, if you're into
the whole sorting thing.

The basic flow of quicksort is as follows:

1. Select a pivot element
2. Partition the data around the pivot; smaller elements to the left, larger to the right
3. Recursively quicksort the partitions

As long as the partitions end up being of "roughly" the same size, we
can expect the algorithm to run in \\(O(n\log n)\\). An ideal pivot
selection would always pick the median, but that's prohibitively
expensive in practice. BSD qsort() selects its pivot by sampling up to
9 input elements, like so:

    pivot = median(median(v[0],         v[n/8],       v[n/4]),
                   median(v[n/2 - n/8], v[n/2],       v[n/2 + n/8]),
                   median(v[n-1 - n/4], v[n-1 - n/8], v[n-1]))

When partitions (or original inputs) are sufficiently small, it pays
off to switch to a low overhead algorithm. In the BSD case, this means
a switch to [insertion
sort](http://en.wikipedia.org/wiki/Insertion_sort) when \\(n\lt
7\\). Insertion sort has quadratic complexity for general inputs, so
it'd be a bad idea to use it for most larger inputs, but its low
overhead makes it ideal for small numbers of elements.

As mentioned, the BSD qsort() really shines on sorted or partially
sorted inputs. This is due to a simple heuristic: whenever a
partitioning round finishes without rearranging any elements, we
switch to insertion sort. Whenever qsort() encounters partially sorted
data, there's a very good chance of this heuristic triggering a linear
insertion sort on the sorted segments.

### Attacking the heuristic

As it turns out, the heuristic that is intended to catch nearly sorted
input also opens up for a very nasty worst case behaviour. Consider an
input created by the following snippet of code:

      for (i = 0;       i < n/2; i++) v[i] = n/2 - i;
      v[n/2] = n/2 + 1;
      for (i = n/2 + 1; i < n;   i++) v[i] = n  + n/2 + 1 - i;

The plot below visualizes this input for \\(n=64\\).

![BSD qsort() input which will trigger premature switch to insertion
 sort and significant performance
 degradation.](/img/anti_freebsd-8.1.0.png)

If we feed this into the previously discussed pivot selection, we'll
get back the element at position \\(n/2\\), i.e. \\(n/2+1\\). The data
is already perfectly partitioned around this element so no elements
will be rearranged. Therefore qsort() will assume that it's facing a
nearly sorted input and happily switch to insertion
sort. Unfortunately, the input is far from sorted and insertion sort
will exhibit catastrophic quadratic behaviour.

![BSD qsort() input which will trigger premature switch to insertion
 sort and significant performance
 degradation.](/img/anti_lines_freebsd-8.1.0.png)

Notice in the plot above how doubling the input size roughly
quadruples the number of comparisons performed. This is the trademark
of an \\(O(n^2)\\) algorithm.

### Only FreeBSD?

As mentioned, the FreeBSD implementation is essentially the same as
that of 4.4BSD Lite, but there are many other descendants of this
BSD. OpenBSD, Dragonfly BSD and OSX all share this qsort().

<!-- find someone with an OSX machine and verify this -->

But not NetBSD. A [2009
commit](http://cvsweb.netbsd.org/bsdweb.cgi/src/lib/libc/stdlib/qsort.c?rev=1.20&content-type=text/x-cvsweb-markup&only_with_tag=MAIN)
removed the switch to insertion sort, citing "*catastrophic performance
for certain inputs*". Other projects borrowing the BSD qsort() have
made similar modifications,
e.g. [PostgreSQL](http://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/port/qsort.c;h=2747df3c5a6507201d0ce6b899ea89eecc623e35;hb=HEAD#l4).

This obviously raises the question: can we trigger quadratic behaviour
in the NetBSD qsort()?

Breaking almost any quicksort
=============================

The same [McIlroy]((http://www.cs.dartmouth.edu/~doug/) who
co-authored *Enginering a sort function* (the article mentioned above;
the one you should read) wrote another neat article called [A Killer
Adversary for
Quicksort](http://www.cs.dartmouth.edu/~doug/mdmspe.pdf). It's a very
nice read.

In a nutshell, this article describes a simple adversarial program
which can reduce almost any quicksort implementation to quadratic
performance. There's no point in rehashing the article - it really is
both well-written and accessible - but let's have a look at what it
can do to NetBSD qsort().

### McIlroy vs NetBSD

![antiqsort killer adversary for NetBSD 6.0
 qsort()](/img/anti_lines_netbsd-6.0.png)

Blam! Quadratic complexity. The absolute numbers aren't quite as bad
as for the regular BSD case, but it is quadratic and that's bad
enough.

The visualization of the BSD killer input was inspired by similar
graphics for Digital Unix in McIlroy's article. As it turns out, the
corresponding visualization for NetBSD is not even remotely as pretty.

![antiqsort killer adversary for NetBSD 6.0
 qsort()](/img/anti_netbsd-6.0.png)

Fortunately, good looks don't matter here. There's no need to reverse
engineer the code that constructs the input. McIlroy's adversary can
generate it for any input size and (almost) any quicksort just by
linking and doing a single round of sorting. The only caveat is that
if the implementation is randomized, then we lose the ability to
"replay" the input at a later time. Again, not a big problem in
practice as very few major C libraries bother with randomization.

### McIlroy vs the world

For fun, we'll also have a look at how the McIlroy adversary fares
against a couple of other libc quicksort implementations. Bear in mind
that quicksort is not the default code path of glibc qsort(). It
prefers using mergesort and only falls back when certain memory limits
come into play. More on that in [the previous
post](/2013/05/31/qsort-shootout.html#glibc).

![McIlroy antiqsort killer adversaries for several quicksort based
 libc qsort()](/img/anti_many_quick.png)

<!-- line plot with all num cmps for all impls against antiqsort -->

### a look at some non-libc sort implementations?

TL;DR
=====

Plain BSD qsort() can be tricked into running insertion sort on its
whole input with terrible performance as a consequence. There exists a
simple technique for triggering similar behaviour in almost any
quicksort implementation. This applies to a fair bit of software used
in the wild, including most major libc implementations.

Over and out.

![antiqsort killer adversary for the quicksort component of glibc 2.17
 qsort()](/img/anti_glibc-2.17_quick.png)

![antiqsort killer adversary for illumos
 qsort()](/img/anti_illumos.png)

![antiqsort killer adversary for illumos
 qsort()](/img/anti_lines_illumos.png)

![antiqsort killer adversary for plan9
 qsort()](/img/anti_plan9.png)

![antiqsort killer adversary for plan9
 qsort()](/img/anti_lines_plan9.png)
