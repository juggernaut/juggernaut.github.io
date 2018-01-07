---
layout: post
title:  "Why I don't always follow YAGNI"
date:   2018-01-06 20:49:00
categories: software 
---

Ignoring the clickbait-y title for a moment, I generally agree with the [YAGNI](https://en.wikipedia.org/wiki/You_aren%27t_gonna_need_it) principle.
I want to share what I believe is an exception to that rule - that is, a situation where doing some upfront work even if you don't immediately need it will pay off in the long run.

Specifically, if you're building services in an OOP language that expose a public[^1] API, <b>I strongly recommend creating separate classes for
 over-the-wire representations of your API entities and your business logic entities.</b> In architect-speak, separate your [Domain Model](https://martinfowler.com/eaaCatalog/domainModel.html)
 from your [Data Transfer Objects](https://martinfowler.com/eaaCatalog/dataTransferObject.html).

In my experience, not doing this leads to the issues detailed below.

### Security

Most applications use libraries to serialize objects to an over-the-wire representation like JSON. [Jackson](https://github.com/FasterXML/jackson)
 is one such popular library for Java.  Jackson does quite a bit of behind-the-scenes "magic" to free you from writing boilerplate. As such, it's not always
 intuitive what class properties are serialized.  For example, a private field becomes serializable by default if you add a getter for it! 

To demonstrate, let's suppose you're building a dating app that offers an API and you have a `Person` class:

{% highlight java %}
public class Person {
    public String name;
    public String bio;
}
{% endhighlight %}

Instances of this class are serialized to JSON and sent over the wire in API responses. Suppose your app also calculates an internal "attractiveness score"
for match-making purposes. If you're using the same class for this so-called "business" logic, you might be tempted to add extra behavior like so:

{% highlight java %}
public class Person {

    public String name;
    public String bio;

    // This is private, so it doesn't get serialized. Right? Right?
    private int attractivenessScore;

    public int getAttractivenessScore() {
        return attractiveNessScore;
    }
}

/* JSON output from jackson
{"name":"Chad","bio":"I like long walks on the beach","attractiveNessScore":0}
*/
{% endhighlight %}

Congrats! You have now ruined the self-esteem of your unsuspecting users! This is a hypothetical example, but if you're working with
any sort of security-sensitive or personal data, the potential ramifications are huge.

### Maintainability

Maintainability issues arise from a lack of separation of concerns between the business and presentation layers. For example, if you want a `snake_case` naming strategy
for your API fields, you might add a `JsonNaming(SnakeCaseStrategy.class)` annotation to the class. Now, if you want to factor out
your business logic into a separate library (for code reuse), you have to pull in the `jackson-annotations` dependency for absolutely
no reason. Another example is [Swagger](https://swagger.io/) annotations - they should strictly live in the presentation layer.

### Versioning

For any long-term API, [versioning](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/) is inevitable. Your software is always evolving,
which means adding new functionality, fixing broken functionality, renaming fields etc, sometimes in a backwards-incompatible way. At this point,
you would have to create new classes for the new version anyway, or even worse, add crufty branching code that juggles data based on versions.

## Summary

If you're building a medium-to-large application with an API contract, [front-load the pain](https://lifehacker.com/to-boost-happiness-stack-the-pain-1778255297)
and create separate classed for your API presentation layer and business layer. PS: there are [tools](http://mapstruct.org/) to ease this pain, but I haven't personally used them.

[^1]: I'd argue even if the only consumer of your APIs is your own frontend/mobile app, it's still a public API
