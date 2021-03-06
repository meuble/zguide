
<A name="toc1-4" title="ØMQ for Multithreading" />
# ØMQ for Multithreading

By Pieter Hintjens <ph@imatix.com>, published Wed 13 October, 2010, 11:46:07.

<A name="toc2-9" title="Introduction" />
## Introduction

[ØMQ][zeromq] wasn't quite sent from the Shiny Future to make our lives better, but it sometimes feels like it.  We originally planned to create a low-latency message-carrying fabric for high-performance applications.  ØMQ is indeed this.  But ØMQ is also probably the best way to build fast, stable multithreaded applications.  I'll explain how ØMQ does this and what it means for you as developer of large, concurrent applications that use as many cores as you can throw at them, with perfect scaling.

This is an introduction, and you don't need any prior experience with ØMQ to read and understand it.  I'll throw sample code at you when it helps.  You do need to be a developer with some experience in multithreading.  Actually the more you've suffered from multithreaded code, the more you'll enjoy this article.  I'll assume you're working on some brand of Linux but the code and explanations are pretty portable.

<A name="toc2-16" title="Getting Started" />
## Getting Started

This article is a rant combined with history lesson, and it will gently turn into a hands-on hacking tutorial.  We will explain the nitty-gritty with running code, using ANSI C because that's the language of the ØMQ API.  However it's trivial to translate these examples into the language you actually want to use.  Nothing important you see will depend on C.

To run these examples you will want to install ØMQ.  It is straight-forward and will take you just a few minutes.  Grab the latest stable package from <http://www.zeromq.com> and follow the instructions on that site.  It's `sh configure; make; sudo make install; sudo ldconfig`.  To install a language binding, check the bindings on <http://www.zeromq.org> and pick the one you want, then follow the instructions.

<center>
<img src="http://github.com/imatix/zguide/raw/masterarticles//images/multithreading_1.png" alt="1">
</center>

Create a directory to work in, and if you want to follow the examples in C, grab two files from the ØMQ Guide examples directory, [zhelpers.h][] and [zmsg.c][].  These wrap ØMQ's message API, which is a bit low-level, with nicer abstractions.

Write a minimal C program to test that you got ØMQ properly installed:

    hello.c:
    ------------------------------------------------------------------
    #include "zmsg.c"
    void main (void) {
        s_version ();
    }

To compile and link with gcc, do this:

    gcc -lzmq hello.c -o hello

And when it compiles and links without errors, and you run it, it will say something like:

    Current ØMQ version is 2.0.9

<center>
<img src="http://github.com/imatix/zguide/raw/masterarticles//images/multithreading_2.png" alt="2">
</center>

<A name="toc2-103" title="Why Write Multithreaded Code?" />
## Why Write Multithreaded Code?

Before we look at how to write multithreaded code using ØMQ, it's worth asking why we want to do this at all.  In my experience, there are two reasons why people write multithreaded code.  Three if you count random insanity, but apart from that:

1. It is a way to get concurrency, i.e. to handle many events in parallel.  A typical example would be to write a web server capable of handling many requests in parallel.
2. It is a way to get performance, i.e. to use more CPU cores in parallel.  A typical example would be a supercomputing grid capable of running thousands of tasks in parallel.

Concurrency can be solved relatively simply.  In 1996, I designed and built Xitami, a webserver capable of handling many clients in parallel.  Xitami could withstand a slashdotting on modest hardware.  Yet it ran in one OS thread, using a microkernel and green threads model. There is no need for real multithreading to get concurrency, you just need event handling that's fast enough to respond in apparent real time.

Multithreading is thus about distributing work over multiple CPUs.  An ideal MT design would let us use 100% of each core, and add cores up to any scale.  However the traditional approach to MT not only wastes a lot of CPU time with wait states, it also fails to scale beyond ten or so cores, due to increasing conflicts between threads.

One example from my own experience.  In 2005 we wrote our AMQP messaging server, OpenAMQ.  The first versions used the same virtual threading engine as Xitami, and could push 50,000 messages per second in and out (100K in total).  These are largish, complex messages, with nasty stateful processing on each message, so crunching 50K in a second was a good result.  This was 5x more than the software we were replacing.  But our goal was to process 100K per second.  So after some thought and discussion with our client, who liked multithreading, we chose the "random insanity" option and built a real multithreaded version of our engine from scratch.

The result was two years of late nights tracking down weird errors and making the code more and more complex until it was finally robust.  OpenAMQ is extremely solid nowadays, and this pain is apparently "normal" in multithreaded applications, but coming from years of doing painless pseudo-threading with Xitami's engine, it was a shock.

Worse, OpenAMQ was slower on one core and did not scale linearly.  It did 35K when running on one core, and 120K when running on four.  So the exercise was worth it, in terms of meeting our goals, but it was horribly expensive.  Worst of all, the client didn't pay for this, we did, it was a fixed price project.

<center>
<img src="http://github.com/imatix/zguide/raw/masterarticles//images/multithreading_3.png" alt="3">
</center>

Before we leaped into the piranha-infested white water of concurrency, I had this vague plan of building OpenAMQ as a cluster of single-threaded processes that would talk to each other without sharing anything.  But we lacked the tools to make that happen.

<A name="toc2-139" title="The Failure of Traditional Multithreading" />
## The Failure of Traditional Multithreading

Concurrent programming was part of my CompSci class in 1981.  That says something about my age but it also says a lot about how long a many very smart people have been trying to solve this problem.

The core problem with (conventional) concurrent programming techniques is that they all make one major assumption that just happens to be insane.  Random insanity seems to be a recurring theme in the software industry.  The assumption maybe comes from the Dijkstran theory that "software = data structures + algorithms", which makes more sense than "software = chicken soup + love" but is still flawed.  Software consisting of object classes and methods is no better, it just merges the data structures and algorithms into a single pot with multiple levels of abstraction like rice noodles.

Before we expound a less insane model of software, let's see why placing data structures (or objects) in such a central position fails to scale across multiple cores.

In the Dikstran model of software, its object-oriented successors, and even the IBM-gifted relational database, data is the golden honey comb that the little busy bees of algorithms work on.  It's all about the data structures, the relations, the indexes, the sets, and the algorithms we use to compute on these sets.  We can call this the "Data + Compute" model of software:

<center>
<img src="http://github.com/imatix/zguide/raw/masterarticles//images/multithreading_4.png" alt="4">
</center>

When two busy algorithmic bees try to modify the same data, they naturally negotiate with each other over who will go first.  It's what you would do in the real world when you 'negotiate' with a smaller, weaker car for that one remaining parking place right next to the Starbucks.  First one in wins, the other has to wait or go somewhere else.

And so the entire mainstream science of concurrent programming has focused on indicator lights, hand signals, horns, bumpers, accident reporting forms, traffic courts, bail procedures, custodial sentences, bitter divorces, alcoholic deaths, generational trauma, and all the other consequences of letting two withdrawn caffeine addicts fight over the same utterly essential parking spot.

Here, with less visual impact and more substance, is what really happens:

* Two threads cannot safely access non-trivial data at the same time.  It is possible to do atomic accesses to short chunks of memory (bytes, words) but it rapidly becomes a question of what CPU you're using.

* There are ways to access larger types like pointers atomically but these require pretty freaky assembly language 'compare and swap' instructions that are definitely not portable, and not something you want to try to teach ordinary developers.

* So, two threads need to agree in advance what data they won't touch at the same time, and for how long.  Unfortunately there is no way to say "this data is special", rather the code has to say, like every over-confident driver heading for disaster, "I'm special".  I.e. every single piece of code that wants to access some data has to do all the effort of being careful.

* Threads can do this by sending each other atomic signals, called 'semaphores'.  Or, they can raise mutual exclusion zones ('mutexes'), which are like signals to the operating system saying, "I want to be the only person in this mutex zone".  Or they can define 'critical sections' that say, "I really don't trust anyone at all, while I'm in this section of code please don't let anyone else at all here".

These techniques all try to achieve the same thing, namely safe access to shared data.  What they all end up doing is:

1. Stopping other threads that want to access the same data.  When the reason for concurrency is to get performance, stopping threads for any reason except "there is no work to do" is a Really Bad Idea.

2. Breaking, because it is impossible to eliminate bugs in such a model.  Concurrent code that works under normal loads will break under heavier loads, as developers wrongly judge the size of critical sections, forget mutexes, or find threads deadlocked like two drivers half-way into the same parking spot.

3. Getting complex, because the solution to all these problems is to add yet more untestable synchronization mechanisms.

In practice, and this is being optimistic, the best classic multithreaded applications can scale to perhaps ten threads, with around ten times the cost of writing equivalent single threaded code, and that's it.  Above ten threads, the cost of locking exceeds real work so that adding another thread will slow things down.

The only way to scale beyond single digit concurrency is to share less data.  If you share less data, or use black magic techniques like flipped data structures (you lock and modify a copy of a structure, then use an atomic compare-and-swap to flip a pointer from the live copy to your new version), you can scale further.  (That last technique serializes writers only, and lets readers work on a safe copy.)

<A name="toc2-196" title="Lots of Chatty Boxes" />
## Lots of Chatty Boxes

Reducing the conflicts over data is the way to scale concurrency.  The more you reduce the conflicts, the more you can scale.  So if there was a way to write real software with zero shared data, that would scale infinitely, right?

The answer is "yes", there is no catch.

As with many things in technology, this is not a new idea, it's simply an old one that never made the mainstream.  Many fantastically great ideas are like this, they don't build on existing (mediocre) work, so are never adopted by the market.  It's the reason we don't all light our homes with safe nuclear power, aka thorium liquid salt.  It's the reason our keyboards still come with a useless CAPS key where the Ctrl key should be.

Historically, the Data + Compute theory of software stems from the earliest days of commercial computers, when IBM estimated the global market at 5,000 computers.  The original model of computing is basically "huge big thing that does stuff":

<center>
<img src="http://github.com/imatix/zguide/raw/masterarticles//images/multithreading_5.png" alt="5">
</center>

And hardware models become software models, so the Big Iron model became Data + Compute.  But in a world where every movable object will eventually have a computer embedded in it, Data + Compute turns into the shared state dog pit where algorithms fight it out over memory that is so expensive it has to be shared.

But there are alternative models of concurrent computing than the shared state dog pit that most mainstream languages and manufacturers have adoped.  The relevant alternate reality from the early 1970's is "computing = lots of boxes that send each other messages".  As Wikipedia says of the [Actor][] model of computing:

> Unlike previous models of computation, the Actor model was inspired by physical laws. ... Its development was "motivated by the prospect of highly parallel computing machines consisting of dozens, hundreds or even thousands of independent microprocessors, each with its own local memory and communications processor, communicating via a high-performance communications network." Since that time, the advent of massive concurrency through multi-core computer architectures has rekindled interest in the Actor model.

Just as Von Neumann's Big Iron view of the world translated into software, the Actor model of lots of chatty boxes translates into a new model for software.  Instead of boxes, think "tasks".  *Software = tasks + messages*.  It turns out that this works a lot better than focusing on pretty bijoux data structures (and who doesn't enjoy crafting an elegant doubly-linked list or super-efficient hash table?)

So here's are some interesting things about the Actor model, apart from "how on earth did mainstream computer science ignore such a powerfully accurate view of software for so long":

* It's a better model of a real world with trillions of CPUs.
* It lets you create massive concurrency with no resource conflicts.
* It scales literally without limit.
* It lets you exercise every CPU to maximum capacity.

To explain again how broken the shared-state model is, imagine there was just one mobile phone directory in the world.  Forget the questions of privacy and who gets to choose which number goes with "Mommy".  Just consider the pain if access had to be serialized.  It would be like going to the post office.  "Ticket number 19,216,855 please!"  "Hello, I'd like the number of..." "HEY, I WAS HERE FIRST!!!"  "But..."  "PISS OFF, OR I'LL CLOBBER YOU!!"

Luckily every little mobile phone has its own directory, and they communicate with each other by getting their captive human slaves to send messages.  It's a great system for the mobile phones, and scales to the point where we have over 3 billion mobile phones on the planet and yet it's a fact that no-one has ever seen two mobile phones get into "road rage" over a SIM card.

But it gets better.  The actor model has more advantages over shared state concurrency:

* While shared state has very fuzzy contracts between threads, Actor thrives on contractual interfaces.  Messages are contracts (if you have even half a brain), that are easy to document, validate, and enforce.

* While shared state is insanely sensitive, like a disturbed girlfriend, to every possible aspect of the environment, Actor is insensitive to language, operating system, CPU architecture, time of day, and choice of decor.

* While shared state is sensitive to timing, and demands precise coordination and synchronization, Actor doesn't know or care.  Tasks are asynchronous and do what they do as they want to do it.

* While shared state looks calm and reliable, it cannot handle stress.  Actor on the other hand, performs as elegantly when hit by massive storms of data.  It just crunches through the work, pedantically, without slowing or stopping.

* While shared state code is complex and has many cross-thread dependencies, Actor code is serial and event driven.  It is 10-100x easier to live with... uhm... write Actor code.

* While shared state code is practically impossible to fully test, Actor code is trivial to stress test and once it works, it always works.

Ironically, the reason IBM were able to run thousands of concurrent interactive sessions on their mainframes in the 80's and 90's was that they basically reinvented the Actor model, ripped out the joy, stuck a suit and tie on what remained, and called it "CICS".  Mainframe transaction monitors turned COBOL sloths into nimble Actors.  For decades the worlds' airlines and banks depended on this to scale their applications up to handle tens of thousands of interactive users.

So, eliminate shared state, turn your application into tasks that communicate only by sending each other messages, and those tasks can run without ever locking or waiting for other tasks to make way for them.  It's kind of like discovering that hey, there are other Starbucks, other parking spaces, and frankly it's easy enough to give every Joe his own private damn city if that's what it takes to stop them fighting.

Of course you need some good connectivity between your tasks.  If you connect them with RFC1149 (avian-flu-over-TCP), they won't get much work done unless you can find a **lot** of pigeons.  And even then, the latency and droppings will wipe you out.

<A name="toc2-266" title="How to use ØMQ for Multithreading" />
## How to use ØMQ for Multithreading

ØMQ is not RFC1149.  No bird seed, no mops.  Just a small library you link into your applications.  Let's look how to send a message from one thread to another.  This program has a main thread and a child thread.  The main thread wants to know when the child thread has finished doing its work:

<code>
//
//  Show inter-thread signalling using ØMQ sockets
//
#include "zhelpers.h"

static void *
child_thread (void *context)
{
    void *socket = zmq_socket (context, ZMQ_PAIR);
    assert (zmq_connect (socket, "inproc://sink") == 0);

    s_send (socket, "happy");
    s_send (socket, "sad");
    s_send (socket, "done");

    zmq_close (socket);
    return (NULL);
}

int main ()
{
    s_version ();
    //  Threads communicate via shared context
    void *context = zmq_init (1);

    //  Create sink socket, bind to inproc endpoint
    void *socket = zmq_socket (context, ZMQ_PAIR);
    assert (zmq_bind (socket, "inproc://sink") == 0);

    //  Start child thread
    pthread_t thread;
    pthread_create (&thread, NULL, child_thread, context);

    //  Get messages from child thread
    while (1) {
        char *mood = s_recv (socket);
        printf ("You're %s\n", mood);
        if (strcmp (mood, "done") == 0)
            break;
        free (mood);
    }
    zmq_close (socket);
    zmq_term (context);
    return 0;
}
</code>

There is no direct mapping from traditional MT code to ØMQ code.  Whereas dog pit shared state threads interact in many indirect and subtle ways, ØMQ threads interact only by sending and receiving messages.


<A name="toc2-322" title="About this document" />
## About this document

This document is articles/multithreading.txt and is processed by [gitdown][].  To change, edit, run `gitdown multithreading.txt`, commit and push back to repository.

[zeromq]: http://www.zeromq.com
[iMatix]: http://www.imatix.com
[zhelpers.h]: http://github.com/imatix/zguide/blob/master/examples/C/zhelpers.h
[zmsg.c]: http://github.com/imatix/zguide/blob/master/examples/C/zmsg.c
[actor]: http://en.wikipedia.org/wiki/Actor_model#History
[gitdown]: http://github.com/imatix/gitdown

(Document is not finished)

(More coming soon...)
