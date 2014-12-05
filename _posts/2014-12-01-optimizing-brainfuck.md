---
layout: post
title: brainfuck optimizations - how they work and how much they help
location: New York
draft: true
---

<br/>
<br/>

> In almost every computation a great variety of arrangements for the
> succession of the processes is possible, and various considerations
> must influence the selection amongst them for the purposes of a
> Calculating Engine. One essential object is to choose that
> arrangement which shall tend to reduce to a minimum the time
> necessary for completing the calculation."
>
> Ada Lovelace, 1843

In this post we'll have a look at a couple of strategies that "reduce
to a minimum the time necessary" for executing brainfuck
code. Brainfuck compiler optimization strategies, in other words.

The chart below illustrates the speedup achieved by the optimization
techniques covered in this post.

![Runtime with no vs all optimizations applied](/img/runtime2.png)

Prerequisites
=============

If you're new to the brainfuck programming language, then head on over
to [the previous post](/2014/09/30/brainfuck-java.html) where we cover
both the basics of the language and some of the issues encountered
when writing a brainfuck-to-java compiler in brainfuck. With that
said, most of this post should be grokkable even without prior
experience of brainfuck.

### An intermediary representation

Our first step is to define an intermediary representation (IR) on
which an optimizer can operate. Working with an IR enables
optimizations that are platform independent and brings more
expressiveness than brainfuck itself. To further illustrate how the IR
functions, we'll also give C implementations for each operation.

In the table below we find all 8 brainfuck instructions, their IR
counterparts and an implementation of the IR in C. This basic mapping
provides a very simple and somewhat naive way of compiling brainfuck.

<table>
<tr>
 <th>BF</th>
 <th>IR</th>
 <th>C</th>
</tr>
<tr>
 <td style="text-align: center;"><code>+</code></td>
 <td><code>Add</code></td>
 <td><code>mem[p]++;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>-</code></td>
 <td><code>Sub</code></td>
 <td><code>mem[p]--;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>&gt;</code></td>
 <td><code>Right</code></td>
 <td><code>p++;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>&lt;</code></td>
 <td><code>Left</code></td>
 <td><code>p--;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>.</code></td>
 <td><code>Out</code></td>
 <td><code>putchar(mem[p]);</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>,</code></td>
 <td><code>In</code></td>
 <td><code>mem[p] = getchar();</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>[</code></td>
 <td><code>Open</code></td>
 <td><code>while(mem[p]) {</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>]</code></td>
 <td><code>Add</code></td>
 <td><code>}</code></td>
</tr>
</table>

At this point the IR is no more expressive than brainfuck itself but
we will extend it later on. The C code depends on the memory area and
pointer having been set up properly, e.g. like this:

    char mem[65536] = {0};
    int p = 0;


### Brainfuck programs worth optimizing

In order to meaningfully evaluate the impact different optimization
techniques can have on performance, we need a set of brainfuck
programs that are sufficiently non-trivial for optimization to make
sense.

[factor.b](http://www.muppetlabs.com/~breadbox/bf/factor.b.txt)
: Brian Raiter's excellent factoring program breaks arbitrarily large integers into their prime factors. In our benchmarks we run it with the number 133333333333337 as input, which factors into 397, 1279 and 262589699.

[awib-0.4.b](http://brokenlink/)
: [Awib](http://code.google.com/p/awib/), by yours truly, is a brainfuck compiler written in brainfuck. In our benchmarks we run awib-0.4 with itself as input using the lang_java backend. In other words: it compiles itself from brainfuck into the Java programming language.

[mandelbrot.b](http://esoteric.sange.fi/brainfuck/utils/mandelbrot/mandelbrot.b)
: Erik Bosman's mandelbrot implementation generates a 128x48 ascii graphics mandelbrot fractal. No input required here.

[hanoi.b](http://www.clifford.at/bfcpu/hanoi.bf)
: This [towers of hanoi solver](http://www.clifford.at/bfcpu/hanoi.html) was created by Clifford Wolf. He used a higher-level language which was then compiled into brainfuck code. No input here either.

[dbfi.b](http://www.hevanet.com/cristofd/bf/dbfi.b)
: Daniel Cristofani's dbfi is a brainfuck interpreter written in brainfuck. In our benchmark we run it with a very special input: we let it interpret a copy of itself, which in turn interprets a dummy program called [hi123](http://mazonka.com/brainf/hi123). This somewhat confusing setup is sometimes referred to as sisihi123 and was used e.g. by Oleg Mazonka when [benchmarking his interpreter, bff4](http://mazonka.com/brainf/).

[long.b](http://mazonka.com/brainf/long.b)
: Our sixth program is a dummy program that does nothing useful but takes a while to run. It also appears to have been created by Mazonka for benchmarking purposes.

Making things faster
====================

So, how can we "reduce to a minimum the time necessary for completing
the calculation"?

Throughout this post we'll look at how different optimizations affect
the execution time of our 6 sample programs. The brainfuck code is
compiled to C code which in turn is compiled with gcc 4.8.2. We use
optimization level 0 (-O0) to make sure we benefit as little as
possible from gcc's optimization engine. Run time is then measured as
average real time over 10 runs on a Lenovo x240 under Ubuntu Trusty.

### Clear loops

<!-- get some stats on how common it is in the sample programs -->

A common idiom in brainfuck is the clear loop: <code>[-]</code>. This
loop subtracts 1 from the current cell an keeps iterating until the
cell reaches zero. Executed naively, the clear loop's runtime is
potentially proportional to the maximum value that a cell can hold
(commonly 255).

We can do better. Let's introduce a <code>Clear</code> operation to
the IR.

<table>
<tr>
 <th>IR</th>
 <th>C</th>
</tr>
<tr>
 <td><code>Add</code></td>
 <td><code>mem[p]++;</code></td>
</tr>
<tr>
 <td><code>Sub</code></td>
 <td><code>mem[p]--;</code></td>
</tr>
<tr>
 <td><code>Right</code></td>
 <td><code>p++;</code></td>
</tr>
<tr>
 <td><code>Left</code></td>
 <td><code>p--;</code></td>
</tr>
<tr>
 <td><code>Out</code></td>
 <td><code>putchar(mem[p]);</code></td>
</tr>
<tr>
 <td><code>In</code></td>
 <td><code>mem[p] = getchar();</code></td>
</tr>
<tr>
 <td><code>Open</code></td>
 <td><code>while(mem[p]) {</code></td>
</tr>
<tr>
 <td><code>Add</code></td>
 <td><code>}</code></td>
</tr>
<tr style="background-color: #aaeeaa;">
 <td><code>Clear</code></td>
 <td><code>mem[p] = 0;</code></td>
</tr>

</table>

In addition to compiling all occurrences of <code>[-]</code> to
<code>Clear</code>, we can also do the same for <code>[+]</code>. This
works since (all sane implemenations of) brainfuck's memory model
provides cells that wrap around to 0 when the max value is exceeded.

![Improvement with clear loop optimization](/img/clearloop.png)

While the clear loop optimization does little to nothing for three of
the sample programs, the impact on long.b and hanoi.b is very
significant. This is obviously as a consequence of these programs
using a fair number of clear loops, but also of them perhaps doing so
a little bit too liberally.

The clear loop optimization is simple enough to implement and can have
significant impact. Thumbs up.

### Contraction

Brainfuck code is often riddled with long sequences of '+', '-', '<'
and '>'. In our naive mapping, every single one of these instructions
will result in a row of C code. Consider for instance the following
brainfuck snippet:

    +++++[->>>++<<<]>>>.

This program will output an ascii newline character (ascii 10,
'\n'). The first sequence of '+' stores the number 5 in the current
cell. After that we have a loop which in each iteration subtracts 1
from the current cell and adds the number 2 to another cell 3 steps to
the right. The loop will run 5 times, so the cell at offset 3 will be
set to 10 (2 times 5). Finally, this cell is printed as output. Using
our naive mapping to C produces the following code:

    mem[p]++;
    mem[p]++;
    mem[p]++;
    mem[p]++;
    mem[p]++;
    while (mem[p]) {
        mem[p]--;
        p++;
        p++;
        p++;
        mem[p]++;
        mem[p]++;
        p--;
        p--;
        p--;
    }
    p++;
    p++;
    p++;
    putchar(mem[p]);

We can clearly do better. Let's extend these four IR operations so
that they accept a single argument <code>x</code> indicating that the
operation should be applied <code>x</code> times:

<table>
<tr>
 <th>IR</th>
 <th>C</th>
</tr>

<tr style="background-color:  #aaeeaa;">
 <td><code>Add(x)</code></td>
 <td><code>mem[p] += x;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Sub(x)</code></td>
 <td><code>mem[p] -= x;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Right(x)</code></td>
 <td><code>p += x;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Left(x)</code></td>
 <td><code>p -= x;</code></td>
</tr>
<tr>
 <td><code>Out</code></td>
 <td><code>putchar(mem[p]);</code></td>
</tr>
<tr>
 <td><code>In</code></td>
 <td><code>mem[p] = getchar();</code></td>
</tr>
<tr>
 <td><code>Open</code></td>
 <td><code>while(mem[p]) {</code></td>
</tr>
<tr>
 <td><code>Add</code></td>
 <td><code>}</code></td>
</tr>
<tr>
 <td><code>Clear</code></td>
 <td><code>mem[p] = 0;</code></td>
</tr>

</table>

Applying this to our snippet produces a much more compact piece of C:

    mem[p] += 5;
    while (mem[p]) {
        mem[p] -= 1;
        p += 3;
        mem[p] += 2;
        p -= 3;
    }
    p += 3;
    putchar(mem[p]);

Compactness doesn't necessarily imply a speedup, so we shouldn't foo bar fixme let's examine what
this technique does to the run time of our sample programs.

![Improvement with contraction](/img/contract.png)

We see various degress of speedup with a median around 50%.

More text.

### Copy loops

Another common construct is the copy loop. Consider for instance this
little snippet: <code>[-&gt;+&gt;+&lt;&lt;]</code>. Just like the
clear loop, the body subtracts 1 from the current cell and iterates
until it reaches 0. But there's a side effect. For each iteration the
body will add 1 to the two cells above the current one, effectively
clearing the current cell while adding it's original value to the
other two cells.

Let's introduce another IR operation called <code>Copy(x)</code> that
adds a copy of the current cell to the cell at offset <code>x</code>.

<table>
<tr>
 <th>IR</th>
 <th>C</th>
</tr>

<tr>
 <td><code>Add(x)</code></td>
 <td><code>mem[p] += x;</code></td>
</tr>
<tr>
 <td><code>Sub(x)</code></td>
 <td><code>mem[p] -= x;</code></td>
</tr>
<tr>
 <td><code>Right(x)</code></td>
 <td><code>p += x;</code></td>
</tr>
<tr>
 <td><code>Left(x)</code></td>
 <td><code>p -= x;</code></td>
</tr>
<tr>
 <td><code>Out</code></td>
 <td><code>putchar(mem[p]);</code></td>
</tr>
<tr>
 <td><code>In</code></td>
 <td><code>mem[p] = getchar();</code></td>
</tr>
<tr>
 <td><code>Open</code></td>
 <td><code>while(mem[p]) {</code></td>
</tr>
<tr>
 <td><code>Add</code></td>
 <td><code>}</code></td>
</tr>
<tr>
 <td><code>Clear</code></td>
 <td><code>mem[p] = 0;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Copy(x)</code></td>
 <td><code>mem[p+x] += mem[p];</code></td>
</tr>
</table>

Applying this to our copy loop example (<code>[-&gt;+&gt;+&lt;&lt;]</code>)
results in three IR operations, <code>Copy(1), Copy(2), Clear</code>,
which in turn compile into the following C code:

    mem[p+1] += mem[p];
    mem[p+2] += mem[p];
    mem[p] = 0;

This will execute in constant time while the naive implementation of
the loop would iterate as many times as the value held in
<code>mem[p]</code>.

![Improvement with copy loop optimization](/img/copyloop.png)

Yay improvement!

### Multiplication loops

Continuing in the same vein, copy loops can be generalized into
multiplication loops. A piece of brainfuck like
<code>[-&gt;+++&gt;+++++++&lt;&lt;]</code> behaves a bit like an
ordinary copy loop, but introduces a multiplicative factor to the
copies of the current cell. It could be compiled into this:

    mem[p+1] += mem[p] * 3;
    mem[p+2] += mem[p] * 7;
    mem[p] = 0

A new IR operation is required and it could look like this:

<table>
<tr>
 <th>IR</th>
 <th>C</th>
</tr>
<tr>
 <td><code>Add(x)</code></td>
 <td><code>mem[p] += x;</code></td>
</tr>
<tr>
 <td><code>Sub(x)</code></td>
 <td><code>mem[p] -= x;</code></td>
</tr>
<tr>
 <td><code>Right(x)</code></td>
 <td><code>p += x;</code></td>
</tr>
<tr>
 <td><code>Left(x)</code></td>
 <td><code>p -= x;</code></td>
</tr>
<tr>
 <td><code>Out</code></td>
 <td><code>putchar(mem[p]);</code></td>
</tr>
<tr>
 <td><code>In</code></td>
 <td><code>mem[p] = getchar();</code></td>
</tr>
<tr>
 <td><code>Open</code></td>
 <td><code>while(mem[p]) {</code></td>
</tr>
<tr>
 <td><code>Add</code></td>
 <td><code>}</code></td>
</tr>
<tr>
 <td><code>Clear</code></td>
 <td><code>mem[p] = 0;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Mul(x, y)</code></td>
 <td><code>mem[p+x] += mem[p] * y;</code></td>
</tr>
</table>

Let's see what it does to our sample programs.

![Improvement with multiplication loop optimization](/img/multiloop.png)

Conclusions.

### Operation offsets

Both the copy loop and multiplication loop optimizations share an
interesting trait: they perform an arithmetic operation at an offset
from the current cell. In brainfuck we often find long sequences of
non-loop operations. These sequences in turn typically contain a fair
number of <code>&lt;</code> and <code>&gt;</code>. Why waste time
moving the pointer around? What if we precalculate offsets for the
non-loop instructions and only update the pointer at the end of these
sequences?

![Improvement with operation offsets](/img/offsetops.png)

It is important to note that implementing operation offsets
effectively means we've also implemented contraction of sequences of
<code>&lt;</code> and <code>&gt;</code>, so some of the speedup would
be there even with the contraction optimization on its own.

Below we list what the IR and C output looks like for the non-loop
operations. Note that while <code>Clear</code>, <code>Copy</code> and
<code>Mul</code> are included in this list, their corresponding
optimizations were not used when measuring the operation offset
speedup given above.

<table>
<tr>
 <th>IR</th>
 <th>C</th>
</tr>

<tr style="background-color:  #aaeeaa;">
 <td><code>Add(x, off)</code></td>
 <td><code>mem[p+off] += x;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Sub(x, off)</code></td>
 <td><code>mem[p+off] -= x;</code></td>
</tr>
<tr>
 <td><code>Right(x)</code></td>
 <td><code>p += x;</code></td>
</tr>
<tr>
 <td><code>Left(x)</code></td>
 <td><code>p -= x;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Out(off)</code></td>
 <td><code>putchar(mem[p+off]);</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>In(off)</code></td>
 <td><code>mem[p+off] = getchar();</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Clear(off)</code></td>
 <td><code>mem[p+off] = 0;</code></td>
</tr>
<tr style="background-color:  #aaeeaa;">
 <td><code>Mul(x, y, off)</code></td>
 <td><code>mem[p+x+off] += mem[p+off] * y;</code></td>
</tr>
</table>

### Applying all optimizations

Finally, let's apply all optimizations.

![Runtime with no vs all optimizations applied](/img/all.png)

Over and out.
