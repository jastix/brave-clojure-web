---
title: Concurrency & Parallelism
link_title: Concurrency & Parallelism
kind: documentation
draft: true
---

One of Clojure's big selling points is its support for concurrent
programming. In this chapter, you'll learn the following:

* Intro to concurrency & parallelism
    * Definitions
    * Why should you care?
* Easy concurrent ops
* What are the dangers?
* The Clojure model of state
* Tool 1: Statelessness
* Tool 2: Atomic updates
* Tool 3: STM
* Tool 4: core.async

## Concurrency and Parallelism Defined

Concurrent and parallel programming involves a lot of messy details at
all level of program execution, from the hardware to the operating
system to the programming language to the code that you're typing out
with your very own fingers. In this section we'll ignore those details
and instead focus on the implementation-independent high-level
concepts.

*Concurrency* refers to *managing* more than one task at the same
time. We can illustrate concurrency with the song "Telephone" by Lady
Gaga. Gaga sings,

    I cannot text you with a drink in my hand, eh

Here, she's explaining that she can only manage one task (drinking).
She flat out rejects the suggestion that she manage more than one
task. However, if she decided to process tasks concurrently, she would
sing,

    I will put down this drink to text you,
    then put my phone away and continue drinking, eh

In this hypothetical universe, Lady Gaga is *managing* two tasks:
drinking and texting. However, she is not *executing* both tasks at
the same time. Instead, she's switching between the two. This is
called *interleaving*. Note that, when talking about interleaving, you
don't have to fully complete a task between switching; Gaga could type
one word, put her phone down, pick up her drink and have a sip, then
switch back to her phone and type another word.

*Parallelism* refers to *executing* more than one task at the same
time. If Madame Gaga were to execute her two tasks in parallel, she
would sing:

    I can text you with one hand
    while I use the other to drink, eh

Clojure has many features which allow you to easily achieve
parallelism. Whereas the Lady Gaga system achieves parallelism by
simultaneously executing tasks on multiple hands, computer systems
generally achieve parallelism by simultaneously executing tasks on
multiple cores or processors.

You can see from this definition that parallelism is a child of
concurrency: in order to execute multiple tasks simultaneously, you
first have to manage multiple tasks. Concurrency can be seen as
potential paralellism.

It's important to distinguish *parallelism* from *distribution*.
Distributed computing is a specialization of parallel computing where
the processors don't reside in the same computer. It'd be like Lady
Gaga asking Beyoncé, "Please text this guy while I drink."

I'm only going to use "parallel" to refer to cohabitating processors.
While there are Clojure libraries that aid distributed programming,
this book only covers parallel programming. That said, the concepts
you'll learn apply directly to distributed programming.

"Concurrent programming" and "parallel programming" refer to how you
decompose a task into parallelizable sub-tasks and the techniques you
use manage the kinds of risks that arise when your program executes
more than one task at the same time.

To better understand those risks and how Clojure help you avoid them,
let's examine how concurrency and parallelism are implemented in
Clojure.

## Clojure Implementation: JVM Threads

I've been using the term "task" to refer to *logical* sequences of
operations which can be performed independently of each other. Texting
can be performed independently of pouring a drink into your face, for
example. So, "task" is an abstract term which says nothing about
implementation. 

I think of a thread as an actual, physical piece of thread that's been
threaded through a sequence of instructions. In my mind, the
instructions are marshmallows, because marshmallows are delicious. The
processor executes these instructions in order. I envision this as an
alligator consuming the instructions, because alligators love
marshmallows (true fact!). So executing a program looks like a bunch
of marshmallows strung out on a line with a pacifist alligator
traveling down the line and eating them one by one.

Here's a single-core processor executing a single-threaded program:

TODO: add image here

A thread can "spawn" a new thread. In a single-processor system, the
processor switches back and forth between the threads (interleaving).
Here's where we things start to get tricky. While the processor will
execute the instructions on each thread in order, it makes no
guarantees about when it will switch back and forth between threads.

Here's an illustration of two threads, "A" and "B", along with a
timeline of how their instructions could be executed. I've lightly
shaded the instructions on thread B to help distinguish them from the
instructions on thread A:

TODO: add image here

Note that this is just a *possible* ordering of instruction execution.
The processor could also have executed the instructions in the order,
"A1, A2, A3, B1, A4, B2, B3", for example. The main idea is that you
can't know what order the instructions will actually take. This makes
the program *nondeterministic*. You can't know beforehand what the
result will be because you can't know the execution order.

To drive this point home, imagine the program in the image above
includes the following pseudo instructions:

| ID | Instruction |
|----+-------------|
| A1 | WRITE X = 0 |
| A2 | READ X      |
| A3 | PRINT X     |
| B1 | WRITE X = 5 |

If the processor follows the order "A1, A2, A3, B1" then the program
will print the number `0`. If it follows the order "A1, B1, A2, A3",
then the program will print the number `5`. Even weirder, if the order
is "A1, A2, B1, A3" then the program will print `0` even though `X`'s
value is `5`.

This little thought experiment demonstrates one of the central
challenges in concurrent programming. I'll cover it in more detail
later in this chapter, along with Clojure's facilities for working
with nondeterminism. For now, though, let's round out our discussion
of threads by showing what happens when multiple cores meet multiple
threads.

Whereas the above example showed concurrent execution on a single
processor through interleaving, a multi-core system will assign a
thread to each core. This allows the computer to execute more than one
thread simultaneously. Each core executes its thread's instructions in
order:

TODO: image

Notice that the image doesn't show the *absolute* ordering of the
instructions. As with interleaving on a single core, there are no
order guarantees so the program is nondeterministic. To make things
even more fun, your programs will typically have more threads than
cores, so each core will have to switch back and forth among multiple
threads.


### Who Cares?

Why should you bother with any of this? The reason is that parallel
programming is essential for writing performant applications on modern
hardware. In recent years CPU designers have focused more on enabling
parallel execution than on increasing clock speed. This is mostly
because increasing clock speed requires exponentially more power,
making it impractical. You can google "cpu power wall", "cpu memory
wall" and "instruction-level parallelism wall" to find out more. As
those creepy Intel bunnies continue smooshing more and more cores
together, it's up to you and me to squueze performance out of them.

Performance can be measured as both *latency*, or the total amount of
time it takes to complete a task, and *throughput*, or the number of
tasks a program completes per second. Parallelism can improve both, as
this diagram shows:

TODO: diagram serial vs parallel execution of tasks

## Easy Concurrent Ops



* Easy concurrent ops
    * control when a task runs, its effect on the current thread, and
      how to get its val
    * future
    * delay
    * promise - show example of searching multiple sources, delivering
      first result
    * combine delay/future
    * show when blocking happens
    * explain that concurrent tasks are composed of sequential ops
* What are the dangers?
    * Commonly understood as shared access to mutable state
        * reference cell
        * mutual exclusion
        * dining philosophers
    * Clojure facilities understood as tools for safety and liveness
* Shared access to mutable state
    * How other languages define state and how that causes trouble
    * We need to define state
        * chain of related values associated with an identity
        * regardless of how you think the world actually works, you
          can agree that in computers we're dealing with information,
          and information is immutable
* Tactic one: statelessness
    * pure functions are parallelizable
    * immutable objects are parallelizable
        * explain how keys and indices can be seen as names
    * pmap
    * ppmap
    * reducers
* Tactic two: atomic updates
    * using `atom`
* Tactic three: STM
* Tactic 4: core.async