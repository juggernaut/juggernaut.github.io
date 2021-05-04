---
layout: post
title:  "Sending webhooks securely"
date:   2021-05-03 18:40:23
categories: technology webhooks
---

Webhooks are a popular way to glue web applications together. On the surface, they're just HTTP requests with a twist -- the "customer" in browser apps is usually the client whereas with webhooks, the server (consumer) is the "customer". This creates some peculiar security implications as we shall see. Turns out it's quite tricky to implement a secure webhook sender -- over the past few months, I've found vulnerabilities even in popular apps that send webhooks. This post is aimed at guiding developers to implement secure webhook senders.

## SSRF
SSRF is when you can trick a server into sending a webhook to its own organization's internal resources (including possibly to itself). If the app provides a way to view webhook responses (e.g for debugging), it could leak sensitive data. A common target is the EC2 instance metadata endpoint (169.254.169.254) that can return IAM credentials. 

Even without access to webhook responses (blind SSRF), an attacker can map out internal networks, modify internal services or 
even remote code execution.

I found a fun SSRF vulnerability in a website monitoring tool (name undisclosed on request) by a popular indie hacker[^1]. The tool accepts a URL to monitor, then makes a request to that URL from the backend by calling an API endpoint `https://api.$unnamed.io/ping?url=$url`. You could set the `url` query param to `http://127.0.0.1/ping` and it would send a request to itself! You could even nest URLs to arbitrary depths (presumably until the URL length limit) to achieve an amplification effect.

### Mitigation
Goes without saying that we should check that the protocol is HTTP/HTTPS only.
Then, at first glance, it seems fairly straightforward to mitigate SSRF attacks - we could block URLs containing private IPs. This is not enough, however. The attacker could simply create a DNS record that points to the private IP address. Therefore, we need to check IPs _after_ DNS resolution. Turns out this is tricky to do correctly. A naive implementation would first do a DNS lookup of the hostname in the URL using `gethostbyaddr()` or equivalent, validate that the IP is not private, then make the actual request to the URL. This involves two DNS resolutions. An attacker can exploit this by returning a valid public IP address on the first resolution, but a private IP the second time. This is called a _DNS rebinding_ attack.

I found two apps vulnerable to this attack - PagerDuty and another I can't disclose because the fix is still in progress.

The right way then is to resolve the hostname only once, and use the resulting IP address to establish the TCP connection for the request. Be careful though -- in some HTTP clients, it may break SNI because the SNI hostname is now taken to be the IP address instead of the hostname.

If you're using Go, Andrew Ayer has a great [blog post](https://www.agwa.name/blog/post/preventing_server_side_request_forgery_in_golang) (with code) on this.

## Authentication
As I alluded to earlier, the "customer" in this case is the webhook consumer (the HTTP server). It's important for the consumer to be able to
authenticate the sender. For example, an app that receives incoming SMS via Twilio webhooks would want to make sure that the sender is indeed a Twilio server and not a rando running curl from their laptop. There are multiple ways to achieve this:

### Request signing
The sender calculates a signature based on request content and sends it along with the request as a header. The receiver can then validate this signature to authenticate the sender. You can either use a shared secret to generate a HMAC (more common), or a private/public keypair to generate a digital signature (DSA). Some things to keep in mind:
* Use at least SHA-256 if using HMAC
* Include a timestamp in the signature (and as a header) to mitigate replay attacks
* Avoid JWTs -- the extra complexity is just not needed here
* Avoid re-using API keys as the shared secret
* Provide a way to revoke keys in case of a leak
* Provide a way to rotate keys periodically (preferably via an API)

An alternative to HMAC is to generate a private/public keypair. Give out the public key to the consumer, and sign requests with the private key. This way, you avoid burdening the consumer with having to safeguard the shared secret -- leaked secrets on Github etc are a common source for hacks. [Ed25519](https://ed25519.cr.yp.to/) is a fantastic algorithm for this use case.

One "drawback" (for lack of a better word) of request signatures is that the signature needs to be verified in application code, hence pushing the security boundary further inside the network. For traditional IT orgs that rely on perimeter security, this may be a problem. Further, it's easy to miss signature verification when adding new routes if testing or review process is inadequate (of course, webhooks continue to be received successfully even with no signature verification, so there's no good way to catch the issue even after deployment)

I discovered that in [StatusPage's](https://www.atlassian.com/software/statuspage) integration with [PagerDuty](https://pagerduty.com), they were missing authentication on incoming PagerDuty webhooks[^2]. The integration allows Pagerduty incidents to automatically
create StatusPage incidents. As such, I could create incidents on a public facing StatusPage with a simple curl (imitating a Pagerduty webhook request) from my laptop. 

### Mutual TLS
With mutual TLS (aka client-side certificates), the webhook consumer can enforce authentication at the edge, instead of having to implement it in the application layer. Apps like Slack, PagerDuty and Google DialogFlow support mutual TLS. 

That said, mutual TLS isn't widely supported - the major cloud providers' load balancer products don't support it. It's also not very well understood, and is hard to configure even in popular servers like nginx.

Another potential issue with this approach is the _confused deputy problem_. A malicious actor can easily create their own account on the sender's service and direct webhooks towards any arbitrary destination. Such webhooks would pass the mutual TLS handshake, so it is not enough merely to verify the authenticity of the sender -- the consumer also needs to verify that the request was generated on their own account/customer ID. This necessitates inspecting the request body or headers to find the account/customer ID (if there's one included), which is awkward to say the least.

I found that Google DialogFlow and PagerDuty both have this issue[^3].

We could avoid this issue if we had a way to verify ownership of the webhook destination URL. This would ensure that a malicious actor wouldn't be able to direct his/her account's webhooks to a legitimate customer without actually owning the customer's endpoint.

Slack and Twitter send verification requests (Twitter calls them CRCs for some reason) with a token that the webhook consumer is required to encrypt with a shared secret and send back.

### IP Allow Lists
If you publish a known set of IP addresses (or subnets) that webhooks will be sent from, consumers can allow only those IPs in their firewall. Not very flexible, as you have to overprovision your IP pool to accomodate future scaling -- given the cost of precious IPv4 addresses, it may not be worth it. Some traditional corporations might require this though -- note that you can offer a combination of 2 or even all 3 authentication methods. They aren't mutually exclusive.

## Certificate Chain Verification
Verifying TLS certificate chains is [notoriously tricky](https://medium.com/@sleevi_/path-building-vs-path-verifying-the-chain-of-pain-9fbab861d7d6). Libs like OpenSSL and BoringSSL are broken when it comes to path building and verification. Which means you should avoid languages that rely on e.g OpenSSL bindings (e.g Ruby/Python) to send webhook requests. Several webhook implementations -- including [Stripe](https://twitter.com/sleevi_/status/1266777220510597122?s=20) -- had outages due to the [AddTrust root expiration issue](https://www.agwa.name/blog/post/fixing_the_addtrust_root_expiration) last year[^4]. 

Instead, use languages like Java or Go that have good native implementations for path building and verification. If you absolutely have to use Ruby/Python/PHP, front it with a TLS-terminating proxy at the edge.

A related problem is keeping trusted CA roots up-to-date. Operating systems don't bundle the latest CA certs, and keeping them updated is a PITA. Java bundles its own trust store, but suffers from similar problems. I suggest using the latest [Mozilla CA bundle](https://wiki.mozilla.org/CA/Included_Certificates) instead.

[^1]: The vulnerability has still not been fixed properly
[^2]: I received a bug bounty for this
[^3]: Google accepted the report but found that it didn't qualify for a reward, and PagerDuty sent me a T-Shirt (something something all I got was this lousy t-shirt)
[^4]: Stripe open-sourced "smokescreen" which is a Go HTTP CONNECT proxy for webhooks, but TLS termination is still handled by (presumably) Ruby.