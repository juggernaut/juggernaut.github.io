---
layout: post
title:  "Rust: str vs String"
date:   2017-10-12 18:40:23
categories: rust 
---
As a Rust newbie, I was confused by the different ways used to represent strings. The ["References and Borrowing"](https://doc.rust-lang.org/book/second-edition/ch04-02-references-and-borrowing.html)  chapter of the Rust book
uses three different types of string variables in the examples: `String`, `&String` and `&str`.

Let's start with the difference between `str` and `String`: `String` is a growable, heap-allocated data structure whereas
`str` is an immutable fixed-length string _somewhere_ in memory [^1].

### String
If you're a Java programmer, a Rust `String` is
semantically equivalent to `StringBuffer` (this was probably a factor in my confusion, as I'm so used to equating `String`
with immutable). As such, a `String` maintains a length _and_ a capacity whereas a `str` only has a `len()` method. As an example:

{% highlight rust %}
let mut s = String::from("Hello, Rust!");
println!("{}", s.capacity()); // prints 12
s.push_str("Here I come!");
println!("{}", s.len()); // prints 24

let s = "Hello, Rust!";
println!("{}", s.capacity()); // compile error: no method named `capacity` found for type `&str`
println!("{}", s.len()); // prints 12
{% endhighlight %}

### &str
You can only ever interact with `str` as a _borrowed_ type aka `&str`. This is called a _string slice_, an immutable view
of a string. This is the preferred way to pass strings around, as we shall see.

### &String
This is a reference to a `String`, also called a _borrowed_ type. This is nothing more than a pointer which you can pass
around without giving up ownership. Turns out a `&String` can be  _coerced_ to a `&str`:

{% highlight rust %}
fn main() {
    let s = String::from("Hello, Rust!");
    foo(&s);
}

fn foo(s: &str) {
    println!("{}", s);
}
{% endhighlight %}

In the above example, `foo()` can take either string slices or
borrowed `String`s, which is super convenient. As such, you almost never need to deal with `&String`s.
The only real use case I can think of is if you want to pass a mutable reference to a function that
needs to modify the string:

{% highlight rust %}
fn main() {
    let mut s = String::from("Hello, Rust!");
    foo(&mut s);
}

fn foo(s: &mut String) {
    s.push_str("appending foo..");
    println!("{}", s);
}
{% endhighlight %}

### Summary
Prefer `&str` as a function parameter or if you want a read-only view of a string; `String` when you want
to own and mutate a string.

[^1]: `str` data can live on the heap, stack or in the binary. [This](https://stackoverflow.com/a/24159933) excellent stackoverflow answer explains the scenarios for each.
