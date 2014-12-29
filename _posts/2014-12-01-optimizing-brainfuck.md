---
layout: post
title: brainfuck optimizations - how they work and how much they help
location: New York
draft: true
---

> In almost every computation a great variety of arrangements for the
> succession of the processes is possible, and various considerations
> must influence the selection amongst them for the purposes of a
> Calculating Engine. One essential object is to choose that
> arrangement which shall tend to reduce to a minimum the time
> necessary for completing the calculation."
>
> Ada Lovelace, 1843

In this post, we'll have a look at how to "reduce to a minimum the
time necessary" for executing brainfuck code. Brainfuck compiler
optimization strategies, in other words. The chart below illustrates
the impact of the techniques covered.

![Runtime with no vs all optimizations applied](/img/runtime2.png)

Prerequisites
=============

If you're new to the brainfuck programming language, then head on over
to [the previous post](/2014/09/30/brainfuck-java.html) where we cover
both the basics of the language and some of the issues encountered
when writing a brainfuck-to-java compiler in brainfuck. With that
said, most of this post should be grokkable even without prior
experience of the language.

### Programs worth optimizing

In order to meaningfully evaluate the impact different optimization
techniques can have, we need a set of brainfuck
programs that are sufficiently non-trivial for optimization to make
sense. Unsurprisingly, most brainfuck programs out there are pretty
darn trivial, but there are some exceptions. In this post we'll be
looking at six such programs.

The interested reader is encouraged to head over to [this
repository](https://github.com/matslina/bfoptimization) for a more
detailed description of these programs.

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

The C code depends on the memory area and pointer having been set up
properly, e.g. like this:

    char mem[65536] = {0};
    int p = 0;

At this point the IR is no more expressive than brainfuck itself but
we will extend it later on.

Making things faster
--------------------

Throughout the remainder of this post we'll look at how different
optimizations affect the execution time of our 6 sample programs. The
brainfuck code is compiled to C code which in turn is compiled with
gcc 4.8.2. We use optimization level 0 (-O0) to make sure we benefit
as little as possible from gcc's optimization engine. Run time is then
measured as average real time over 10 runs on a Lenovo x240 running
Ubuntu Trusty.

### Contraction

Brainfuck code is often riddled with long sequences of <code>+</code>,
<code>-</code>, <code>&lt;</code> and <code>&gt;</code>. In our naive
mapping, every single one of these instructions will result in a row
of C code. Consider for instance the following brainfuck snippet:

    +++++[->>>++<<<]>>>.

This program will output an ascii newline character (ascii 10,
'\n'). The first sequence of <code>+</code> stores the number 5 in the current
cell. After that we have a loop which in each iteration subtracts 1
from the current cell and adds the number 2 to another cell 3 steps to
the right. The loop will run 5 times, so the cell at offset 3 will be
set to 10 (2 times 5). Finally, this cell is printed as output.

Using our naive mapping to C produces the following code:

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

Analyzing the code of our six sample programs is encouraging. For
instance, about 75% of the instructions in factor.b, hanoi.b and
mandelbrot.b are contractable, i.e. part of a sequence of at least 2
immediately adjacent <code>&gt;</code>, <code>+</code>,
<code>&lt;</code> or <code>-</code>. Lingering around 40% we have
dbfi.b and long.b, while awib-0.4.b is at 60%. Still, we shouldn't
stare ourselves blind at these figures; it could be that these
sequences are executed rarely. Let's look at an actual measurement of
the speedup.

![Improvement with contraction](/img/contract.png)

The impact is impressive overall and especially so for mandelbrot.b
and factor.b. On the other end of the spectrum we have dbfi.b. All in
all, contraction appears to be a straightforward and effective
optimization that all brainfuck compilers should consider
implementing.


### Clear loops

A common idiom in brainfuck is the clear loop: <code>[-]</code>. This
loop subtracts 1 from the current cell and keeps iterating until the
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

Inspecting our sample programs reveals that roughly 8% of the
instructions in hanoi.b and long.b are part of a clear loop, while the
corresponding figure for the other programs is around 2%.

![Improvement with clear loop optimization](/img/clearloop.png)

As expected, the impact of the clear loop optimization is modest for
all but long.b and hanoi.b, but the speedup for these two is
impressive. Conclusion: the clear loop optimization is simple to
implement and will in some cases have significant impact. Thumbs up.

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
which in turn compiles into the following C code:

    mem[p+1] += mem[p];
    mem[p+2] += mem[p];
    mem[p] = 0;

This will execute in constant time while the naive implementation of
the loop would iterate as many times as the value held in
<code>mem[p]</code>.

![Improvement with copy loop optimization](/img/copyloop.png)

Improvement across the board, except for dbfi.b.

### Multiplication loops

Continuing in the same vein, copy loops can be generalized into
multiplication loops. A piece of brainfuck like
<code>[-&gt;+++&gt;+++++++&lt;&lt;]</code> behaves a bit like an
ordinary copy loop, but introduces a multiplicative factor to the
copies of the current cell. It could be compiled into this:

    mem[p+1] += mem[p] * 3;
    mem[p+2] += mem[p] * 7;
    mem[p] = 0

Let's replace the <code>Copy</code> operation with a more general
<code>Mul</code> operation and have a look at what it does to our
sample programs.

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

![Improvement with multiplication loop optimization](/img/multiloop.png)

While most programs benefit slightly from the shift from
<code>Copy</code> to <code>Mul</code>, long.b improves significantly
and hanoi.b not at all. The explanation is simple: long.b
contains no copy loops but one deeply nested multiplication loop;
hanoi.b has no multiplication loops but many copy loops.


### Operation offsets

Both the copy loop and multiplication loop optimizations share an
interesting trait: they perform an arithmetic operation at an offset
from the current cell. In brainfuck we often find long sequences of
non-loop operations and these sequences typically contain a fair
number of <code>&lt;</code> and <code>&gt;</code>. Why waste time
moving the pointer around? What if we precalculate offsets for the
non-loop instructions and only update the pointer at the end of these
sequences?

![Improvement with operation offsets](/img/offsetops.png)

Below we list what the IR and C output looks like for the non-loop
operations. Note that while <code>Clear</code> and <code>Mul</code>
are included in this list, the test was run with the operation offset
optimization in isolation, so these two operations were never emitted
by the compiler.

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

Here be some conclusions of the impact of this optimization. Have to
rerun it first though.

### Applying all optimizations

Finally, let's apply all optimizations.

![Runtime with no vs all optimizations applied](/img/all.png)

Some conclusion. A note about how it matters that the optimizations
are applied in the correct order.

Going further
-------------

### Smaller tweaks

So far we've covered a handful of common, potentially high-impact
techniques, but there are of course also a number of additional,
smaller optimizations that can be applied. Here are some examples:

- Generalize <code>Clear</code> into <code>Set(x)</code> that sets the
  current cell to <code>x</code>. <code>Clear</code> now becomes the
  special case <code>Set(0)</code> and sequences like <code>Set(0)
  Add(47)</code> can be contracted into <code>Set(47)</code>

- Contract sequences of cancelling operations so that
  e.g. <code>Add(4) Sub(1)</code> becomes <code>Add(3)</code>

- Eliminate obviously dead code, e.g. any loop opened immediately
  after an instruction that necessarily results in the current cell
  being 0, such as <code>Close</code> and <code>Clear</code>.

Such optimizations will typically have fairly small impact on their
own but can be worth implementing after having covered the main
techniques discussed previously.

### Pre-execution or something like that

Some brainfuck programs run without ever reading any input. A compiler
can simple execute such programs at compile time and reduce them to a
simple sequence of output operations.

Perhaps less obviously, the same applies to all prefixes of brainfuck
programs up to the first loop containing an input operation. A
compiler can execute these prefixes at compile time and replace them
with more compact and efficient code. As long as the state of the
memory area and the pointer is the same, the program will operate
identically.

Summary
-------

Brainfuck is fun. Over and out.
