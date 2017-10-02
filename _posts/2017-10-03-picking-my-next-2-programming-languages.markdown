---
layout: post
title:  "Picking my next two programming languages"
date:   2017-09-27 18:40:23
categories: "rust,kotlin"
---

I believe a competent backend developer should be proficient in at least one language in each of three categories:
1. Systems programming: Used for writing lower-level software ranging from operating system kernels to network programming. 
2. Scripting: Used for writing test automation, one-off scripts and various system administration tasks.
3. Application programming: Used for writing line-of-business applications like accounting software, office software etc.

Note that the lines between these categories are often blurred and a programming language can be a good fit for writing
software in multiple categories. Nevertheless, the classification serves as a good rule of thumb.

This is an exciting time for programming languages with several new languages gaining traction. In this blog post, I'm
going to talk about my process for picking my next two programming languages - FIXME

## Systems programming

C has been and still is the undisputed king of systems programming. Most major operating systems and databases are written
in C. The first programming language I learned was C, and it quickly became (by default :)) my favorite programming language.
Although it has served us well over the decades, C isn't without faults. It is not [memory safe](https://en.wikipedia.org/wiki/Memory_safety),
which has opened up thousands of security vulnerabilities and bugs. Even code written by experienced programmers like [nginx](https://nvd.nist.gov/vuln/detail/CVE-2009-2629)
is vulnerable to common buffer under/overflows. As a programmer of more modest means, I'm looking for a language that will allow me to
write memory safe code by default.

### The candidates

Without spending time doing too much analysis, I chose Go and Rust ... simply because of the traction they have (FIXME) in popular
open source projects. Docker (Go) and Firefox (Rust) (FiXME)

Go:
1. Allows nulls
2. No generics
3. Concurrency built-in
4. GC'ed

Rust:
1. No nulls
2. Memory safe (borrow-checker)
3. Functional programming constructs built-ins

A big reason I was swayed in Rust's favor was no nulls by default.

## Scripting

Staying with python

## Application programming

Java is the 10 thousand pound gorilla in this space. Java has been my primary programming language for the last few years. Even though
features like (limited) type inference and Java 8 functional constructs have alleviated a lot of the verbosity, the baggage still
remains. I'm looking for a more lightweight language that allows me to express business logic without getting in the way.
I have no intent of abandoning the Java ecosystem. The JVM is simply the best runtime out there. This limits our search to languages
that have (FIXME) the JVM as their primary target.

### The candidates

Clojure - eliminated due to dynamically typed. That leaves us with Scala and Kotlin. Scala is a language with a massive footprint. I also admit to being
a little bit biased here - in my anecdotal experience, a lot of Scala programmers I've met have been ivory-tower programmers, more
interested in debating Category Theory than getting shit done.

Kotlin is lightweight, has coroutines built into the language and comes with excellent support in my IDE of choice (IntelliJ).
