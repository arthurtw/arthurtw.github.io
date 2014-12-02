---
title: Rust borrow and lifetimes
layout: post
---

[Rust](http://www.rust-lang.org/) is a new programming language under [active development toward 1.0](http://blog.rust-lang.org/2014/09/15/Rust-1.0.html). I might write another blog about Rust and why I think it’s great, but today I’ll just focus on its borrow and lifetimes system, which has stumped many Rust newcomers including myself. This post assumes you have some basic understanding of Rust. If not yet, you may want to read its [Guide](http://doc.rust-lang.org/guide.html) and [Pointer Guide](http://doc.rust-lang.org/guide-pointers.html) first.

*EDIT (Dec-2): Added 8 scope charts for code samples.*

## Resource ownership and borrow

Rust achieves memory safety without GC by using a sophiscated borrow system. For any resource (stack memory, heap memory, file handle and so on), there is exactly one owner which takes care of its resource deallocation, if needed. You may create new bindings to refer to the resource using `&` or `&mut`, which is called a borrow or mutable borrow. The compiler ensures all owners and borrowers behave correctly.

### Copy and move

Before we jump into the borrow system, we should know how Rust handles copy and move. This [SO answer](http://stackoverflow.com/a/24253573) is a great read. Basically, in assignments and function calls:

1. If the value is copyable (only involving primitive types, no resources e.g. memory or file handle involved), the compiler defaults to copy.
1. Otherwise, the compiler moves (transfers) the ownership and invalidates the original binding.

In short, pod (plain old data) => copy, non-pod (linear types) => move.

Here are a few additional notes for your reference:

* Rust copy is like C. Every by-value use of a value is a byte copy (shallow memcpy copy) instead of a semantic copy or clone.
* To make a pod struct non-copyable, you may use a [NoCopy](http://doc.rust-lang.org/std/kinds/marker/struct.NoCopy.html) marker field, or implement [Drop](http://doc.rust-lang.org/std/ops/trait.Drop.html) trait.

After a move, the ownership is transferred to the next owner.

### Resource deallocation

Rust frees any resource as soon as its ownership disappears, that is, when:

1. the owner goes out of scope, or
1. the owning binding changes (thus the original binding becomes void).

### Owner’s and borrower’s privileges and restrictions

This section is based on the [Rust Guide](http://doc.rust-lang.org/guide.html), with a mention of copy and move in the privileges part.

An owner has some privileges. It may:

1. control resource deallocation,
1. lend the resource, immutably (multiple borrows) or mutably (exclusive), and
1. hand over the ownership (with a move).

An owner has some restrictions, too:

1. During a borrow, the owner may NOT (a) mutate the resource, or (b) mutably lend the resource.
1. During a *mutable* borrow, the owner may NOT (a) access the resource, or (b) lend the resource.

A borrower has some privileges, too. In addition to accessing or mutating the borrowed resource, a borrower may also share the borrow further:

1. A borrower may share (copy) an immutable borrow.
1. A *mutable* borrower may hand over (move) the mutable borrow. (Note that mutable reference is moved by default.)

### Code samples

Enough talk. Let’s see some code. (You may run Rust code at [play.rust-lang.org](http://play.rust-lang.org/).) In all examples below, we’ll use `struct Foo` which is *non-copyable* because it contains a boxed (heap-allocated) value. Using non-copyable resources makes the operations more restrictive, which is a good thing for learning.

For every code sample, we also provide a “scope chart” to illustrate the scopes of owner, borrowers etc. The curly braces in the header line match the curly braces in the code.

#### Owner *cannot* access resource during a mutable borrow

This code wouldn’t compile if we uncomment the last `println!` line:

{% highlight rust %}
struct Foo {
    f: Box<int>,
}

fn main() {
    let mut a = Foo { f: box 0 };
    // mutable borrow
    let x = &mut a;
    // error: cannot borrow `a.f` as immutable because `a` is also borrowed as mutable
    // println!("{}", a.f);
}
{% endhighlight %}

               { a x * }
       owner a   |_____|
    borrower x     |___| x = &mut a
    access a.f       |   error

It violates owner’s restriction \#2(a). If we put `let x = &mut a;` in a nested block, the borrow ends before the `println!` line and this would work:

{% highlight rust %}
fn main() {
    let mut a = Foo { f: box 0 };
    {
        // mutable borrow
        let x = &mut a;
        // mutable borrow ends here
    }
    println!("{}", a.f);
}
{% endhighlight %}

               { a { x } * }
       owner a   |_________|
    borrower x       |_|     x = &mut a
    access a.f           |   OK

#### Borrower *can* move the mutable borrow to a new borrower

This code shows the borrower’s privilege \#2: mutable borrower `x` can hand over (move) the mutable borrow to a new borrower `y`.

{% highlight rust %}
fn main() {
    let mut a = Foo { f: box 0 };
    // mutable borrow
    let x = &mut a;
    // move the mutable borrow to new borrower y
    let y = x;
    // error: use of moved value: `x.f`
    // println!("{}", x.f);
}
{% endhighlight %}

               { a x y * }
       owner a   |_______|
    borrower x     |_|     x = &mut a
    borrower y       |___| y = x
    access x.f         |   error

After the move, the original borrower `x` can no longer access the borrowed resource.

## Borrow scope

Things start getting interesting if we pass the references (`&` and `&mut`) around, and that’s where many Rust newcomers’ confusions begin.

### Lifetime

In the whole borrow story, it’s really important to know where a borrower’s borrow starts and ends. In the [Lifetimes Guide](http://doc.rust-lang.org/guide-lifetimes.html), it’s called a *lifetime*:

> A lifetime is a static approximation of the span of execution during which the pointer is valid: it always corresponds to some expression or block within the program.

However, I would like to use the term *borrow scope* to describe the scope where the borrow is effective. Note that it actually differs from the lifetime definition above. (I first saw that term in a Rust [RFC discussion](https://github.com/rust-lang/rfcs/pull/431), though my definition may differ.) I will give reasons why I avoid using lifetimes later. For now, let’s just put lifetimes aside.

### & = borrow

A few things about borrow:

Firstly, just remember `&` = borrow and `&mut` = mutable borrow. Wherever you see an `&`, there *is* a borrow.

Secondly, when an `&` shows up in any struct (in its field) or function/closure (in its return type or captured references), the struct/function/closure *is* a borrower, and all borrow rules apply.

Thirdly, for every borrow, there is exactly *an* owner and a *single* or *multiple* borrowers.

### Borrow scope extension

A few things about borrow scope:

Firstly, a borrow scope:

* *is* the scope where the borrow is effective, and
* *is not* necessarily the lexical scope of the initial borrower, because the borrower can extend the borrow scope (see below).

Secondly, a borrower can extend the borrow scope through a copy (immutable borrow) or move (mutable borrow) that takes place in assignments or function calls. The receiver (can be a new binding, struct, function or closure) then becomes a *new borrower*.

Thirdly, a borrow scope is the *union* of all borrowers’ scopes, and the borrowed resource must be valid through the whole borrow scope.

### Borrow formula

From the last point, we have this **borrow formula**:

> resource scope >= borrow scope = union of all borrowers’ scopes

### Code sample

Let’s see some example of borrow scope extension. The `struct Foo` is the same as before:

{% highlight rust %}
fn main() {
    let mut a = Foo { f: box 0 };
    let y: &Foo;
    if false {
        // borrow
        let x = &a;
        // share the borrow with new borrower y, hence extend the borrow scope
        y = x;
    }
    // error: cannot assign to `a.f` because it is borrowed
    // a.f = box 1;
}
{% endhighlight %}

                 { a { x y } * }
      resource a   |___________|
      borrower x       |___|     x = &a
      borrower y         |_____| y = x
    borrow scope       |=======|
      mutate a.f             |   error

Even though the borrow happens inside the `if` block and the borrower `x` goes out of scope after the `if` block, it has extended the borrow scope through an assignment `y = x;`, so there are two borrowers: `x` and `y`. According to the borrow formula, the borrow scope is the union of borrower `x` and borrower `y`’s scopes, which ranges from the first borrow `let x = &a;` through the end of the `main` block. (Note that the binding `y` is not a borrower before the `y = x;` line.)

You might have noticed that the `if` block will never get executed since the condition is always `false`, but the compiler still forbids the resource owner `a` to access its resource. This is because all the borrow checking happens at **compile-time**, nothing to do with the program runtime execution.

## Borrowing multiple resources

So far, we only focus on the borrow of a single resource. Can a borrower borrow multiple resources? Of course! For example, a function may take two references and returning one of them depending on certain criteria, e.g. which one has a larger value in its field:

{% highlight rust %}
fn max(x: &Foo, y: &Foo) -> &Foo
{% endhighlight %}

The `max` function returns an `&` pointer, hence it is a borrower. The return result can be from either input parameter, so it is borrowing two resources.

### Named borrow scope

When there are multiple `&` pointers as inputs, we need to specify their relationship using *named lifetimes* as defined in the [Lifetimes Guide](http://doc.rust-lang.org/guide-lifetimes.html#named-lifetimes). But for now, let’s just call them *named borrow scopes*.

The above code wouldn’t be accepted by the compiler without specifying the relationship between borrowers, i.e. which borrowers are *grouped* in which borrow scope. The following implementation is valid:

{% highlight rust %}
fn max<'a>(x: &'a Foo, y: &'a Foo) -> &'a Foo {
    if x.f > y.f { x } else { y }
}
{% endhighlight %}

    (All resources and borrowers are grouped in borrow scope 'a.)
                      max( {   } ) 
        resource *x <-------------->
        resource *y <-------------->
    borrow scope 'a <==============>
         borrower x        |___|
         borrower y        |___|
       return value          |___|   pass to the caller

In this function, we have one borrow scope `'a` and three borrowers: the two input parameters, and the function return result. The aforementioned borrow formula still applies, but now *every* borrowed resource must satisfy the formula. See the example below.

### Code sample

In the following code, let’s use the above `max` function to pick up the bigger `Foo` between `a` and `b`:

{% highlight rust %}
fn main() {
    let a = Foo { f: box 1 };
    let y: &Foo;
    if false {
        let b = Foo { f: box 0 };
        let x = max(&a, &b);
        // error: `b` does not live long enough
        // y = x;
    }
}
{% endhighlight %}

                  { a { b x (  ) y } }
       resource a   |________________| pass
       resource b       |__________|   fail
     borrow scope         |==========|
    temp borrower            |_|       &a
    temp borrower            |_|       &b
       borrower x         |________|   x = max(&a, &b)
       borrower y                |___| y = x

Until `let x = max(&a, &b);`, things are fine because `&a` and `&b` are temporary references which are valid only in the expression, and the third borrower `x` borrows the two resources (either `a` or `b` but to the borrow checker, it borrows both) till the end of the `if` block, so the borrow scope is from `let x = max(&a, &b);` to the end of the `if` block. Both resources `a` and `b` are valid through the whole borrow scope, hence satisfying the borrow formula.

Now if we uncomment the last assignment `y = x;`, `y` becomes the fourth borrower, and the borrow scope is extended to the end of the `main` block, causing resource `b` to fail the test of the formula.

## Struct as a borrower

In addition to functions and closures, a *struct* can also borrow multiple resources by storing multiple references in its field(s). We’ll see some examples below and how the borrow formula applies. Let’s use this `Link` struct to store a reference (an immutable borrow):

{% highlight rust %}
struct Link<'a> {
    link: &'a Foo,
}
{% endhighlight %}

### Struct to borrow multiple resources

Even with only one field, struct `Link` can borrow multiple resources:

{% highlight rust %}
fn main() {
    let a = Foo { f: box 0 };
    let mut x = Link { link: &a };
    if false {
        let b = Foo { f: box 1 };
        // error: `b` does not live long enough
        // x.link = &b;
    }
}
{% endhighlight %}

                 { a x { b * } }
      resource a   |___________| pass
      resource b         |___|   fail
    borrow scope     |=========|
      borrower x     |_________| x.link = &a
      borrower x           |___| x.link = &b

In the above example, borrower `x` is borrowing resource from owner `a`, and the borrow scope is till the end of the `main` block. So far so good. If we uncomment the last assignment `x.link = &b;`, `x` is also trying to borrow resource from owner `b`, which would make resource `b` to fail the test of the borrow formula.

### Function to extend borrow scope without a return value

A function without a return value can also extend the borrow scope through its input parameters. For example, this function `store_foo` takes a mutable reference of `Link`, and stores a reference (immutable borrow) of `Foo` in it:

{% highlight rust %}
fn store_foo<'a>(x: &mut Link<'a>, y: &'a Foo) {
    x.link = y;
}
{% endhighlight %}

In the following code, the resource owned by `a` is the borrowed resource; the `Link` struct mutably referenced by `x` is the borrower (i.e. `*x` is the borrower); the borrow scope is till the end of the `main` block.

{% highlight rust %}
fn main() {
    let a = Foo { f: box 0 };
    let x = &mut Link { link: &a };
    if false {
        let b = Foo { f: box 1 };
        // store_foo(x, &b);
    }
}
{% endhighlight %}

                 { a x { b * } }
      resource a   |___________| pass
      resource b         |___|   fail
    borrow scope     |=========|
     borrower *x     |_________| x.link = &a
     borrower *x           |___| x.link = &b

If we uncomment the last function call `store_foo(x, &b);`, the function will try to store `&b` to `x.link`, making resource `b` another borrowed resource and failing the test of the borrow formula, since resource `b`’s scope does not cover the whole borrow scope.

### Multiple borrow scopes

It is possible to have multiple named borrow scopes in a function. For example:

{% highlight rust %}
fn superstore_foo<'a, 'b>(x: &mut Link<'a>, y: &'a Foo,
                          x2: &mut Link<'b>, y2: &'b Foo) {
    x.link = y;
    x2.link = y2;
}
{% endhighlight %}

In this (probably not very useful) function, two disjointed borrow scopes are involved. Each borrow scope would have its own borrow formula to satisfy.

## Why lifetime is confusing

Lastly, I want to explain why I think the term *lifetime* used by Rust’s borrow system is confusing (and I thus avoid using it in this blog post).

When we talk about borrow, there are three different kinds of “lifetime” involved:

* A: the lifetime of the resource owner (or the owned/borrowed resource)
* B: the “lifetime” of the whole borrow, i.e. from the first borrow to the last return
* C: the lifetime of an individual borrower or borrowed pointer

When one says “lifetime”, it can refer to any of the above. If multiple resources and borrowers are involved, things get even more confusing. For example, what does a “named lifetime” refer to in the declaration of a function or struct? Does it mean A, B or C?

In our previous `max` function:

{% highlight rust %}
fn max<'a>(x: &'a Foo, y: &'a Foo) -> &'a Foo {
    if x.f > y.f { x } else { y }
}
{% endhighlight %}

What does lifetime `'a` mean here? It shouldn’t be A, because two resources are involved and they have different lifetimes. It cannot be C, because there are three borrowers: `x`, `y` and the function return value, and they all have different lifetimes, too. Does it mean B? Probably. But the whole borrow scope is not a concrete object, how can it have a “lifetime”? Calling it lifetime is just confusing.

Some may say it means the minimal lifetime requirements to the borrowed resources’ lifetimes. That makes sense in some way, but how can we call the minimal lifetime requirements “a lifetime”?

The ownership/borrow concept itself is already complicated. The confusion that the term “lifetime” brings makes learning the concept even more baffling, I would say.

P.S. Using the A, B and C defined above, the borrow formula becomes:

> A >= B = C<sub>1</sub> U C<sub>2</sub> U ... U C<sub>n</sub>

## Learning Rust is worth your time!

Although the borrow and ownership thing may take you a while to grok, it’s an interesting learn. Rust tries to achieve memory safety without GC, and it’s doing pretty well so far. Some people say learning Haskell changes the way you program. I think learning Rust is worth your time, too.

Hope this blog post provides a little help.
