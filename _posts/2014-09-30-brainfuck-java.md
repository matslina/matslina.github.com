---
layout: post
title: compiling brainfuck to java in brainfuck
location: New York
draft: false
---

When your primary goal in language design is to enable the
construction of a really tiny compiler, there’s a good chance you’ll
end up with something like brainfuck. This is a minimalistic and
slightly weird language. Most who attempt to program in it will agree
that it is aptly named. But brainfuck is also turing-complete and as
such capable of solving any and all computational problems.

Awib is a brainfuck compiler written entirely in brainfuck. It has
multiple compiler backends making it capable of compiling brainfuck to
well-performing executables (Linux IA32 ELF) as well as to several
programming languages (C, Tcl, Ruby and Go).

In this post we’ll have a look at some of the challenges involved in
adding a Java backend to awib.

## Brainwhat?

Brainfuck’s computational model is simple. There is a contiguous
memory area of *cells* and a *pointer* into said memory. Although the
exact size can differ between implementations, it is generally safe to
assume that each memory cell has 8 bits or more. All cells are
initially set to 0.

    +---+---+---+---+---+---+-----
    | 0 | 0 | 0 | 0 | 0 | 0 | ...
    +---+---+---+---+---+---+-----
      ^

There are 8 single character instructions that manipulate either the
pointer or the cell at which the pointer points. A straightforward way
of explaining the brainfuck instruction set is to translate it into a
more widely known language, such as Java. If we start off by declaring
the memory area and the pointer like this:

    int p = 0;
    byte[] mem = new byte[65536];

Then the brainfuck instruction set can be defined like this:

<table>
<tr>
 <th>Brainfuck</th>
 <th>Meaning</th>
 <th>Java</th>
</tr>
<tr>
 <td style="text-align: center;"><code>+</code></td>
 <td>Increment the cell currently pointed at</td>
 <td><code>mem[p] += 1;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>-</code></td>
 <td>Decrement the cell currently pointer at</td>
 <td><code>mem[p] -= 1;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>&gt;</code></td>
 <td>Move pointer 1 cell rightwards</td>
 <td><code>p += 1;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>&lt;</code></td>
 <td>Move pointer 1 cell leftwards</td>
 <td><code>p -= 1;</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>.</code></td>
 <td>Output the cell currently pointed at</td>
 <td><code>System.out.write(mem[p]);</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>,</code></td>
 <td>Read a byte of input into the cell currently pointed at</td>
 <td><code>System.in.read(mem, p, 1);</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>[</code></td>
 <td>If the current cell is non-zero then enter a loop</td>
 <td><code>while (mem[p]!=0) {</code></td>
</tr>
<tr>
 <td style="text-align: center;"><code>]</code></td>
 <td>If the current cell is non-zero then keep looping</td>
 <td><code>}</code></td>
</tr>
</table>

While the language itself is simple, using it to write non-trivial
programs can be hard.

## Awib is a brainfuck compiler written in brainfuck

A great way of grokking the brainfuck language is to implement it. The
process of writing a compiler or an interpreter makes both syntax and
semantics a lot clearer. When you think about it, should it not be
even more educational to write the compiler itself in brainfuck?
Perhaps not a bullet proof argument, but that's what led to the
creation of awib. It's a brainfuck compiler written in brainfuck.

Compilers are commonly composed of one or more frontends and one or
more backends. The frontends accept source code as input and produce
an intermediary representation (IR) as output. The backends in turn
accept IR as input and produce executable programs as
output. Commonly there will also be an optimization step between
frontend and backend. GCC is a great example of a compiler structured
like this; it has frontends for e.g. C, C++ and Fortran; it has
backends for e.g. arm, sparc and i386.

The awib compiler is structured in the same way. The frontend reads
brainfuck source code from stdin, translates it into IR and performs a
couple of optimizations. Control is then handed over to the
appropriate backend which translates the IR into executable binaries
or into another programming language.

              +------- awib ---------------------------------------+
              |    +----------+     +-----------+     +---------+  |
    brainfuck |    |          | IR  |           | IR  |         |  | executable
    ----------+--->| Frontend |---->| Optimizer |---->| Backend |--+------------>
              |    |          |     |           |     |         |  |
              |    +----------+     +-----------+     +---------+  |
              +----------------------------------------------------+

Awib's IR itself is like brainfuck on steroids; it operates on the
same memory model but is much more expressive. A good example is the
IR instruction ADD(). When presented with a sequence of lets say 5
adjacent "+" instructions, awib's frontend will pass on the IR
instruction ADD(5). Similar instructions of course exists for
subtraction ("-") and for moving the pointer (">" and "<").

Although compiling brainfuck to i386 Linux executables is a pretty
hairy process - and the 386_linux backend certainly is a tad complex -
many of the backends are conceptually simple. The Ruby, Go, Tcl and C
backends operate by iterating over the IR in a single pass. For each
instruction the compiler will output a single piece of code and in
some cases also the decimal representation of the instruction's
argument. E.g., the C backend translates the instruction
<code>ADD(47)</code> into <code>*p+=47;</code>.

## Naively failing

At first glance it would seem as if Java could be treated in the same
way as any other programming language. After all, we previously were
able to define brainfuck in terms of Java, so why wouldn’t it work?
This was the initial approach adopted by the awib development team
(read: me) and it certainly looked promising.

    $ ./awib < hello.b > Bf.java
    $ javac Bf.java
    $ java Bf
    Hello World!

Excellent. After implementing, polishing, and testing, it was finally
time to release. But before we do, let’s make sure that this new
version of awib can compile itself. It would be embarrassing if the
compiler wasn’t self-hosting.

    $ ./awib < awib-0.4.b > Bf.java
    $ javac Bf.java
    Bf.java:39: error: code too large
        private void _0() {
                     ^
    1 error

Facepalm...

As it turns out, Java has a strict 64k limit on the size of compiled
methods. For a sufficiently large input, the naive mapping will result
in a method much larger than that. As demonstrated by [a 1999 bug
report](https://web.archive.org/web/20110425065511/http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=4262078),
several other Java code generators have been affected by this limit
over the years and it’s unlikely to disappear anytime soon. We need a
different approach.

The path forward is clear: we must break up our program into multiple
methods and somehow have them be invoked in the correct order. We have
immediately been moved from the happy land of trivial IR to Java
mappings to someplace worse.

## Brainfuck makes life hard

The horror of brainfuck is also the beauty of brainfuck. Implementing
the simplest thing can be a pain in the neck but the end result is
often like a work of art. Here are some of the brainfuck-specific
hurdles with implementing the aforementioned approach.

### A method for splitting methods

One option for circumventing the 64k limit is to simply create a new
method for each loop body encountered. Implementing this would be a
piece of cake in most programming languages: compile non-loop
instructions as usual, recursively compile loop bodies, attach the
compiled loops to a list of methods and finally append these method
declarations at the end of the output before returning.

<!--
    function compile(input):
        methods = list()
        for i = 0; i < input.size(); i++:
            ...
            if input[i] == [
                end = find_end_of_loop(input, i)
                methods.append(compile(input[i ... end]))
-->

Alas, brainfuck does not support recursion and maintaining the method
list will require a lot of scanning back and forth over the memory
area (no random memory access in brainfuck, remember?) Turning the
recursive implementation into an iterative one would require
introducing a stack, adding further complexity.

What we want is a way of compiling the IR into Java in a single pass,
without copying code around and only using very simple data
structures. Here's one way to do it:

    count = 1
    for op in input
        ...
        if op = [
            print "    while (mem[p] != 0) {
            print "        method_" + count + "();"
            print "    }"
            print "    method_" + (count + 1) + "();"
            print "}"
            print "private void method_" + count + "() {"
            stack.push(count + 1)
            count += 2
        if op = ]
            print "}"
            print "private void method_" + stack.pop() + "() {"
        ...

This executes in a single pass over the IR and only requires us to
maintain a counter and a stack of integers (previous counter
values). Much more brainfuck-friendly.

### Representing large integers

As mentioned, cells in the memory area can't be assumed to be larger
than 8 bits, so a single cell counter can not be expected to count
beyond 255 (2<sup>8</sup>-1). The chosen algorithm will increment the
counter twice for each loop encountered so single cell counters won't
allow more than 127 loops being compiled. Not nearly enough for
anything but the smallest of programs.

Our counters must span multiple cells and it seems as if two cells,
giving us 16 bits, should do the trick. 32767 ((2<sup>16</sup>-1) / 2)
loops ought to be enough for anybody, no?

The most straightforward way to implement this is to start both cells
at 0 and on each increment check if the least significant cell holds
255. If it does then we reset it to 0 and increment the most
significant cell. If it does not then incrementing the least
significant cell is enough.

    function increment(hi, lo):
        if lo < 255:
            return (hi, lo + 1)
        return (hi + 1, 0)

In practice, always having to do a comparison against 255 becomes
annoyingly expensive. Brainfuck only allows us to check if cells are
zero or non-zero, so checking for an non-zero value X requires first
subtracting X from the cell, then checking for zero, then restoring
the cell's original value. An easier approach is to initialize the
counters to 255 and decrement instead of incrementing:

    function decrement(hi, lo):
        if lo != 0:
            return (hi, lo - 1)
        return (hi - 1, 255)

The difference may appear subtle in our high-level pseudo code, but it
makes a real difference when implementing in brainfuck:

    +<-[>-]>[>]<[+++++++++++++++[-<++++++++++++++++>]<-<->>]

Beautiful. This is actually a little bit different from the pseudo
code decrement. Deciphering precisely how is left as an exercise to
the reader.

### Storing the stack

Implementing a stack in a random access memory model is
straightforward: maintain an array of stack frames and use an integer
variable to keep track of where the top frame resides. A stack push
means incrementing the integer and writing a frame; a stack pop means
reading a frame and decrementing the integer.

Brainfuck's memory model changes some things. The lack of random
access makes the stack top variable redundant and the lack of dynamic
memory allocation requires us to dedicate a stack segment of the
memory area ourselves. We also need to make sure that the stack can
grow a fair bit to allow for the compilation of programs with many
levels of nested loops.

    +---------------+---+---+---+-   -+---+---+------------+
    | stack segment | 0 | 1 | 1 | ... | 1 | 0 | IR segment |
    +---------------+---+---+---+-   -+---+---+------------+

The diagram above illustrates awib's memory layout in this phase of
the compilation. Each IR instruction is two cells or larger so
processing an instruction effectively frees up two cells for the stack
segment. Since each stack frame spans two cells, this guarantees that
the stack won't expand into and ruin the IR segment.

Most instructions processed will not be loops, so we can expect the
area between the IR and the stack to grow larger and larger. We won't
be able to move the pointer to the stack using a constant number of
'<' instructions. One option would be to maintain a counter of the
number of cells between the two segment and use that to maneuver
between the segments. An easier solution is to fill the gap with
non-zero cells, in our case 1's, so that we can traverse the gap using
a simple brainfuck idiom:

    [<]

This snippet opens a loop, moves the pointer a single step left and
then closes the loop. Since brainfuck loops only terminate when the
cell currently pointed at is zero, the loop will move the pointer
leftwards over the gap until the cell holding 0 is reached. Replacing
the '<' with a '>' can obviously be used to move back to the IR
segment.

## Where did this get us?

Putting all this together resulted in an awib backend that appears to
function as it should. It passes all of awib's unit and system tests
and successfully compiles awib itself. The commented source code of
the backend is available
[here](https://code.google.com/p/awib/source/browse/trunk/lang_java/backend.b?spec=svn117&r=116).

To try it out, just download
[awib-0.4.b](http://awib.googlecode.com/svn/builds/awib-0.4.b) and
follow the instructions in the file. Awib is, as of version 0.2,
polyglot in C, Tcl and bash so you can easily bootstrap it locally
without installing any other brainfuck interpreter or
compiler. Remember to prepend the string "@lang_java" to your
brainfuck code to instruct awib to use the Java backend.

That was all. Thanks for reading.

-

*This article was originally published at [Hakka
 Labs](http://www.hakkalabs.co/articles/compiling-brainfuck-to-java-in-brainfuck).*
