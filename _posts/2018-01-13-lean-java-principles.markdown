---
layout: post
title:  "The Principles of Lean Java"
date:   2018-01-13 20:40:00
categories: software 
---

I began my software development career writing Python. It wasn't until a couple of years later that I began to write Java in earnest. Like many Python/Ruby programmers, my opinion
of Java at the time was shaped by various blog posts and HackerNews comments bashing Java's verbosity and heaviness. So when I started doing Java development
around 2012, I was pleasantly suprised - modern Java was easy to use, lightweight, and dare I say even fun to write. The subsequent years have seen this trend
accelerate, with new Java releases that help reduce boilerplate and the popularity of high-quality, lightweight libraries and frameworks. 

This is not to say that Java's reputation is entirely unfounded. I have also had to deal with legacy frameworks, XML hell and over-abstracted code. Thankfully,
these things can be almost entirely avoided. This blog post is an attempt to distill my experience into a few general (opinionated) principles that I've used to write lean[^1],
modern Java.

### 1. Prefer annotations/code over XML

Configuration should be easy for humans to read and edit. XML's signal-to-noise ratio is just too [low](https://blog.codinghorror.com/xml-the-angle-bracket-tax/).
Thankfully, annotations/straight-up-code have already won this battle in the Java world. So much so that, Spring, one of the worst proliferators of XML as configuration, stopped
supporting [XML-based request mapping](https://jira.spring.io/browse/SPR-5757)

### 2. Prefer functional code over imperative code

Java 8 introduced lambdas and streams that help reduce a lot of boilerplate. Still, I've seen code that looks like this (in Java 8)

{% highlight java %}
List<Subscription> activeSubscriptions = new ArrayList<>();
for (Subscription s: subscriptions) {
    if (s.isActive()) {
        activeSubscriptions.add(s);
    }
}
{% endhighlight %}

vs the much more succint version using streams:

{% highlight java %}
List<Subscription> activeSubscriptions = subscriptions.stream()
        .filter(Subscription::isActive)
        .collect(toList());
{% endhighlight %}

### 3. Embrace modern deployment tools

The traditional way of deploying Java web applications was to package up your code and deploy it within a separate application server like Tomcat. This meant that it was hard
to set up a local development environment that matched the production setup. With the rise of container technology, the application and its dependencies are packaged up into a single, easy-to-deploy artifact. This is obviously at odds with the application server model.

So, I agree that [Java application servers are dead](https://www.slideshare.net/ewolff/java-application-servers-are-dead). Modern Java web applications use embedded servlet containers like [Jetty](http://www.eclipse.org/jetty/) and package up all dependencies into an "uber-jar". I'm a big fan of the [Dropwizard](http://www.dropwizard.io/) framework which adopts this approach.

Expect this trend to accelerate with the introduction of a [module sytem](https://www.oracle.com/corporate/features/understanding-java-9-modules.html) in Java 9.

### 4. Eschew heavyweight standards

It seems like a portion of the Java community love adhering to every standard put out by the JCP. In my opinion,
 some of these like [JPA](http://www.oracle.com/technetwork/java/javaee/tech/persistence-jsp-140049.html) are too heavyweight for most applications.
 I dislike ORMs in general, let alone an ORM API as a standard. Instead, I recommend using a thin wrapper over SQL like [JDBI](http://jdbi.org/) or [jOOQ](https://www.jooq.org/) if you don't like writing raw SQL.

This doesn't mean I'm for eschewing every standard out there - in fact, Dropwizard, the framework I mentioned above, is built on top of a JAX-RS implementation. Adopt standards
where they make sense.


### 5. Embrace non-Java tools

This might seem counter-intuitive in an article about Java, but I've noticed that some Java developers tend to use Java for every conceivable task.
For example, it is essential to pick up a scripting language like Python for one-off automation tasks or Unix tools like grep/awk/sed for log munging.

Full-stack developers will also need to learn HTML/CSS/Javascript, but I'd recommend backend Java developers learn the basics too. With DevOps gaining
popularity, it doesn't hurt to learn configuration/infrastructure management tools like [Chef](https://www.chef.io) or [Puppet](https://puppet.com/).

## Summary

Despite stiff competition, the Java ecosystem is still one of the best (if not the best) platforms to build modern web application backends. Some heavy-weight,
 enterprise-y baggage still remains, but is not hard to avoid.

[^1]: By "lean" I do not mean [Lean Software Development](https://en.wikipedia.org/wiki/Lean_software_development), rather "lean" in the real sense of the word - devoid of unnecessary burden
