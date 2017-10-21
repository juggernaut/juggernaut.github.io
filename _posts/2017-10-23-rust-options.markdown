---
layout: post
title:  "Rust: Using Options by example"
date:   2017-10-23 18:40:23
categories: rust 
---
Rust avoids the [billion dollar mistake](https://en.wikipedia.org/wiki/Tony_Hoare#Apologies_and_retractions) of including 
`null`s in the language. Instead, we can represent a value that might or might not exist with the `Option` type.
 This is similar to Java 8 `Optional` or Haskell’s `Maybe`. There is [plenty](http://www.nickknowlson.com/blog/2013/04/16/why-maybe-is-better-than-null/)
 of material out there detailing why an Option type is better than null, so I won’t go too much into that.

In Rust, `Option<T>` is an _enum_ that can either be `None` (no value present) or `Some(x)` (some value present).
 As a newbie, I like to learn through examples, so let’s dive into one.

## Example
Consider a struct that represents a person’s full name. The first and last names are mandatory, whereas the middle name
 may not be specified. We can represent such a struct like this [^1]:

{% highlight rust %}
struct FullName {
    first: String,
    middle: Option<String>,
    last: String,
}
{% endhighlight %}

Let’s create full names with/without a middle name:

{% highlight rust %}
let alice = FullName {
    first: String::from("Alice"),
    middle: Some(String::from("Bob")), // Alice has a middle name
    last: String::from("Smith")
};

let jon = FullName {
    first: String::from("Jon"),
    middle: None, // Jon has no middle name
    last: String::from("Snow")
};
{% endhighlight %}

Suppose we want to print the middle name if it is present. Let’s start with the simplest method, `unwrap()`:

{% highlight rust %}
println!("Alice's middle name is {}", alice.middle.unwrap()); // prints Bob
It works! Let’s try it with Jon:
{% endhighlight %}

{% highlight rust %}
println!("Jon's middle name is {}", jon.middle.unwrap()); // panics
{% endhighlight %}

So, `unwrap()` panics and exits the program when the Option is empty i.e None. This is less than ideal.

## Pattern matching
Since `Option` is actually just an `enum`, we can use pattern matching to print the middle name if it is present, or a default message if it is not.

{% highlight rust %}
println!("Jon's middle name is {}",
    match jon.middle {
        None => "No middle name!",
        Some(x) => x,
    }
);
{% endhighlight %}

This fails compilation with the error:

```
error[E0308]: match arms have incompatible types
  --> src/main.rs:28:9
   |
28 | /         match jon.middle {
29 | |             None => "No middle name!",
30 | |             Some(x) => x,
31 | |         }
   | |_________^ expected &str, found struct `std::string::String`
```

Recall in my [earlier]({{ site.baseurl }}{% post_url 2017-10-12-rust-str-vs-String %}) post, that a string _literal_ is actually 
a string _slice_. So our `None` arm is returning a string slice,
but our `Some` arm is returning the _owned_ `String` struct member. Turns out we can conveniently use `ref` in a pattern match
to borrow a reference. Again, recalling that `&String` can be coerced to `&str`, this solves our type mismatch problem.

{% highlight rust %}
println!("Jon's middle name is {}",
    match jon.middle {
        None => "No middle name!",
        Some(ref x) => x, // x is now a string slice
    }
);
{% endhighlight %}

This works!

## Option methods
Pattern matching is nice, but `Option` also provides several useful methods. We can achieve what we did in the previous section with `unwrap_or()`:

{% highlight rust %}
println!("Alice's middle name is {}",
    alice.middle.unwrap_or("No middle name!".to_owned()));
{% endhighlight %}

### map

`map()` is used to transform `Option` values. For example, we could use `map()` to print only the middle initial:

{% highlight rust %}
println!(
    "Alice's full name is {} {} {}",
    alice.first,
    alice.middle.map(|m| &m[0..1]).unwrap_or(""), // Extract first letter of middle name if it exists
    alice.last
);
{% endhighlight %}

However, this fails to compile with the very clear error:

```
42 | |         alice.middle.map(|m| &m[0..1]).unwrap_or(""),
   | |                               -     ^ `m` dropped here while still borrowed
   | |                               |
   | |                               borrow occurs here
```

Ah, so `map()` _consumes_ the contained value, which means the value does not live past the scope of the `map()` call!
Luckily, the `as_ref()` method of `Option` allows us to borrow a reference to the contained value:

{% highlight rust %}
println!(
    "Alice's full name is {} {} {}",
    alice.first,
    alice.middle.as_ref().map(|m| &m[0..1]).unwrap_or(""), // as_ref() converts Option<String> to Option<&String>
    alice.last
);
{% endhighlight %}

Instead of first using `map()` to transform to another `Option` and then unwrapping it, we can use the convenience
method `map_or()` which allows us to do this in one call:

{% highlight rust %}
alice.middle.as_ref().map_or("", |m| &m[0..1])
{% endhighlight %}

### and_then

`and_then()` is another method that allows you to compose Options (equivalent to flatmap in other languages).
Suppose we have a function that returns a nickname for a real name, if it knows one. For example, here is such a
function (admittedly, one that has a very limited worldview):

{% highlight rust %}
fn get_alias(name: &str) -> Option<&str> {
    match name {
        "Bob" => Some("The Builder"),
        "Elvis" => Some("The King"),
        _ => None,
    }
}
{% endhighlight %}

Now, to figure out a person’s middle name’s nickname (slightly nonsensical, but bear with me here), we could do:

{% highlight rust %}
let optional_nickname = alice.middle.as_ref().and_then(|m| get_nickname(&m));
println!("Alice's middle name's nickname is {}",
    optional_nickname.unwrap_or("(none found)")); // prints "The Builder"
{% endhighlight %}

In essence, `and_then()` takes a closure that returns another `Option`. If the `Option` on which `and_then()` is called is present,
then the closure is called with the present value and the returned `Option` becomes the final result. Otherwise, the final result
remains `None`. As such, in the case of `jon`, since the middle name is `None`, the `get_nickname()` function will not be called at all,
and the above will print “(none found)”.

## Summary
Rust provides a robust way to deal with optional values. The `Option` enum has several other useful methods I didn't cover. You can
find the full reference [here](https://doc.rust-lang.org/std/option/enum.Option.html).

[^1]: Experienced Rust programmers would probably have the struct members be string slices, but that would require use of lifetimes, which is outside the scope of this post.
