---
layout: post
title:  "Rust: Builder pattern by example"
date:   2017-11-02 18:40:23
categories: rust 
---
To begin learning Rust in earnest, I recently started writing a client library for Twilio. As a library writer, (quoting
Alan Kay) you want to make simple things simple but complex things possible. For example, to make an [outbound call](https://www.twilio.com/docs/api/voice/making-calls)
with Twilio, there are three required parameters but a whole lot of optional parameters that aren't used very often.

Rust does not have default function arguments, so for populating a `struct` with a large number of optional fields,
the builder pattern is preferred. One such example is making an outbound call via Twilio.

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

This struct has a large number of fields, and as such forcing the application programmer to populate the entire struct
consisting of mostly `None` values is unergonomic. Rust does not (yet) have default values for struct fields or default
function arguments, although they have been proposed. #Note about default trait# A nice way to solve this is to use
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

But what if we had more complex builder logic, like:
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

We can fix this by re-assigning to builder each time a builder method is called:
<pre>
let mut builder = OutboundCallBuilder::new("tom", "jerry", "http://www.example.com");
if (need_fallback) {
    <b>builder</b> = builder.with_fallback_url("http://www.fallback.com");
}
let call = builder.build();
</pre>
