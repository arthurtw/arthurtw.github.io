---
title: Rust, an Anti-Sloppy Programming Language
layout: post
---

> Disclaimer: The taste for programming languages is very subjective, so is this blog post. Please take it with a grain of salt.
>
> EDIT (Dec-22): Modified some wording - GC in the cons; dynamic language comparison. Mentioned async I/O in web development section.

Several new programming languages emerged recently. Among them, [Rust](http://www.rust-lang.org/) particularly interests me. In this blog, I’m going to share my impressions of Rust, and compare it with some other programming languages.

## The barrier to learn Rust

It took me a few tries to jump into Rust. There exist some barriers to learn Rust, such as:

1. **The language has been changing rapidly.** Rust has no BDFL. It evolves from the contributions of a core team and the community. Before it hits 1.0, it’s been a bit difficult to chase the moving target. Fortunately, [1.0 alpha](http://blog.rust-lang.org/2014/12/12/1.0-Timeline.html) is weeks away and things shall settle down quickly.

1. Related to the above, **Rust’s learning resource is very limited.** [The guide](http://doc.rust-lang.org/guide.html) along with other [official documentations](http://doc.rust-lang.org/index.html) and [Rust by Example](http://rustbyexample.com/) are great. However, Rust is more complicated than that. You often have to sift through RFCs, blogs or even Github comments to find out the information you need, and are still unsure if what you read are the latest. Even post 1.0, it will take a while before sufficient learning resources appear. (I’m also looking forward to a good, authoritative Rust book, though I bet it’ll be thick.)

1. **The ownership system and borrow checker can confuse newcomers.** To achieve memory safety without GC, Rust has a sophisticate borrow and ownership system. This new thing often baffles newcomers.

1. **Rust compiler is very rigid.** I call Rust an anti-sloppy language. For anything Rust compiler cannot sufficiently infer, you have to specify your intention clearly, some of which you are even unaware of. This along with other learning barriers often makes one’s early Rust experience frustrating.

## The pros

Rust has many strengths. Some of them are unique.

#### Memory safety without GC

This is arguably Rust’s most significant achievement. For lower-level programming languages that allow direct memory manipulation, memory errors such as use-after-free or memory leak are expensive at runtime. Modern C++ provides better facilities to cope with that, but it requires good engineering discipline (read: programmers can still do unsafe things as before) and thus cannot reliably solve the problem at large scale in my view.

Indeed, Rust programmers can write unsafe code in an `unsafe` block, but (1) the programmer must do so intentionally, and (2) the `unsafe` blocks could be limited to a very small percentage of the code base and put under close scrutiny.

Garbage collector is the most common way to address memory safety. If you can live with GC, you have many choices. However, Rust’s ownership system does not only provides memory safety, but also data and resource safety. (See below.)

#### RAII on resources

[RAII](http://en.wikipedia.org/wiki/Resource_Acquisition_Is_Initialization) is a strange term but conveniently conveys the idea. According to Wikipedia, RAII works on stack-allocated objects. Rust’s ownership system works on heap-allocated objects, too. This makes automatic resource release such as memory deallocation, file or socket close predictable and guaranteed at compile-time.

Dynamic languages like Python or Ruby provide some related capabilities, but not as powerful as Rust IMO.

#### Concurrency without data races

Rust ensures data safety in concurrent programming, that is, only multiple readers or a single writer is allowed to access the data at any given point of time.

#### Algebraic data type

In addition to the common tuple and struct types, Rust also provides enum types (Rust’s sum types or variant types) and pattern matching. It’s amazing to see such an advanced type system on a systems programming language.

#### Composition over inheritance

Rust clearly prefers type composition over inheritance. I’m in this camp, so I consider it a strength. (Actually, Rust’s inheritance is not baked yet, not likely before 1.0.) Traits play a key role in Rust’s generics support.

## The cons (sort of)

#### Everything must be crystal clear

Rust is an anti-sloppy language. Everything must be crystal clear to the compiler, or it’ll keep rejecting you until there is no grey area. Usually, this is a good thing for code quality, but it can also be overkill for quick prototyping or one-off tasks.

The upside: you tend to write better, clearer code in Rust. When it’s internalised, the friction may get less or even go away. (I’m not there yet.)

#### GC is secondary

Rust provides basic GC: [`Rc`](http://doc.rust-lang.org/std/rc/struct.Rc.html), reference counting and [`Arc`](http://doc.rust-lang.org/std/sync/struct.Arc.html), atomically reference counting, without cyclic detection. However, they are not the default in the language, and you end up with using standard Rust memory management (stack, `&` and `Box`) more often. If memory is a non-issue to your application, you are still paying a price for Rust’s GC-free memory safety.

#### Expressiveness is not a goal

Expressiveness or elegance is not a goal of Rust. It’s certainly not bad in this regard, just not as wonderful as you may wish if you care it a lot.

#### Higher entry barrier

Rust is not a language you may quickly pick up and code professionally in a few  weeks. (OK, maybe you are smarter, but at least not in a few days.) Rust is perhaps smaller than C++, but definitely larger than many programming languages. Compared with other languages, it’s not among the most accessible.

On the other hand, the entry barrier is a filter. Rust code quality may be of higher quality partly due to this.

## Rust versus other languages

#### Dynamic languages

Dynamic languages, or scripting languages, are on the other end of the programming language spectrum. Compared to Rust, coding in dynamic languages is usually faster with less friction. I think dynamic languages work better than Rust for:

- quick prototyping or one-off tasks
- code not for production use, or the cost of runtime error is low
- one-man shop
- semi-automated job (e.g. log parsing/analysis, batch text processing)
- etc.

In those cases, the effort to get everything exactly right might not be worth it. On the contrary, I think Rust (post 1.0) will work better for:

- a modestly-sized or larger team
- code for long-running production use
- code with a longer lifetime which requires regular maintenance and/or refactoring
- code for which you would write more unit tests to safeguard

In short, when code quality matters. Dynamic languages give you the initial coding speed, but you pay the price later: writing more tests, breaking the pipeline, or even production outages. The Rust compiler forces you to get a lot of things right at compile-time, which is the least expensive place to identify and fix bugs.

#### Go

It can be debate-baiting to compare these two, but since I’ve spent some time on learning Go, I’d like to share my subjective opinions anyway. Compared with Rust, I like Go for the following:

- lightweight - the language is small (and simple is very powerful)
- gofmt - it removes much mental burden from developers when coding
- goroutine/channel
- instant compilation speed

Why I did not continue with Go:

- It’s too minimal. The type system and the language itself are not very extensible.
- Coding in Go is a bit dry to me. It reminds me of the days coding in Java: works well for enterprise development, mechanical, and... less fun. (It’s certainly a personal taste.)
- The backing by Google helps Go’s popularity, but also casts some concern on me. When community interests and company interests are not in line, the former may be de-prioritised. Surely every company weighs the company interests more. There’s nothing wrong with that. I’m just... a bit hesitated. (Many corporate-backed languages or frameworks may share a similar problem. Mozilla is, at least, not stock-price-driven.)

#### Nim

[Nim](http://nim-lang.org/index.html) (formerly Nimrod) is a very interesting language. It compiles to C, so the performance is pretty good. Its appearance resembles Python, a language I’ve always been enjoying coding with. It’s a GC language, but provides soft realtime support with more predictable GC behaviour. It has an interesting effect system. Overall, I like the language much.

Its biggest challenge is its ecosystem maturity. The language itself is well-designed and relatively stable, but nowadays, for a programming language to be successful, that’s far from enough. Documentations, standard libraries, package repositories, supporting frameworks, 3rd-party and community engagement... to refine everything to production quality is not a light task. Without some fulltime people behind, the last mile can be very arduous. Among the new programming languages, it has not gained a solid foothold yet.

That said, I wish the language successful. I’m still watching.

#### Others

There are other languages like [Julia](http://julialang.org/) and [D](http://dlang.org/). Julia is a dynamic language with good performance and smooth C calling. (If you prefer dynamic languages and REPL, you should take a look.) Julia has attracted a lot of attentions from numerical and scientific fields. Though it can be a general-purpose language, I believe the initial community of a language has a strong influence on its evolvement.

D is an attempt for a better C++, at least initially. Its 1.0 version was in 2007, so it’s really not a new language. It’s a good language, too, but a few factors may have attributed to its modest adoption after years: the Phobos/Tango split in its early days, the reliance on garbage collection for memory safety, and its initial proposition as a better C++.

## Why I think Rust has a pretty good chance

There are so many new programming languages these days. Why I think Rust can stand out? The following reasons are my take:

#### A true systems programming language

Being embeddable is not a small ask. It limits your choice to only a few, literally C and C++. (That may be why [Skylight](https://www.skylight.io/) picked Rust for Ruby extension development even when the bet was very risky.) Rust has done an amazing job on [removing](https://github.com/rust-lang/rust/pull/19654) its runtime overhead. This gives Rust a unique proposition.

#### Null-free

The null object/pointer (the so-called [billion-dollar mistake](http://en.wikipedia.org/wiki/Pointer_%28computer_programming%29#Null_pointer)) is a common source of runtime errors. Only very few programming languages are null-free, mainly functional programming languages. That‘s because it requires an advanced type system to eliminate null objects. Usually, algebraic data type and pattern matching are needed to deal with it at the language syntax level.

#### A low-level language with advanced high-level constructs

Being a bare-metal language down to kernel development ([in theory](https://github.com/pczarn/rustboot), at least), Rust also provides many higher-level features including algebraic data type, pattern matching, trait, type inference, etc.

#### Strong community and real use cases

Rust community is very friendly and active. (This is subjective, of course.) It also has a few solid use cases including Rust compiler, [Servo](https://github.com/servo/servo/), Skylight etc. during its language development.

#### No big mistakes so far

At times, an organization-backed programming language or framework could inadvertently get things screwed up. Luckily, the Rust core team have been doing a great job so far. Keep up the good work, Rust team!

## Rust for web development

Being a systems programming language, is Rust suitable for web development? This is also a question I’ve been seeking an answer for.

#### Libraries and frameworks

First and foremost, some basic HTTP libraries must be ready. ([Are we web yet](http://arewewebyet.com/) provides some information.) The pioneer [rust-http](https://github.com/chris-morgan/rust-http) is obsolete; its supposed successor, [Teepee](http://teepee.rs/), is almost dormant. Fortunately, [Hyper](https://github.com/hyperium/hyper) seems a good candidate. Being [adopted](http://blog.servo.org/2014/12/09/twis-14/) by Servo, the Rust’s symbiotic project, I read it as a blessed Rust HTTP library.

Rust standard library has not supported asynchronous I/O yet. An external library, [mio](https://github.com/carllerche/mio), can be used for nonblocking socket I/O. Green threads support was removed as part of the I/O simplification effort.

There are some Rust web frameworks under active development, such as [Iron](https://github.com/iron/iron/) and [nickel.rs](http://nickel.rs/). It may take a while before things settle down.

#### Is Rust for web?

Eventually, libraries and frameworks will be ready. The questions is, is Rust itself a programming language suitable for web development? Are Rust’s low-level feature and memory safety overkill?

I think in the end, it depends on your expectation of the project. Similar to my comparison between Rust and dynamic languages, for short-lived projects, Rust’s strictness might not be worth it. But if you expect the product to survive a while, say, half a year or longer, Rust can be a good choice.

#### Is Rust for web start-ups?

What about start-ups? They need quick prototyping and turnaround. This is a more complicated question, but my stance remains. If you expect your product to live longer, the programming language choice matters, and Rust is worth a serious look. From businessperson's perspective, a language that lets you prototype quickly gives major advantages, and one can always refactor or improve the system bottleneck later. The engineering reality is, refactoring often costs higher than it appears, and even you’ve revamped many parts of your system, the code written in your earliest days is still creeping in some corner. It just sticks over years.

Of course, Rust is still not 1.0, and things are not quite ready yet. If you can’t wait, either accept the early adopter's cost, or pick a more mature solution.

## You should try Rust!

Whether using Rust or not, you should give it a try (maybe after 1.0 alpha). Expect some early friction, but it shall go away quickly.
