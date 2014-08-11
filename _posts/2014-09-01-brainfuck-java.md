---
layout: post
title: compiling brainfuck to java in brainfuck
location: New York
draft: true
---

When your primary goal in language design is to enable the
construction of a really tiny compiler, then there’s a good chance
you’ll end up with something like brainfuck. This is a minimalistic
and slightly weird language. Most who attempt to program in it will
agree that it is aptly named. But brainfuck is also turing-complete
and as such capable of solving any and all computational problems.

Awib is a brainfuck compiler written entirely in brainfuck. It has
multiple compiler backends making it capable of compiling brainfuck to
well performing executables (Linux IA32 ELF) as well as to several
programming languages (C, Tcl, Ruby and Go).

In this post we’ll have a look at some of the challenges involved in
adding a Java backend to awib.

## Brainwhat?

Brainfuck’s computational model is simple. There is a contiguous
memory area of *cells* and a *pointer* into said memory. Although the
exact size can differ between implementations, it is generally safe to
assume that each memory cell has 8 bits or more. All cells are
initially set to 0. There are 8 single character instructions that
manipulate either the pointer or the cell at which the pointer points.

    +---+---+---+---+---+---+-----
    | 0 | 0 | 0 | 0 | 0 | 0 | ...
    +---+---+---+---+---+---+-----
      ^

<!-- will prettify that diagram -->

A straightforward way of explaining the brainfuck instruction set is
to translate it into a more widely known language such as Java. If we
start off by declaring the memory area and the pointer like this:

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
 <td>Move pointer 1 cell leftwards</td>
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
 <td><code>while (mem[p]) {</code></td>
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

<!-- this paragraph confuses -->
Among the (infinitely) many computational problems brainfuck can
solve, we find the problem of brainfuck compilation: given a brainfuck
program as input, produce an executable representation of the program
as output. The awib compiler does precisely that; it’s a brainfuck
compiler written in in brainfuck.

<!-- nf suggested mentioning GCC as example -->
Compilers are commonly composed of one or more frontends and one or
more backends. The frontends accept source code as input and produce
an intermediary representation (IR) as output. The backends accept IR
as input and produces executable programs as output.

The awib compiler is structured in the same way. The frontend reads
brainfuck source code from stdin, translates it into IR and performs a
couple of optimizations. Control is then handed over to the
appropriate backend which translates the IR into executable binaries
or into another programming language.

              +------- awib ---------------------------------------+
              |    +----------+     +-----------+     +---------+  |
    brainfuck |    |          | IR  |           | IR  |         |  | executable
    -----------+-->| Frontend |---->| Optimizer |---->| Backend |--+------------>
              |    |          |     |           |     |         |  |
              |    +----------+     +-----------+     +---------+  |
              +----------------------------------------------------+

<!-- will prettify that diagram -->

<!-- Restructure this paragraph. Is "optimization" the best term? -->
The IR itself is like brainfuck on steroids. Instead of the +
instruction, which increments the current cell by 1, awib’s IR
features the ADD() instruction. One of awib’s optimizations consists
of contracting multiple adjacent "+" instructions into a single
ADD(x). Similar optimizations exist for instructions like subtraction
and for moving the pointer.

<!-- Not sure the example at the end is very illuminating. -->
Although compiling brainfuck to i386 Linux executables is a pretty
hairy process which makes the 386_linux backend a tad complex, many of
the other backends are conceptually simple. The Ruby, Go, Tcl and C
backends operate by iterating over the IR. For each instruction the
compiler will output a single piece of code and in some cases also the
decimal representation of its argument. E.g., the C backend translates
the instruction <code>ADD(47)</code> into <code>*p+=47;</code>.

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

## A method for splitting methods

The path forward is clear: we must break up our program into multiple
methods and somehow have them be invoked in the correct order. We have
immediately been moved from the happy land of trivial IR to Java
mappings to someplace worse.

<!-- do a better job motivating why this approach is bad -->

Creating a separate method for all larger loop bodies is an option
that would be acceptable in most programming language. Implementing
that in brainfuck is another story. It would potentially be too memory
consuming, would require some non-trivial data structures and would
certainly require multiple passes over the IR.

Here's an alternate approach:

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

## Brainfuck makes life hard

The horror of brainfuck is also the beauty of brainfuck. Implementing
the simplest thing can be a pain in the neck but the end result is
often like a work of art. Here are some of the brainfuck-specific
hurdles with implementing the aforementioned approach.

### Representing large integers

Since we can't assume cells to be larger than 8 bits, we have to
expect individual cells to count to at most 255 (2<sup>8</sup>). Every
loop we encounter will result in 2 methods being created, so if the
counter and stack frames are single cells then we can at most compile
127 loops. No good.

Consequently, the counter and the stack frames must span multiple
ells. Using 2 cells gives us at least 16 bits, which allows for just
over 32000 loops per program. That should be enough for everyone.

THIS IS THE POINT BEYOND WHICH EVERYTHING IS A BRAIN
DUMP / RANDOM THOUGHTS / GIBBERISH.

### Printing in base 10

Expressing an integer's decimal representation in ascii is not too
difficult in brainfuck. Awib uses a simple snippet of 240 instructions
in several places but this doesn't support printing a 16-bit integer
spanning two cells. Here we chose a simple workaround by just printing
the two values separately with a delimiting underscore character.

### Storing the stack

In a random access model, implementing a stack is pretty
straightforward. More tricky in brainfuck. We ended up storing the
stack below the IR and letting it grow towards the IR. Every IR
operation processed frees up 2 cells of stack space and since we at
most push 2 cells per operation, we're good.

    +---------------+---+---+---+-   -+---+---+------------+
    | stack segment | 0 | 1 | 1 | ... | 1 | 0 | IR segment |
    +---------------+---+---+---+-   -+---+---+------------+

<!-- will prettify that diagram -->

We use a "1-sled" for the area between the stack and the remaining
code. The sled allows us to move from the code area to the stack using
a simple brainfuck idiom:

    [<]

This code opens a loop, moves the pointer a single step left and then
closes the loop. Since brainfuck loops only terminate when the cell
currently pointed at is zero, the loop will end up moving the pointer
leftwards repeatedly until the cell left of the 1-sled holding zero is
reached.

## Where did this get us?

Putting all the pieces together resulted in an awib backend that
appears to function as it should. \o/
