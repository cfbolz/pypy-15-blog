The First 15 Years of PyPy — a Personal Retrospective
========================================================

A few weeks ago I (=Carl Friedrich Bolz-Tereick) gave a keynote_ at ICOOOLPS in
Amsterdam with the above title. I was very happy to have been given that
opportunity, since a number of our papers have been published at ICOOOLPS,
including the very first one I published when I'd just started my PhD. I decided
to turn the talk manuscript into a (longish) blog post, to make it available to a wider audience.
Note that this blog post describes my personal recollections and research, it is
thus necessarily incomplete and coloured by my own experiences.

.. _keynote: https://conf.researchr.org/event/ecoop-issta-2018/icooolps-2018-papers-tbd-15-years-of-pypy-a-retrospective

PyPy has turned 15 years old this year, so I decided that that's a good reason
to dig into and talk about the history of the project so far. I'm going to do
that using the lens of how performance developed over time, which is from
something like 2000x slower than CPython, to roughly 7x faster. In this post
I am going to present the history of the project, and also talk about some
lessons that we learned.

The post does not make too many assumptions about any prior knowledge of what
PyPy is, so if this is your first interaction with it, welcome! I have tried to
sprinkle links to earlier blog posts and papers into the writing, in case you
want to dive deeper into some of the topics.

As a disclaimer, in this post I am going to mostly focus on ideas, and not
explain who had or implemented them. A huge amount of people contributed to the
design, the implementation, the funding and the organization of PyPy over the
years, and it would be impossible to do them all justice.


.. contents::

2003: Starting the Project
----------------------------

On the technical level PyPy is a Python interpreter written in Python, which is
where the name comes from. It also has an automatically generated JIT compiler,
but I'm going to introduce that gradually over the rest of the blog post, so
let's not worry about it too much yet. On the social level PyPy is an
interesting mixture of a open source project, that sometimes had research done
in it.

The project got started in late 2002 and early 2003. To set the stage, at that
point Python was a significantly less popular language than it is today. `Python
2.2`_ was the version at the time, Python didn't even have a ``bool`` type yet.

.. _`Python 2.2`: https://www.python.org/download/releases/2.2/

In fall 2002 the PyPy project was started by a number of Python programmers on a
mailing list who said
something like (I am exaggerating somewhat) "Python is the greatest most
wonderful most perfect language ever, we should use it for absolutely
everything. Well, what aren't we using it for? The Python virtual machine itself
is written in C, that's bad. Let's start a project to fix that."

Originally that project was called "minimal python", or "ptn", later gradually
renamed to PyPy. Here's the `mailing list post`_ to announce the project more
formally::

    Minimal Python Discussion, Coding and Sprint
    --------------------------------------------

    We announce a mailinglist dedicated to developing
    a "Minimal Python" version.  Minimal means that
    we want to have a very small C-core and as much
    as possible (re)implemented in python itself.  This
    includes (parts of) the VM-Code.


.. _`mailing list post`: https://mail.python.org/pipermail/python-list/2003-January/235289.html


Why would that kind of project be useful? Originally it wasn't necessarily meant
to be useful as a real implementation at all, it was more meant as a kind of
executable explanation of how Python works, free of the low level details of
CPython. But pretty soon there were then also plans for how the virtual machine
(VM) could be bootstrapped to be runnable without an existing Python
implementation, but I'll get to that further down.


2003: Implementing the Interpreter
------------------------------------

In early 2003 a group of Python people met in Hildesheim (Germany) for the first
of many week long development sprints, organized by Holger Krekel. During that
week a group of people showed up and started working on the core interpreter.
May 2003 a second sprint was organized by Laura Creighton and Jacob Halén in
Gothenburg (Sweden). And already at that sprint enough of the Python bytecodes
and data structures were implemented to make it possible to run a program that
computed how much money everybody had to pay for the food bills of the week. And
everybody who's tried that for a large group of people knows that that’s an
amazingly complex mathematical problem.

In the next two years, the project continued as a open source project with
various contributors working on it in their free time, and meeting for the
occasional sprint. In that time, the rest of the core interpreter and the core
data types were implemented.

There's not going to be any other code in this post, but to give a bit of a
flavor of what the Python interpreter at that time looked like, here's the
implementation of the ``DUP_TOP`` bytecode after these first sprints. As you can
see, it's in Python, obviously, and it has high level constructs such as method
calls to do the stack manipulations::

    def DUP_TOP(f):
        w_1 = f.valuestack.top()
        f.valuestack.push(w_1)


Here's the early code for integer addition::

    def int_int_add(space, w_int1, w_int2):
        x = w_int1.intval
        y = w_int2.intval
        try:
            z = x + y
        except OverflowError:
            raise FailedToImplement(space.w_OverflowError,
                                    space.wrap("integer addition"))
        return W_IntObject(space, z)

(the current_ implementations_ look slightly but not fundamentally different.)

.. _current: https://bitbucket.org/pypy/pypy/src/090965df249b458918fbcbd0a407b40d2a3d29b4/pypy/interpreter/pyopcode.py?at=default&fileviewer=file-view-default#pyopcode.py-577
.. _implementations: https://bitbucket.org/pypy/pypy/src/090965df249b458918fbcbd0a407b40d2a3d29b4/pypy/objspace/std/intobject.py?at=default&fileviewer=file-view-default#intobject.py-561

Early organizational ideas
-----------------------------

Some of the early organizational ideas of the project were as follows. Since the
project was started on a sprint and people really liked that style of working
PyPy continued to be developed on various subsequent sprints.

From early on there was a very heavy emphasis on testing. All the parts of the
interpreter that were implemented had a very careful set of unit tests to make
sure that they worked correctly. At the sprints there was also an emphasis on
doing pair programming to make sure that everybody understood the codebase
equally. There was also a heavy emphasis on writing good code and on regularly
doing refactorings to make sure that the codebase remained nice, clean and
understandable. Those ideas followed from the early thoughts that PyPy would be
a sort of readable explanation of the language.

There was also a pretty fundamental design decision made at the time. That was
that the project should stay out of language design completely. Instead it would
follow CPython's lead and behave exactly like that implementation in all cases.
The project therefore committed to being almost quirk-to-quirk compatible and to
implement even the more obscure (and partially unnecessary) corner cases of
CPython.

All of these principles continue pretty much still today (There are a few places
where we had to deviate from being completely compatible, they are documented
here__).

.. __: http://doc.pypy.org/en/latest/cpython_differences.html


2004-2007: EU-Funding
----------------------

While all this coding was going on it became clear pretty soon that the goals
that various participants had for the project would be very hard to achieve with
just open source volunteers working on the project in their spare time.
Particularly also the sprints became expensive given that those were just
volunteers doing this as a kind of weird hobby. Therefore a couple of people of
the project got together to apply for an EU grant in the framework programme 6
to solve these money problems. In mid-2004 that application proved to be
successful. And so the project got a grant of a 1.3 million Euro for
two years to be able to employ some of the core developers and to make it
possible for them work on the project full time. The EU grant went to seven
small-to-medium companies and Uni Düsseldorf. The budget also contained money to
fund sprints, both for the employed core devs as well as other open source
contributors.

The EU project started in December 2004 and that was a fairly heavy change in
pace for the project. Suddenly a lot of people were working full time on it, and
the pace and the pressure picked up quite a lot. Originally it had been a
leisurely project people worked on for fun. But afterwards people discovered
that doing this kind of work full time becomes slightly less fun, particularly
also if you have to fulfill the ambitious technical goals that the EU proposal
contained. And the proposal indeed contained a bit everything to increase its
chance of acceptance, such as aspect oriented programming, semantic web, logic
programming, constraint programming, and so on. Unfortunately it
turned out that those things then have to be implemented, which can be called
the first thing we learned: if you promise something to the EU, you'll have to
actually go do it (After the funding ended, a lot of these features were
actually removed from the project again, at a `cleanup sprint`_).

.. _`cleanup sprint`: https://morepypy.blogspot.com/2007/11/sprint-pictures.html


2005: Bootstrapping PyPy
----------------------------

So what were the actually useful things done as part of the EU project?

One of the most important goals that the EU project was meant to solve was the
question of how to turn PyPy into an actually useful VM for Python. The
bootstrapping plans were taken quite directly from Squeak_, which is a Smalltalk
VM written in a subset of Smalltalk called Slang, which can then be bootstrapped
to C code. The plan for PyPy was to do something similar, to define a restricted
subset of Python called RPython, restricted in such a way that it should be
possible to statically compile RPython programs to C code. Then the Python
interpreter should only use that subset, of course.

.. _Squeak: http://wiki.squeak.org/squeak

The main difference to the Squeak approach is that Slang, the subset of Squeak
used there, is actually quite a low level language. In a way, you could almost
describe it as C with Smalltalk syntax. And RPython was really meant to be a
much higher level language, much closer to Python, with full support for single
inheritance classes, and most of Python's built-in data structures.

$$ image

(BTW, you don’t have to understand any of the illustrations in this blog post,
they are taken from talks and project reports we did over the years so they are
of archaeological interest only and I don’t understand most of them myself.)

From 2005 on, work on the RPython type inference engine and C backend started in
earnest, which was sort of co-developed with the RPython language definition and
the PyPy Python interpreter. This is also roughly the time that I joined the
project as a volunteer.

And at the second sprint I went to, in July 2005, two and a half years after the
project got started, we managed to bootstrap the PyPy interpreter to C for the
first time. And then when we ran the compiled program, it of course immediately
segfaulted. The reason for that was that the C backend had turned characters
into signed chars in C, while the rest of the infrastructure assumed that they
were unsigned chars. After we fixed that, the second attempt worked and we
managed to run an incredibly complex program, something like ``6 * 7``. That
first bootstrapped version was really really slow, a couple of hundred times
slower than CPython.

$$ image champagne

RPython's Modularity Problems
--------------------------------

Now we come to the first thing I would say we learned in the project, which is
that the quality of tools we thought of as internal things still matters a lot.
One of the biggest technical mistakes we've made in the project was that we
designed RPython without any kind of story for modularity. There is no concept
of modules in the language or any other way to break up programs into smaller
components. We always thought that it would be ok for RPython to be a little bit
crappy. It was meant to be this sort of internal language with not too many
external users. And of course that turned out to be completely wrong later.

That lack of modularity led to various problems that persist until today. The
biggest one is that there is no separate compilation for RPython programs at
all! You always need to compile all the parts of your VM together, which leads
to infamously bad compilation times.

Also by not considering the modularity question we were never forced to fix
some internal structuring issues of the RPython compiler itself.
Various layers of the compiler keep very badly defined and porous interfaces between
them. This was made possible by being able to work with all the program information in one heap,
making the compiler less approachable and maintainable than it maybe could be.

Of course this mistake just got more and more costly to fix over time, 
and so it means that so far nobody has actually done it. 
Not thinking more carefully about RPython's design, particularly its
modularity story, is in my opinion the biggest technical mistake the project
did.


2006: The Meta-JIT
-------------------

After successfully bootstrapping the VM we did some fairly straightforward
optimizations on the interpreter and the C backend and managed to reduce the
slowdown versus CPython to something like 2-5 times slower. That's great! But of
course not actually useful in practice. So where do we go from here?

One of the not so secret goals of Armin Rigo, one of the PyPy founders, was to
use PyPy together with some advanced `partial evaluation`_ magic sauce to
somehow automatically generate a JIT compiler from the interpreter. The goal was
something like, "you write your interpreter in RPython, add a few annotations
and then we give you a JIT for free for the language that that interpreter
implements." 

.. _`partial evaluation`: https://en.wikipedia.org/wiki/Partial_evaluation

Where did the wish for that approach come from, why not just write a JIT for
Python manually in the first place? Armin had actually done just that before he
co-founded PyPy, in a project called Psyco_. Psyco was an extension module for
CPython that contained a method-based JIT compiler for Python code. And Psyco
proved to be an amazingly frustrating compiler to write. There were two main
reasons for that. The first reason was that Python is actually quite a complex
language underneath its apparent simplicity. The second reason for the
frustration was that Python was and is very much an alive language, that gains
new features in the language core in every version. So every time a new Python
version came out, Armin had to do fundamental changes and rewrites to Psyco, and
he was getting pretty frustrated with it. So he hoped that that effort could be
diminished by not writing the JIT for PyPy by hand at all. Instead, the goal was
to generate a method-based JIT from the interpreter automatically. By taking the
interpreter, and applying a kind of advanced transformation to it, that would
turn it into a method-based JIT. And all that would still be translated into a
C-based VM, of course.

.. _Psyco: http://psyco.sourceforge.net/


$$ image from Psyco presentation at EuroPython 2002


The First JIT Generator
----------------------------

From early 2006 on until the end of the EU project a lot of work went into
writing such a JIT generator. The idea was to base it on runtime partial
evaluation. Partial evaluation is an old idea in computer science. It's supposed
to be a way to automatically turn interpreters for a language into a compiler
for that same language. Since PyPy was trying to generate a JIT compiler, which
is in any case necessary to get good performance for a dynamic language like
Python, the partial evaluation was going to happen at runtime.

There are various ways to look at partial evaluation, but if you've never heard
of it before, a simple way to view it is that it will compile a Python function
by gluing together the implementations of the bytecodes of that function and
optimizing the result.

The main new ideas of PyPy's partial-evaluation based JIT generator as opposed
to earlier partial-evaluation approaches are the ideas of "promote" and the idea
of "virtuals". Both of these techniques had already been present (in a slightly
less general form) in Psyco, and the goal was to keep using them in PyPy. Both
of these techniques also still remain in use today in PyPy. I'm
going on a slight technical diversion now, to give a high level explanation of
what those ideas are for. 

$$ image timeshifter


Promote
--------

One important ingredient of any JIT compiler is the ability to do runtime
feedback. Runtime feedback is most commonly used to know something about which
concrete types are used by a program in practice. Promote is basically a way to
easily introduce runtime feedback into the JIT produced by the JIT generator.
It's an annotation_ the implementer of a language can use to express their wish
that specialization should happen at *this* point. This mechanism can be used to
express `all kinds of`_ runtime feedback, moving values from the interpreter
into the compiler, whether they be types or other things.

.. _annotation: https://morepypy.blogspot.com/2011/03/controlling-tracing-of-interpreter-with_15.html
.. _`all kinds of`: https://morepypy.blogspot.com/2011/03/controlling-tracing-of-interpreter-with_21.html


Virtuals
----------

Virtuals are a very aggressive form of `partial escape analysis`_. A dynamic
language often puts a lot of pressure on the garbage collector, since most
primitive types (like integers, floats and strings) are boxed in the heap, and
new boxes are allocated all the time.

With the help of virtuals a very significant portion of all allocations in the
generated machine code can be removed fully. And even if they can't be removed
fully, often the allocation can be delayed, or moved into an error path or even
into a deoptimization_ path and thus disappear from the generated machine code
completely.

.. _`partial escape analysis`: http://www.ssw.uni-linz.ac.at/Research/Papers/Stadler14/Stadler2014-CGO-PEA.pdf
.. _deoptimization: http://bibliography.selflanguage.org/_static/dynamic-deoptimization.pdf

This optimization really is the super-power of PyPy's optimizer, since it
doesn't work only for primitive boxes but for any kind of object allocated on
the heap with predictable lifetime.

As an aside, while this kind of partial escape analysis is sort of new for
object-oriented languages, it has actually existed in Prolog-based partial
evaluation systems since the 80s, because it's just extremely natural there.


JIT Status 2007
-----------------

So, back to our history. We're now in 2007, at the end of the EU project (you
can find the EU-reports we wrote during the projects here_). The EU project
successfully finished, we survived the final review with the EU. So, what's the
status of the JIT generator? It works kind of, it can be applied to PyPy. It
produces a VM with a JIT that will turn Python code into machine code at runtime
and run it. However, that machine code is not particularly fast. Also, it tends
to generate many megabytes of machine code even for small Python programs. While
it's always faster than PyPy without JIT, it's only sometimes faster than
CPython, and most of the time Psyco still beats it. On the one hand, this is
still an amazing achievement! It's arguably the biggest application of partial
evaluation at this point in time! On the other hand, it's still quite 
disappointing in practice, particularly in the context of our assumption that
it should have been possible to surpass the speed of Psyco with the approach.

.. _here: http://doc.pypy.org/en/latest/index-report.html


2007: RSqueak and other languages
-------------------------------------

After the EU project ended we did all kinds of things. Like sleep for a month
for example, and have the cleanup sprint that I already mentioned. We also had a
slightly unusual sprint in Bern, with members of the `Software Composition
Group`_ of Oscar Nierstrasz. As I wrote above, PyPy had been heavily influenced
by Squeak Smalltalk, and that group is a heavy user of Squeak, so we wanted to
see how to collaborate with them. At the beginning of the sprint, we decided
together that the goal of that week should be to try to write a Squeak virtual
machine in RPython, and at the end of the week we'd gotten surprisingly far with
that goal. Basically most of the bytecodes and the Smalltalk object system
worked, we had written an image loader and could run some benchmarks (during the
sprint we also regularly updated a blog_, the success of which led us to start_
the PyPy blog).

.. _blog: http://pypysqueak.blogspot.com/

.. _start: https://morepypy.blogspot.com/2007/10/first-post.html

The development of the Squeak interpreter was very interesting for the project,
because it was the first real step that moved RPython from being an
implementation detail of PyPy to be a more interesting project in its own right.
Basically a language to write interpreters in, with the eventual promise to get
a JIT for that language sort of for free. That Squeak implementation is now
called RSqueak_ ("Research Squeak").

.. _`Software Composition Group`: http://scg.unibe.ch/
.. _RSqueak: https://github.com/hpi-swa/RSqueak

I'll not go into more details about any of the other language implementations in
RPython in this post, but over the years we've had a large variety of language
of them done by various people and groups, most of them as research vehicles,
but also some as real language implementations. Some very cool research results
came out of these efforts, here's a slightly outdated `list of some of them`_.

.. _`list of some of them`: https://rpython.readthedocs.io/en/latest/examples.html


The use of RPython for other languages complicated the PyPy narrative a lot, and
in a way we never managed to recover the simplicity of the original project
description "PyPy is Python in Python". Because now it's something like "we have
this somewhat strange language, a subset of Python, that's called RPython, and
it's good to write interpreters in. And if you do that, we'll give you a JIT for
almost free. And also, we used that language to write a Python implementation,
called PyPy.". It just doesn't roll off the tongue as nicely.




2008-2009: Four More JIT Generators
------------------------------------

Back to the JIT. After writing the first JIT generator as part of the EU
project, with somewhat mixed results, we actually wrote several more JIT
generator prototypes with different architectures to try to solve some of the
problems of the first approach. To give an impression of these prototypes,
here’s a list of them.

- The second JIT generator we started working on in 2008 behaved exactly like
  the first one, but had a meta-interpreter based architecture, to make it more
  flexible and easier to experiment with.

- The third one was an experiment based on the second one which changed
  compilation strategy. While the previous two had compiled many control flow
  paths of the currently compiled function eagerly, that third JIT was sort of
  maximally lazy and stopped compilation at every control flow split to avoid
  guessing which path would actually be useful later when executing the code.
  This was an attempt to reduce the problem of the first JIT generating way too
  much machine code. Only later, when execution went down one of the not yet
  compiled paths would it continue compiling more code.

- The fourth JIT generator was a pretty strange prototype, a runtime partial
  evaluator for Prolog, to experiment with various specialization trade-offs. It
  had an approach that we gave a not at all humble name, called "perfect
  specialization". $$$

- The fifth JIT generator is the one that we are still using today. Instead of
  generating a method-based JIT compiler from our interpreter we switched to
  generating a tracing JIT compiler. Tracing JIT compilers were sort of the
  latest fashion at the time, at least for a little while.


2009: Meta-Tracing
----------------------

So, how did that tracing JIT generator work? A `tracing JIT`_ generates code by
observing and logging the execution of the running program. This yields a
straight-line trace of operations, which are then optimized and compiled into
machine code. Of course most tracing systems mostly focus on tracing loops.

.. _`tracing JIT`: https://en.wikipedia.org/wiki/Tracing_just-in-time_compilation

As we discovered, it's actually quite simple to `apply a tracing JIT to a generic
interpreter`_, by not tracing the execution of the user program directly, but by
instead tracing the execution of the interpreter while it is running the user
program (here's the paper_ we wrote about this approach).

.. _`apply a tracing JIT to a generic interpreter`: https://morepypy.blogspot.com/2009/03/applying-tracing-jit-to-interpreter.html
.. _paper: https://bitbucket.org/pypy/extradoc/raw/default/talk/icooolps2009/bolz-tracing-jit-final.pdf

So that's what we implemented. Of course we kept the two successful parts of the
first JIT, promote_ and virtuals_ (both links go to the papers about these
features in the meta-tracing context). 

.. _promote: https://bitbucket.org/pypy/extradoc/raw/default/talk/icooolps2011/bolz-hints-final.pdf
.. _virtuals: https://bitbucket.org/pypy/extradoc/raw/default/talk/pepm2011/escape-tracing.pdf


Why did we Abandon Partial Evaluation?
--------------------------------------

So one question I get asked quite regularly when telling this story is, why did
we think that tracing would work better than partial evaluation (PE)? One of the
hardest parts of compilers in general and partial evaluation based systems in
particular is the decision when and how much to inline, how much to specialize,
as well as the decision when to split control flow paths. In the PE based JIT
generator we never managed to control that question. Either the JIT would
inline too much, leading to useless compilation of all kinds of unlikely error
cases. Or it wouldn't inline enough, preventing necessary optimizations.

Meta tracing solves this problem with a hammer, it doesn't make particularly
complex inlining decisions at all. It instead decides what to inline by
precisely following what a real execution through the program is doing. Its
inlining decisions are therefore very understandable and predictable, and it
basically only has one heuristic based on whether the called function contains a
loop or not: If the called function contains a loop, we'll never inline it, if
it doesn't we always try to inline it. That predictability is actually what was
the most helpful, since it makes it possible for interpreter authors to
understand why the JIT did what it did and to actually influence its inlining
decisions by changing the annotations in the interpreter source.


2009-2011: The PyJIT Eurostars Project
-------------------------------------------

While we were writing all these JIT prototypes, PyPy had sort of reverted back
to being a volunteer-driven open source project (although some of us, like
Antonio Cuni and I, had started working for universities and other project
members had other sources of funding). But again, while we did the work it
became clear that to get an actually working fast PyPy with generated JIT we
would need actual funding again for the project. So we applied to the EU again,
this time for a much smaller project with less money, in the Eurostars_
framework. We got a grant for three participants, merlinux, OpenEnd and Uni
Düsseldorf, on the order of a bit more than half a million euro. That money was
specifically for JIT development and JIT testing infrastructure.

.. _Eurostars: https://morepypy.blogspot.com/2010/12/oh-and-btw-pypy-gets-funding-through.html


Tracing JIT improvements
------------------------------

When writing the grant we had sat together at a sprint and discussed extensively
and decided that we would not switch JIT generation approaches any more. We all
liked the tracing approach well enough and thought it was promising. So instead
we agreed to try in earnest to make the tracing JIT really practical. So in the
Eurostars project we started with implementing sort of fairly standard JIT
compiler optimizations for the meta-tracing JIT, such as:

- constant folding
- dead code elimination
- `loop invariant code motion`_ (taking LuaJIT's approach)
- better heap optimizations
- faster deoptimization (which is actually a bit of a mess in the
  meta-approach)
- and dealing more efficiently with Python frames objects and the
  features of Python's debugging facilities

.. _`loop invariant code motion`: https://bitbucket.org/pypy/extradoc/raw/default/talk/dls2012/dls04-ardo.pdf


2010: speed.pypy.org
----------------------

In 2010, to make sure that we wouldn't accidentally introduce speed regressions
while working on the JIT, we implemented infrastructure to build PyPy and run
our benchmarks nightly. Then, the http://speed.pypy.org website was implemented
by Miquel Torres, a volunteer. The website shows the changes in benchmark
performance compared to the previous *n* days. It didn't sound too important at
first, but this was (and is) a fantastic tool, and an amazing motivator over the
next years, to keep continually improving performance.


Continuous Integration
-----------------------

This actually leads me to something else that I'd say we learned, which is that
continuous integration is really awesome, and completely transformative to have
for a project. This is not a particularly surprising insight nowadays in the
open source community, it's easy to set up continuous integration on github
using Travis or some other CI service. But I still see a lot of research
projects that don't have tests, that don't use CI, so I wanted to mention it
anyway. As I mentioned earlier in the post, PyPy has a quite serious testing
culture, with unit tests written for new code, regression tests for all bugs,
and integration tests using the CPython test suite. Those tests are `run
nightly`_ on a number of architectures and operating systems.

.. _`run nightly`: http://buildbot.pypy.org/

Having all this kind of careful testing is of course necessary, since PyPy is
really trying to be a Python implementation that people actually use, not just
write papers about. But having all this infrastructure also had other benefits,
for example it allows us to trust newcomers to the project very quickly.
Basically after your first patch gets accepted, you immediately get commit
rights to the PyPy repository. If you screw up, the tests (or the code reviews)
are probably going to catch it, and that reduction to the barrier to
contributing is just super great.

This concludes my advertisement for testing in this post.


2010: Implementing Python Objects with Maps
-----------------------------------------------

So, what else did we do in the Eurostars project, apart from adding traditional
compiler optimizations to the tracing JIT and setting up CI infrastructure?
Another strand of work, that went on sort of concurrently to the JIT generator
improvements were deep rewrites in the Python runtime, and the Python data
structures. I am going to write about two exemplary ones here.

The first such rewrite is fairly standard. Python instances are similar to
Javascript objects, in that you can add arbitrary attributes to them at runtime.
Originally Python instances were backed by a dictionary in PyPy, but of course
in practice most instances of the same class have the same set of attribute
names. Therefore we went and implemented `Self style maps`_, which are often
called `hidden classes`_ in the JS world to represent instances instead. This
has two big benefits, it allows you to generate much better machine code for
instance attribute access and makes instances use a lot less memory.

.. _`Self style maps`: https://morepypy.blogspot.com/2010/11/efficiently-implementing-python-objects.html
.. _`hidden classes`: https://richardartoul.github.io/jekyll/update/2015/04/26/hidden-classes.html


2011: Container Storage Strategies
--------------------------------------

Another important change in the PyPy runtime was rewriting the Python container
data structures, such as lists, dictionaries and sets. A fairly straightforward
observation about how those are used is that in a significant percentage of
cases they contain type-homogeneous data. As an example it's quite common to
have lists of only integers, or lists of only strings. So we changed the list,
dict and set implementations to use something we called `storage strategies`_. With
storage strategies these data structures use a more efficient representations if
they contain only primitives of the same type, such as ints, floats, strings.
This makes it possible to store the values without boxing them in the underlying
data structure. Therefore read and write access are much faster for such type
homogeneous containers. Of course when later another data type gets added to
such a list, the existing elements need to all be boxed at that point, which is
expensive. But we did a study_ and found out that that happens quite rarely in
practice. A lot of that work was done by Lukas Diekmann.

.. _`storage strategies`: https://morepypy.blogspot.com/2011/10/more-compact-lists-with-list-strategies.html
.. _study: http://tratt.net/laurie/research/pubs/html/bolz_diekmann_tratt__storage_strategies_for_collections_in_dynamically_typed_languages/


Deep Changes in the Runtime are Necessary
-------------------------------------------

These two are just two examples for a number of fairly fundamental changes in
the PyPy runtime and PyPy data structures, probably the two most important ones,
but we did many others. That leads me to another thing we learned. If you want
to generate good code for a complex dynamic language such as Python, it's
actually not enough at all to have a good code generator and good compiler
optimizations. That's not going to help you, if your runtime data-structures
aren't in a shape where it's possible to generate efficient machine code to
access them.

Maybe this is well known in the VM and research community. However it's the main
mistake that in my opinion every other Python JIT effort has made in the last 10
years, where most projects said something along the lines of "we're not
changing the existing CPython data structures at all, we'll just let LLVM
inline enough C code of the runtime and then it will optimize all the overhead
away". That never works very well.


JIT Status 2011
--------------------

So, the Eurostars project has ended, what's the status of the JIT? Well, it
seems this meta-tracing stuff really works! We finally started actually
believing in it, when we reached the point in 2010 where self-hosting PyPy was
actually faster__ than bootstrapping the VM on CPython. Speeding up the
bootstrapping process is something that Psyco never managed at all, so we
considered this a quite important achievement. At the end of
Eurostars, we were about 4x faster than CPython on our set of benchmarks.

.. __: https://morepypy.blogspot.com/2010/11/snake-which-bites-its-tail-pypy-jitting.html


2012-2017: Engineering and Incremental Progress
--------------------------------------------------

2012 the Eurostars project was finished and PyPy reverted yet another time back
to be an open source project. From then on, we've had a more diverse set of
sources of funding: we received some crowd funding via the `Software Freedom
Conservancy`_ and contracts of various sizes from companies to implement various
specific features, often handled by Baroque Software. So in the next couple of
years
we revamped various parts of the VM. We improved the GC in major_ ways. We
optimized the implementation of the JIT compiler to improve warmup_ times_. We
implemented backends for various CPU architectures (including PowerPC_ and
s390x_). We tried to reduce the number of performance cliffs and make the JIT
useful in a broader set of cases. And we pushed quite significantly to be more
compatible with CPython, particularly the Python 3 line as well as extenion
module support.

.. _`Software Freedom Conservancy`: https://sfconservancy.org/
.. _major: https://morepypy.blogspot.com/2013/10/incremental-garbage-collector-in-pypy.html
.. _warmup: https://morepypy.blogspot.com/2015/10/pypy-memory-and-warmup-improvements-2.html
.. _times: https://morepypy.blogspot.com/2016/04/warmup-improvements-more-efficient.html
.. _PowerPC: https://morepypy.blogspot.com/2015/10/powerpc-backend-for-jit.html
.. _s390x: https://morepypy.blogspot.com/2016/04/pypy-enterprise-edition.html

CPyExt
---------

Another very important strand of work that took a lot of effort in recent years
was CPyExt. One of the main blockers of PyPy adoption had always been the fact
that a lot of people need specific C-extension modules at least in some parts of
their program, and telling them to reimplement everything in Python is just not
a practical solution. Therefore we worked on CPyExt, an emulation layer  to make
it possible to run `CPython C-extension modules`_ in PyPy. Doing that was a very
painful process, since the CPython extension API leaks a lot of CPython
implementation details, so we had to painstakingly emulate all of these details
to make it possible to run extensions. That this works at all remains completely
amazing to me! But nowadays CPyExt is even getting quite good, a lot of the big
numerical libraries such as Numpy and Pandas are now supported.

.. _`CPython C-extension modules`: https://morepypy.blogspot.com/2010/04/using-cpython-extension-modules-with.html

Python 3
---------

Another main
focus of the last couple of years has been to catch up with the CPython 3 line.
Originally we had ignored Python 3 for a little bit too long, and were trailing
several versions behind. In 2016 and 2017 we had a grant_ from the Mozilla open
source support program of $200000 to be able to catch up with Python 3.5. This
work is now basically done, and we are starting to target CPython 3.6 and will
have to look into 3.7 in the near future.

.. _grant: https://morepypy.blogspot.com/2016/08/pypy-gets-funding-from-mozilla-for.html


Incentives of OSS compared to Academia
------------------------------------------------------

So, what can be learned from those more recent years? One thing we can observe
is that a lot of the engineering work we did in that time is not really science
as such. A lot of the VM techniques we implemented are kind of well known, and
catching up with new Python features is also not particularly deep researchy
work. Of course this kind of work is obviously super necessary if you want
people to use your VM, but it would be very hard to try to get research funding
for it. PyPy managed quite well over its history to balance phases of more
research oriented work, and more product oriented ones. But getting this balance
somewhat right is not easy, and definitely also involves a lot of luck. And, as
has been discussed a lot, it's actually very hard to find funding for open
source work, both within and outside of academia.


Meta-Tracing really works!
------------------------------

Let me end with what's in my mind the main positive technical result of PyPy the
project. Which is that the whole idea of using a meta-tracing JIT can really
work! Currently PyPy is about 7 times faster than CPython on a broad set of
benchmarks. Also, one of the very early motivations for using a meta-jitting
approach in PyPy, which was to not have to adapt the JIT to new versions of
CPython proved to work: indeed we didn't have to change anything in the JIT
infrastructure to support Python 3.

RPython has also worked and improved performance for a number of other
languages. Some of these interpreters had wildly different architectures.
AST-based interpreters, bytecode based, CPU emulators, really inefficient
high-level ones that allocate continuation objects all the time, and so on. This
shows that RPython also gives you a lot of freedom in deciding how you want to
structure the interpreter and that it can be applied to languages of quite
different paradigms.

I'll end with a list of the people that have contributed code to PyPy over its
history, more than 350 of them. I'd like to thank all of them and the various
roles they played. To the next 15 years!

$$$ images


Acknowledgements
----------------------

A lot of people helped me with this blog post. Tim Felgentreff made me give the
keynote, which lead me to start collecting the material. Samuele Pedroni
gave essential early input when I just started planning the talk, and also gave
feedback on the blog post. Maciej Fijałkowski gave me feedback on the post, in
particular important insight about the more recent years of the project. $$$
more
All remaining errors are of course my own.
