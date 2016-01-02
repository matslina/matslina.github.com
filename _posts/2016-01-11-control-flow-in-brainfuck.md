---
layout: post
title: control flow in brainfuck
location: New York
draft: true
---

Imperative programming languages normally offer developers a multitude
of control flow statements. We are accustomed to using loops like
<code>for</code> and <code>while</code>; conditional branches like
<code>if</code> and <code>switch</code>; subroutine invocations and
perhaps even unconditional jumps like <code>goto</code>. Similarly,
there is usually a plethora of arithmetical operators that make
comparing things a breeze.

True to its name, brainfuck is markedly different. Its only control
flow statement also doubles as its only means of performing
arithmetical comparison. In this blog post, we'll have a look at some
common high level control flow mechanisms and how the equivalent
functionality can be implemented in brainfuck.

The reader is encouraged to read up a bit about brainfuck before
proceeding, for instance by checking out the first half or so of [this
previous post](/2014/09/30/brainfuck-java.html), but no prior
experience of the language should really be needed.

Control flow and arithmetical comparison in brainfuck
=====================================================

Brainfuck's <code>[</code> and <code>]</code> instructions are
functionally equivalent to a "while non-zero" loop, i.e. a while loop
that keeps iterating until some variable reaches 0. If the variable is
zero at the beginning of the loop, then the loop will never be
entered. A brainfuck program like this:

    []

Is roughly equivalent to

    while (x != 0) {
    }

Programming with "while non-zero" as the only conditional statement
and as the only means of arithmetical comparison can be a bit of a
challenge. As the rest of this post illustrates, it is however
sufficient.

<!--
This shouldn't come as a surprise either, since brainfuck
is turing-complete and as such can solve any solveable computational
problem.
-->

<!-- this section felt a bit thin. pointless. -->

if non-zero (destructively)
===========================

Let's say we wish to write a program like this:

    x = read()
    if (x != 0):
        write(x)

Accomplishing the same in brainfuck, using only "while non-zero", is
not too difficult. Replace the <code>if (x != 0)</code> with a
<code>while (x != 0)</code> and then make sure <code>x == 0</code>
before the loop is able to iterate a second time. Brainfuck doesn't
have an assignment instruction, so clearing <code>x</code> requires
using a nested while loop:

    x = read()
    while (x != 0):
        write(x)
        while (x != 0):
            x = x - 1

As clunky as this may seem, each line in the example corresponds to a
brainfuck instruction, and the resulting program is as elegant as it
is terse:

    ,[.[-]]

Let's walk through the code. Input is read into a memory cell using
the <code>,</code> instruction. A "while non-zero" loop is opened
using <code>[</code> and the <code>.</code> instruction writes the
memory cell as output. The inner loop <code>[-]</code> repeatedly
decrements the memory cell until it reaches zero which allows
execution to move past the final <code>]</code>.

![if non-zero, destructively, 4 as input](/img/bfflow_ifnonzero_destructive_1.gif)

The animation above shows the program being executed with the number 4
provided as input. At each step we can see the currently executing
instruction highlighted in red and how it affects the memory cell. If
the number 0 was read instead, execution would flow like this:

![if non-zero, destructively, 0 as input](/img/bfflow_ifnonzero_destructive_0.gif)

To summarize, <code>if (x != 0) { stuff }</code> can be implemented in
brainfuck like this:

    [ stuff [-]]

Albeit in a way that destroys the value held in <code>x</code>.


if non-zero (non-destructively)
===============================

So what if we need to retain the byte read as input? Well,
<code>x</code> itself definitely needs to be cleared for the "while
non-zero" loop to terminate, but there's no reason we can't make a
copy of it in the process.

    y = 0
    x = read()
    while (x != 0) {
        write(x)
        while (x != 0) {
            x = x - 1
            y = y + 1
        }
    }

We introduce a new variable <code>y</code> and initialize it to 0. The
clear loop has been modified so that each time <code>x</code> is
decremented, <code>y</code> is also incremented. Our clear loop has
been turned into a move loop. However, brainfuck doesn't really have
variables, so the pseudo code can't just be translated line by line as
we did in the destructive case.

Instead of variables, brainfuck has a single, large contiguous memory
area. All memory cells are initially set to 0. There is a pointer that
points at the leftmost cell in the memory area. Brainfuck is all about
moving that pointer around and operating on the cell it points at.

Using the <code>&lt;</code> and <code>&gt;</code> instructions, we can
move the pointer left and right respectively. This allows us to make
use of a second memory cell in place of the variable
<code>y</code>. In this case, we use the cell to the right of where we
store <code>x</code>:

    ,[.[->+<]]

Notice that the only difference from the previous (destructive) "if
non-zero" is that we've replaced <code>[-]</code> with
<code>[-&gt;+&lt;]</code>. Instead of clearing <code>x</code>, we move
it.

![if non-zero, non-destructively, 4 as input](/img/bfflow_ifnonzero_nondestructive_1.gif)

More generally, we can implement <code>if (x != 0) { stuff }</code>
non-destructively like this:

    [ stuff [->+<]]

Beware that <code>stuff</code> must be kept from tampering with both
the current cell and the one above it, since these are both accessed
in the move loop.

if zero
=======

At this point we've familiarized ourselves with two ways of checking
if something is non-zero. What about the opposite? How do we check
<code>if (x == 0)</code>? Well, here's one way:

    y = 1
    x = read()
    if (x != 0) {
        y = y - 1
    }
    if (y != 0) {
        stuff
    }

The variable <code>y</code> holds a flag that gets cleared if and only
if <code>x != 0</code>. In other words, if <code>x == 0</code> then
and only then will the flag remain set. Very simple stuff both in
pseudo code and in brainfuck. For the latter, we will of course have
to use one of our "if non-zero" constructs in place of the pseudo
code's if statements.

    >+<,        # set flag and read x
    [>-<[-]]    # if x non zero then clear flag
    >[ stuff -] # if flag still set then do stuff

The following two animations visualize the execution of such a
program, where <code>stuff</code> writes the flag as output if <code>x
== 0</code>. First with 0 as input:

![if zero, destructively, 0 as input](/img/bfflow_ifzero_destructive_0.gif)

And then with 4 as input:

![if zero, destructively, 4 as input](/img/bfflow_ifzero_destructive_1.gif)

Note that while we chose to use the destructive "if non-zero" here, we
could just as well have used the non-destructive one.


if zero (non-destructively, efficiently)
========================================

An issue with the approaches we've seen so far, destructive and
non-destructive alike, is that they can be rather slow. Due to the
clear or move loops involved, the run time of the code will be
proportional to the value we're checking. E.g., if that <code>x</code>
happens to be <code>155</code> then it will take <code>155</code>
iterations before both clear and move loops terminate.

We can do better:

    >+<,[>-]>[>]<[ stuff -]

<!-- TODO mention that pseudo code is pointless here -->

<!-- TODO this is more complex than previous examples. those loops
          with only 1 > serve to make sure pointer gets repositioned
          at same cell for all inputs. -->

Let's say the byte read is non-zero. The <code>[&gt;-]&gt;</code> will
then clear the flag and reposition the pointer to the cell above the
flag. Since this cell is 0, the following <code>[&gt;]</code> won't be
executed and the final <code>&lt;</code> repositions the pointer back
to the flag cell, which will still be 0. Consequently, if the byte
read was non-zero, then <code>stuff</code> will not be executed.

![if zero, non-destructively, efficiently, 4 as input](/img/bfflow_ifzero_nondestructive_efficient_1.gif)

Say the byte read was zero. This time <code>[&gt;-]</code> won't be
entered, which leaves the flag cell set to 1. The <code>[&gt;]</code>
will be now be entered and therefore reposition the pointer to the
empty cell above the flag. The final <code>&lt;</code> moves the
pointer back to the flag cell, which will still be 1. Consequently, if
the byte read was zero, then <code>stuff</code> will be executed.

![if zero, non-destructively, efficiently, 0 as input](/img/bfflow_ifzero_nondestructive_efficient_0.gif)

if equal
========

So why have we spent so much time devising these checks for something
being zero and non-zero? What is it all good for? Well, brainfuck
doesn't have any instructions for checking if one thing equals
another, but we can use the arithmetic instructions and "if zero" to
the same effect. If <code>x == A</code>, where <code>A</code> is some
constant, then it must also hold that <code>x - A == 0</code>.

Say we wish to write <code>if (x == 4) { stuff }</code>. Here's how:

    >+<,                # set a flag and read x
    ----                # subtract 4 from x
    [>-]>[>]<[ stuff ]  # if x became 0: do stuff
    <++++[-]            # restore and clear x

Notice how we took care to restore <code>x</code> by adding 4, even
though we then immediately clear it with a <code>[-]</code>. This is
strictly speaking not required in this tiny program, but it does
illustrate an important point. Here's an example of the program
executing with the number 2 provided as input:

![if equal 2 as input](/img/bfflow_ifequal_2.gif)

In most brainfuck environments, including the one executing in the
animation, cells are 8 bit unsigned integers. Subtracting 2 from 0
means subtracting 2 from 256, so the first cell ends up holding
254. Clearing that value with a <code>[-]</code> loop would take 254
iteration in our case, but in other brainfuck environments it may take
many, many more. For instance, if cells were 32 bits, then we'd be
looking at more than 4 billion iterations which would severely impact
the performance of the program. Portable brainfuck code recognizes
this and always makes sure to restore variables before iterating over
them.

Here's what the computation looks like when the number 4 is read and
the equality condition is met:

![if equal 4 as input](/img/bfflow_ifequal_1.gif)

switch
======

The final mechanism we'll consider is the <code>switch</code>
statement. Say we wish to write this program:

    x = read()
    switch (x) {
      case 5 {
        foo
      }
      case 2 {
        bar
      }
    }

This can of course be accomplished with a sequence of "if equal", but
there is another relatively common pattern that is worth mentioning:

    +>,--[---[<->+++++[-]]
    <[- foo ]>]
    <[- bar ]

This can of course be extended to any number of cases.


<!--
    +>,--[---[<->+++++[-]]<[- case 5 ]>]<[- case 2 ]


    +>        # the first cell is a flag initially set to 1
    ,         # second cell holds the value read
    --[       # enter outer loop iff value != 2
    ---[      # enter inner loop iff value != 5
    <->       # clear the flag in the first cell
    ++++++[-] # restore the original value and then clear the cell
    ]         # and exit the inner loop

    <[-       # if the flag is still set at this point then the
              # inner loop cannot have been entered; consequently
              # this loop is only entered if value == 5;
              # we clear the flag and whatever is required for case 5

    ]>]       # close the case 5 body and the outer loop
    <[-       # the only way the flag is still set at this point is if
              # value == 2; so if this loop is entered then we clear the
              # flag and do whatever is required for case 2
    ]         # and then we're done
-->
