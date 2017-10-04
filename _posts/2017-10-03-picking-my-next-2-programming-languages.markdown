---
layout: post
title:  "Picking my next two programming languages"
date:   2017-10-03 18:40:23
categories: rust kotlin
---

(**TL;DR** [Rust](https://www.rust-lang.org) and [Kotlin](https://kotlinlang.org/), read on to find out why...)

I believe a competent backend developer should be proficient in at least one language in each of three categories:
1. **Systems programming**: Used for writing lower-level software ranging from operating system kernels to network programming. 
2. **Scripting**: Used for writing test automation, one-off scripts and various system administration tasks.
3. **Application programming**: Used for writing line-of-business applications like accounting software and web application backends.

Note that the lines between these categories are often blurred and a programming language can be a good fit for writing
software in multiple categories. Nevertheless, the classification serves as a good rule of thumb.

I'm currently comfortable writing Java, Python and C; probably a little _too_ comfortable. I believe that growth happens when you step out of 
your comfort zone. In that spirit, I've been pondering picking up a new programming language or two to
broaden my horizons.

Turns out this is an exciting time for programming languages; I have several new ones to choose from!  This post details my (rather unscientific)
process of how I chose what language(s) to learn.

## Systems programming

C has been and still is the undisputed king of systems programming. Most major operating systems and databases are written
in C. The first programming language I learned was C, and it quickly became (by default :)) my favorite programming language.
Although it has served us well over the decades, C isn't without faults. It is not [memory safe](https://en.wikipedia.org/wiki/Memory_safety),
which has resulted in thousands of security vulnerabilities and bugs. Even code written by experienced programmers, in well-maintained 
open-source software like [nginx](https://nvd.nist.gov/vuln/detail/CVE-2009-2629)
is vulnerable to common buffer under/overflows. As a programmer of more modest means, I'm looking for a language that will allow me to
write memory safe code by default.

### The candidates

Without spending time doing too much analysis, I chose Go and Rust simply because some of the most popular
open source projects are written using them. [Docker](https://www.docker.com/) is written in Go and [Servo](https://servo.org/) is Rust's flagship project.

I worked through tutorials for both languages, and wrote up a pros and cons list for each.

### Golang

**Pros:**
* Simple, easy to learn
* Wide adoption, backed by Google
* Garbage collected
* Built-in concurrency constructs ([goroutines](https://gobyexample.com/goroutines))

**Cons:**
* Allows nulls
* No generics
* No standard package manager

### Rust
**Pros:**
* Non-nullability by default
* Advanced memory safety features ([borrow-checking](https://doc.rust-lang.org/1.8.0/book/references-and-borrowing.html))
* Functional programming constructs built-ins
* Backed by Mozilla
* Standard package manager ([Cargo](http://doc.crates.io/))

**Cons:**
* Not as widely adopted
* Learning curve is steeper due to new concepts like variable lifetimes
* Concurrency constructs like async/await are not built-in

### And the winner is..

**Rust**. As I wrote the lists, it became clear to me that Go doesn't really offer that much over my primary language, Java.
In comparison, Rust has some really novel concepts, allowing you to write memory-safe code without a garbage collector. However,
the biggest factor that swayed me in favor of Rust was non-nullability by default. Tony Hoare called null references
the "billion dollar mistake" and the creators of Rust seems to have taken note.

## Scripting

Python hits the sweet spot for me here. I use it for all of my automation tasks, and I just don't feel the need to reach for
something new here.

## Application programming

Java is the ten-thousand pound gorilla in this space. It gets a lot of hate, but I actually really like Java. I do admit, though,
that it can get too verbose at times. Even though features like type inference and Java 8 functional constructs have alleviated a lot of the verbosity, the baggage still
remains. 

I'm looking for less verbose language that allows me to express business logic without getting in the way.
However, I have no intent of abandoning the Java ecosystem. The JVM is simply the best runtime out there.
This limits my search to languages that primarily target the JVM.

### The candidates

The obvious candidates are Clojure, Scala and Kotlin. I prefer a statically typed language for large application codebases, so that leaves us with Scala and Kotlin.
I created the pros and cons lists as before:

### Scala

**Pros:**
* Wide adoption by companies like Twitter, LinkedIn etc.
* Much stronger type system than Java.
* Can write purely functional code, purely imperative or a mixture of both. 

**Cons:**
* Massive feature footprint 
* Steep learning curve
* No built-in async-await, although there is a [pending proposal](http://docs.scala-lang.org/sips/pending/async.html)

### Kotlin

**Pros**:
* Lightweight
* Shorter learning curve coming from Java
* Built-in [coroutines](https://kotlinlang.org/docs/reference/coroutines.html)
* Excellent IntelliJ support since it's made by the same company

**Cons**:
* New kid on the block, not much adoption/production usage.
* Coroutines are still experimental as of this writing.

### And the winner is...
**Kotlin**. The main factor in favor of Kotlin was the tighter feature set and lightweight-ness, if that's a word.
I also admit to being a little bit biased here - in my anecdotal experience, a lot of Scala programmers I've met have been ivory-tower programmers, more
interested in debating Category Theory than getting shit done. In comparison, Kotlin just feels more pragmatic.

## Summary

Learning a new programming language in addition to its ecosystem takes months to years. I realize I'm being ambitious by choosing not one but two languages
to learn. The hope is that this endeavor will help me become a better programmer, and who knows I may even end up writing code professionally in one of these
languages in the future!
