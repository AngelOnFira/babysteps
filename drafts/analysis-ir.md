I've noticed that the ideas that I post on my blog are getting much
more "well rounded". That is a problem. It means I'm waiting too long
to write about things. So I want to post about something that's a bit
more half-baked -- it's an idea that I've been kicking around to
create a kind of informal "analysis API" for rustc.

### The problem statement

I am interested in finding better ways to support advanced analyses
that "layer on" to rustc. I am thinking of projects like Prusti,
Facebook's XXX, or the work to extend Galois's Sawzall. Most of these
projects are attempting to analyze **safe Rust code** and prove useful
properties about it. So, for example, they might try to show that a
certain piece of code will never panic, or to show that it meets
certain functional contracts specified in comments.

### In theory, Rust is a great fit for analysis

You'll notice that several of the tools I mentioned above are
pre-existing tools. Prusti, for example, is adapting an existing
project called Viper, which was built to analyze languages like C# or
Java. However, actually doing that analysis *in practice* is often
quite difficult, precisely because of the kinds of pervasive, mutable
aliasing that those languages encourage.

Pervasive aliasing means that if you see code like

```java
a.setCount(0);
```

it can be quite difficult to be sure whether that call might also
modify the state of some variable `b` that happens to be floating
around. If you are trying to enforce contracts like "in order to call
this method, the count must be greater than zero", then it's important
to know which variables are affected by calls like `setCount`.

Rust's ownership/borrowing system can be really helpful here. The
borrow checker rules ensure that it's fairly easy to see what data a
given Rust function might read or mutate. This is of course the key to
how Rust is able to steer you away from data races and segmentation
faults -- but the key insight here is that those same properties can
also be used to make higher-level correctness guarantees. Even better,
many of the more complex analyses that analysis tools might need --
e.g., alias analysis -- map fairly well onto what the Rust compile
already does.

### In practice, analyzing Rust is a pain

Unfortunately, while Rust ought to be a great fit for analysis tools,
it's a horrible pain to try and implement such a tool **in practice**.
The existing tooling just wasn't built with this sort of use case in mind.

Ideally, I think what we would want is some way for analyzer tools to
leverage the compiler itself. They ought to be able to use the
compiler to do the parsing of Rust code, to run the borrow check, and
to consrtuct MIR. They should then be able to access the [MIR] and the
accompanying borrow check results and use that to construct their own
internal IRs (in practice, virtually all such verifiers would prefer
to start from an abstraction level like MIR, and not from a raw Rust
AST). They should be able to ask the compiler for information about
the layout of data structures in memory and other things they might
need, too, or for information about the type signature of other
methods.

### Enter: on-demand analysis and library-ification

A few years back, the idea of enabling analysis tools to interact with
the compiler and request this sort of detailed information would have
seemed like a fantasy. But the architectural work that we've been
doing lately is actually quite a good fit for this use case.

I'm referring to two different trends:

* on-demand analysis 
* library-ification

### On-demand analysis

On-demand analysis is basically the idea that we should structure the
compiler's internal core into a series of "queries". Each query is a
pure function from some inputs to an output, and it might be something
like "parse this file" (yielding an AST) or "type-check this function"
(yielding a set of errors). The key idea is that each query can in
turn invoke other queries, and thus execution begins from the *end
state* that we want to reach ("give me an executable") and works its
way *backwards* to the first few steps ("parse this file"). This winds
up fitting quite nicely with incremental computation as well as
parallel execution. It's also a great fit for IDEs, since it allows us
to do "just as much work" aswe have to" in order to figure out key
bits of information (e.g., "what is the type of the expression at the
cursor").