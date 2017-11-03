---
layout: post
title:  "Rust: Builder pattern by example"
date:   2017-11-02 18:40:23
categories: rust 
---
To begin learning Rust in earnest, I recently started writing a client library for Twilio [^1]. As a library writer, (quoting
Alan Kay) you want to make simple things simple but complex things possible. For example, to make an [outbound call](https://www.twilio.com/docs/api/voice/making-calls)
with Twilio, there are three required parameters but a whole lot of optional parameters that aren't used very often.

## Need for the builder pattern

All outbound call parameters can be representing as a struct that looks like this (I've omitted most of the optional parameters for brevity):

{% highlight rust %}
struct OutboundCall<'a> {
    from: &'a str,
    to: &'a str,
    url: &'a str,
    fallback_url: Option<&'a str>,
    status_callback: Option<&'a str>,
    ...
}
{% endhighlight %}

This struct has a large number of fields. As such, forcing the application programmer to populate the entire struct
consisting of mostly `None` values is unergonomic. Rust does not (yet) have default values for struct fields or default
function arguments, although they have been [proposed](https://github.com/rust-lang/rfcs/pull/257) [^2]. A nice way to solve this is to use
[the builder](https://en.wikipedia.org/wiki/Builder_pattern) pattern:

{% highlight rust %}
let call = OutboundCallBuilder::new("tom", "jerry", "http://www.example.com")
    .with_fallback_url("http://fallback.com")
    .with_status_callback("http://status.com")
    ...
    .build();
{% endhighlight %}

Here, we only accept the mandatory fields in the constructor, and provide methods to optionally fill in the rest of
the fields.

As a Rust newbie, there were a couple of subtleties involved in implementing the builder pattern. Let's go over them.

## A first pass

My first stab at a builder implementation looked like this:

{% highlight rust %}
struct OutboundCallBuilder<'a> {
    from: &'a str,
    to: &'a str,
    url: &'a str,
    fallback_url: Option<&'a str>,
}

impl<'a> OutboundCallBuilder<'a> {
    fn new(from: &'a str, to: &'a str, url: &'a str) -> OutboundCallBuilder<'a> {
        OutboundCallBuilder {
            from,
            to,
            url,
            fallback_url: None,
        }
    }

    fn with_fallback_url(mut self, fallback_url: &'a str) -> Self {
        self.fallback_url = Some(fallback_url);
        self
    }

    fn build(self) -> OutboundCall<'a> {
        OutboundCall {
            from: self.from,
            to: self.to,
            url: self.url,
            fallback_url: self.fallback_url,
        }
    }
}
{% endhighlight %}

Here, each builder method returns `self` to allow for chaining, so we can write:
{% highlight rust %}
let call = OutboundCallBuilder::new("tom", "jerry", "http://www.example.com")
    .with_fallback_url("http://fallback.com")
    .build();
{% endhighlight %}

This works! But what if we had more complex builder logic, like:
{% highlight rust %}
let builder = OutboundCallBuilder::new("tom", "jerry", "http://www.example.com");
if (need_fallback) {
    builder.with_fallback_url("http://www.fallback.com");
}
let call = builder.build();
{% endhighlight %}

This fails with a compile error:
<pre>
error[E0382]: use of moved value: `builder`
  --> src/main.rs:53:16
   |
52 |     builder.with_fallback_url("http://www.fallback.com");
   |     ------- value moved here
53 |     let call = builder.build();
   |                ^^^^^^^ value used here after move
</pre>

Aha, this is because each builder method is taking ownership of `self` and the return statement is relinquishing ownership 
back to the caller. There's only one problem - the return value is not assigned to anything, and hence there is no owner! We can make the
compiler happy by re-assigning to builder each time a builder method is called,
so that the owner is not dropped prematurely:
<pre>
let mut builder = OutboundCallBuilder::new("tom", "jerry", "http://www.example.com");
if (need_fallback) {
    <b>builder</b> = builder.with_fallback_url("http://www.fallback.com");
}
let call = builder.build();
</pre>
This is pretty inelegant IMO and again places an unnecessary burden on the library consumer. 

## Getting it right

To avoid having the builder methods take ownership of `self`, we can instead take and return a mutable reference
to `self`:

{% highlight rust %}
  fn with_fallback_url(&mut self, fallback_url: &'a str) -> &mut Self {
        self.fallback_url = Some(fallback_url);
        self
    }
{% endhighlight %}

This solves the multi-statement builder problem, but compilation now fails on the one-liner instead:
<pre>
   |
72 |       let call = OutboundCallBuilder::new("tom", "jerry", "http://www.example.com")
   |  ________________^
73 | |         .with_fallback_url("http://fallback.com")
   | |_________________________________________________^ cannot move out of borrowed content
</pre>

Duh, of course! My `build()` method is still _consuming_ (taking ownership) of `self`, but we're passing
it a borrowed reference. Let's fix that by consuming `self` by reference in `build()`:
{% highlight rust %}
fn build(&self) -> OutboundCall<'a> {
    ...
}
{% endhighlight %}

This allows for both multi-line and one-liners! Note, however, that I could do this only because my `struct` doesn't require _owned_
data. If it did, then `build()` would be required to take ownership of `self` - in this case, there would
be no option but to sacrifice some usability.

[^1]: **Disclaimer**: I work at Twilio, but this will be an unofficial library. Also to reiterate, all opinions expressed on this blog are my own and do not reflect the views of my employer.
[^2]: An alternate way is to derive the `Default` trait for the struct, as this [stackoverflow answer](https://stackoverflow.com/a/19653453) indicates. However, it still requires the user to pass `..Default::default()` which is IMO ugly.
