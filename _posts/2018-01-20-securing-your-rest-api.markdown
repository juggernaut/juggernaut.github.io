---
layout: post
title:  "Exploring REST API Authentication Mechanisms"
date:   2018-01-20 20:40:00
categories: software 
---

With the popularity of single page applications (SPAs) and mobile apps, the role of web application backends has changed. They are no longer behemoths that
that render server-side HTML, but lightweight REST APIs that do little more than serve data in a structured format (usually JSON).
As a result of this shifting landscape, there seems to be some confusion and varying advice on how to securely implement authentication. Traditional web
applications use cookies and server-side sessions, but modern applications employ a variety of authentication mechanisms from JWTs to request signing to 
OAuth.

This post is a distillation of my research on this topic around the web. The answer to this question (like most things), is that it depends on the use case.
In essence, it depends on _who_ interacts with the API. Is it other servers running custom programs, or is it actual humans interacting with the API via
a browser or mobile app? This post mostly focuses on the latter.

NOTE: It is presumed that the API is accessible over TLS only. Allowing plain HTTP in 2018 is not an option.

## Machine-to-Machine

If your API is going to only be used by other servers, HTTP basic auth is sufficient. Remember though, TLS is absolutely mandatory (in all cases),
so that the secret is not sent in clear-text. This is the approach taken by API-as-a-product companies like Stripe and Twilio.
Basically, this means generating a sufficiently random secret for each user (or account).
With each API request, the client sends the username and secret as a base64-encoded string in an `Authorization` header.

Setting it up is easy - XXX: TODO

## Browser/Mobile users

Most applications are designed for human users who interact with the application through a web browser. This means, you need to implement password-based login -
you can't just generate a random secret token and expect a human to remember it. Other practical considerations include needing to guard against browser-based exploits like XSS and CSRF.

### To JWT or not?

JSON Web Tokens (JWTs) are a popular way to secure REST APIs. In a nutshell, a JWT is a token issued by the server and signed with a secret key known only to
the server. The client then presents this token with every request, and the server verifies the signature and grants access according to the claims contained
within. Since the authentication process does not involve a datastore lookup, it is considered to be "stateless". REST puritans often consider this
approach more "RESTful" than other stateful authentication mechanisms.

After reading a few blogs and comments by security experts, I'm convinced by their argument that JWTs are a bad
idea for 99% of use cases. The linked articles do a much better job of explaining why, but to summarize:
1. A JWT can't be invalidated server-side: Since JWTs are "stateless" authentication, a token that has already been issued can't be revoked until it
   expires. You _can_ implement server-side blacklists, but this means you need a database lookup, thus negating the "stateless" advantage of JWTs.
2. JWTs encourage insecure practices: JWTs are relatively large in size, so they might exceed the cookie size limit (4k). This necessitates storing them in
   local storage, which can leave you [vulnerable to XSS attacks](https://stormpath.com/blog/where-to-store-your-jwts-cookies-vs-html5-web-storage)
3. Critical vulnerabilities have been found in many JWT implementations
4. You simply don't need the complexity of JWTs for most use cases

### The solution

Instead of JWTs or other fancy auth mechanisms, just go with the simplest possible solution - server-side state. _Gasp_, isn't REST supposed to be "stateless",
though?  Well, the reality is that security is hard as it is, without the needless complexity that comes with trying to 100% conform to some architectural
philosophy. Unless you are Google or Amazon, you're going to need to be pragmatic and choose simplicity.

The authentication flow looks like this:
1. User logs in, which triggers a request to a login endpoint in your app
2. The endpoint verifies the credentials, and generates a cryptographically secure token with an expiration time
3. The token is saved in a backend datastore and returned to the client in a secure, http-only cookie
4. The browser sends the token automatically with subsequent requests in a cookie. This can be used to authenicate the request

Wait, isn't this how server-side sessions work? Indeed, and this should cover the majority of use cases. Which means that if you're deploying in a servlet
environment, you should be able to use the built-in session manager of your container. Of course, you still need to guard against CSRF, which I'll explain
later.

If your API is actually RESTful though, you probably don't need to store anything other than the authenticated user in the session. Depending on your use case,
configuration and overhead of session management may not be worth it. Another issue is that server-side sessions are designed to be ephemeral. This is great 
for traditional web applications, but if you're building a mobile app for example, you typically don't want to annoy the user with frequent login prompts. 
I guess this is what drove a lot of developers to use stateless auth like JWTs in the first place. It's attractive to simply mint a JWT with a long expiry and
forget about maintaining server-side state.

### Rolling your own for fun, not profit 

DISCLAIMER: I'm not a security expert, and this code is not reviewed by a security expert. For production, you should use a hardened library like
Spring-Security.

I have to admit Spring-Security has an enterprise-y smell to it. I wanted to implement a lightweight authentication flow from scratch in the
framework of my choice, Dropwizard, as a learning exercise. There are ways to integrate Dropwizard with spring security, which is probably what you should do in
production.

With that out of the way, let's get down to the implementation. I like to take a progressive approach, starting out with the simplest possible working code, and
then working up to the completed solution. Let's start with creating the login resource. It takes a username and password, generates a random token and returns
it in a `Set-Cookie` header. Note that secure password storage and verification is outside the scope of this post; please refer to [this](https://codahale.com/how-to-safely-store-a-password/) excellent post by Coda Hale for how to do that.

{% highlight java %}
@POST
public Response login(
        @NotEmpty @FormParam("username") String username,
        @NotEmpty @FormParam("password") String password) {
    if (username.equals("johndoe") && password.equals("SECRET")) {
        String newToken = generateAndStoreToken(JOHN_DOE_USER_ID);
        // NOTE: The last two arguments, HttpOnly and Secure Flags must be set to true
        NewCookie cookie = new NewCookie(TOKEN_COOKIE_NAME, newToken, null, null, null, EXPIRES_SECS, true, true);
        return Response.noContent().cookie(cookie).build();
    }
    return Response.status(Response.Status.UNAUTHORIZED).build();
}
{% endhighlight %}

We haven't implemented `generateAndStoreToken` yet, so let's see how we can generate the token first.

{% highlight java %}
public class TokenCredentials {

    private static final SecureRandom PRNG = new SecureRandom();

    public static String generateRandomToken() {
        final byte[] tokenBytes = new byte[20];
        PRNG.nextBytes(selectorBytes);
        return BaseEncoding.base16().lowerCase().encode(tokenBytes);
    }
}
{% endhighlight %}

All this does is read 20 bytes from `SecureRandom` which reads from a platform-dependent source of randomness (on Linux, the source is usually `/dev/urandom`),
and hex-encodes them to a String.

#### Storing the token

We need to store the token in a backend database, so that we can verify that subsequent requests carry the correct token. Let's start by defining an
`auth_details` table:

{% highlight sql %}
CREATE TABLE auth_tokens (
    id int(11) unsigned NOT NULL AUTO_INCREMENT,
    auth_token char(40) NOT NULL,
    user_id int(11) unsigned NOT NULL,
    expires timestamp NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY (user_id)
);
{% endhighlight %}

This schema assumes that the user can only be logged in from one browser/device. You can trivially extend this to support multiple devices by including a device
id/name and generating an auth token per device. Also, to keep things simple, we assume that we represent an user by an id and actual user details are stored in
another table.

Now, let's define a Data Access Object (DAO) for this table. Dropwizard bundles the excellent JDBI library which allows us to very easily create DAOs:

{% highlight java %}
public interface AuthTokenDao {

    @SqlUpdate("INSERT INTO auth_tokens (user_id, token, expires) VALUES (:userId, :token, TIMESTAMPADD(SECOND, :expires, NOW()))")
    int insertToken(@Bind("userId") int userId, @Bind("token") String token, @Bind("expires") long expires);

}
{% endhighlight %}

By giving our login resource access to this DAO, we can insert the generated token in a DB.

### Hooking into Dropwizard

First, a little introduction to the authentication hooks Dropwizard provides:

1. An interface called `Authenticator` with a single method `authenticate()`, which takes credentials as a parameter and optionally returns an authenticated user.
2. An abstract class `AuthFilter` with common logic that you can extend to easily create and register a Jersey filter.

Implementing the `Authenticator` interface turns out be straightforward. We simply query the database if a user exists with the provided token and that it
hasn't expired. Let's add a method to our DAO that does this:

{% highlight java %}
@SqlQuery("SELECT user_id from auth_tokens where token = :token AND TIMESTAMPDIFF(SECOND, NOW(), expires) > 0")
Integer getUserForToken(@Bind("token") String token);
{% endhighlight %}

With that, our authenticator implementation looks like this:

{% highlight java %}
public class TokenAuthenticator implements Authenticator<String, User> {

    private final AuthTokenDao dao;

    public TokenAuthenticator(AuthTokenDao dao) {
        this.dao = dao;
    }

    @Override
    public Optional<User> authenticate(String token) throws AuthenticationException {
        return Optional.ofNullable(this.dao.getUserForToken(token)).map(User::new);
    }
}
{% endhighlight %}

The final piece of the puzzle is implementing a Jersey filter that will extract the token from the cookie and call our custom authenticator to decide whether the
request should be allowed or not. Fortunately, the Dropwizard provides a class which does most of this work for us, and we only need to extend it and implement
the main `filter()` method:

{% highlight java %}
public class TokenAuthFilter extends AuthFilter<String, User> {

    public static final String TOKEN_COOKIE_NAME = "token";

    @Override
    public void filter(final ContainerRequestContext requestContext) throws IOException {
        final Map<String, Cookie> cookies = requestContext.getCookies();
        final Cookie tokenCookie = cookies.get(TOKEN_COOKIE_NAME);
        if (tokenCookie == null ||
                Strings.isNullOrEmpty(tokenCookie.getValue()) ||
                !authenticate(requestContext, tokenCookie.getValue(), "COOKIE_TOKEN_AUTH")) {
            throw new WebApplicationException(unauthorizedHandler.buildResponse(prefix, realm));
        }
    }
}
{% endhighlight %}

Most of the heavy lifting is done by the `authenticate()` method of the base class, and all we need to do here is get the cookie value from the request.

The only thing left to do now is to let our filter know that it needs to use our custom `TokenAuthenticator` and register the filter with Jersey. You can do
this in the main `Application` class, like so:

{% highlight java %}
final TokenAuthenticator authenticator = new TokenAuthenticator(authTokenDao);
environment.jersey().register(new AuthDynamicFeature(
    new TokenAuthFilter.TokenAuthFilterBuilder()
        .setAuthenticator(authenticator)
        .buildAuthFilter()));
{% endhighlight %}

That's it! You can now use the built-in `@Auth` annotations to protect resources:

{% highlight java %}
@GET
public Response getGreeting(@Auth User user) {
    final Greeting greeting = new Greeting("Hello there!");
    return Response.ok(greeting).build();
}
{% endhighlight %}

## Wait, we aren't protected yet!

Storing the authentication token in a cookie opens us up to Cross-Site Request Forgery (CSRF) attacks. Jersey includes a
[CrsfProtectionFilter](http://blog.alutam.com/2011/09/14/jersey-and-cross-site-request-forgery-csrf/), but DO NOT USE IT! The rationale for this filter is 
that custom headers can only be set by Javascript and that too within its origin (Same-Origin Policy). As such, it would be sufficent to verify only the
existence of the custom header, and its value does not matter. However, Flash (*sigh*) can be used to craft a request with [arbitrary
headers](http://lists.webappsec.org/pipermail/websecurity_lists.webappsec.org/2011-February/007533.html), so this is not a secure strategy.

Another commonly used CSRF mitigation is double-submit cookies i.e require all POST requests to include a form parameter that should be equal to the value set
in a Cookie header. This works because a cross-origin attacker cannot read or write cookie headers, thus forcing them to guess a sufficiently random value.
For example, the popular Python framework, Django, uses this approach. Again, [there are ways to defeat this
technique](https://security.stackexchange.com/questions/59470/double-submit-cookies-vulnerabilities/61039#61039) - if an attacker controls a sub-domain
of your site, they can plant arbitrary cookies on your domain, tricking your server into accepting the request.

The most secure technique involves using [Synchronizer
Tokens](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet#Synchronizer_.28CSRF.29_Tokens). This is essentially the same
technique as generating authentication tokens, the only difference being we give Javascript access to the cookie by _not_ setting the `HttpOnly` flag. This will
allow your client-side Javascript to extract the CSRF token from the cookie and send it in a separate header. Your app can then verify this token server-side,
just like we did with the auth token.

Obviously, you shouldn't set the CSRF token value to be the same as the auth token after we went through the trouble of
making the auth token unreadable to Javascript. A SHA-256 digest of the auth token will suffice, which is what AngularJS
[recommends](https://docs.angularjs.org/api/ng/service/$http#cross-site-request-forgery-xsrf-protection). This is useful in cases when you don't want the
overhead of generating and verifying an additional token. In our little learning exercise, however, we can just add an additional column, `csrf_token`, to our
database table. Since we are doing a query per request anyway, we won't incur any additional cost. The implementation is pretty straightforward, and I won't get
into it any further. The full example can be found on github here.

### A note about timing attacks

Timing attacks rely on exploiting the fact that standard string comparison functions will exit as soon as the first non-matching character is found. For
example, comparing the string "ABC" to "XYZ" is slightly faster than comparing it to "AYZ". With enough trials and a reliable signal, an attacker could,
in theory, deduce the secret.

In our case, the auth token comparison occurs in the database query. One way to protect against this is to use ["split
tokens"](https://paragonie.com/blog/2017/02/split-tokens-token-based-authentication-protocols-without-side-channels) - in essence, you split up the auth token
into two parts, a selector and a validator. Use the selector as a key to the database query and then do a constant-time string comparison against the validator
in your app. [MessageDigest.isEqual](https://docs.oracle.com/javase/8/docs/api/java/security/MessageDigest.html#isEqual-byte:A-byte:A-) built-in to the JDK, and
guava's [HashCode.equals](https://google.github.io/guava/releases/21.0/api/docs/com/google/common/hash/HashCode.html#equals-java.lang.Object-) both do
constant-time comparisons.

However, pulling off such an attack is [non-trivial](https://security.stackexchange.com/a/111128), especially over noisy networks like the public Internet. I
wouldn't bother guarding against such attacks, unless you are building an extremely security-sensitive app or are a high value target.
