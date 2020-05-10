---
layout: post
title:  "Rediscovering the joy of hacking"
date:   2020-05-10 18:40:23
categories: technology python
---

I fell in love with computers because I could make the computer do whatever I told it to - and that felt like magic. Well, that, and computer games :-)

It can be easy to lose sight of this fact when you're knee deep in the "business" of software development. Agile, JIRA, architecture documents, dealing with PMs etc can get quite exhausting.
 While I fully understand the value of these activities (or at least a subset of them), sometimes you just want to bang out code and solve a problem you have without having to  answer to _anyone_  or worry about handling SPOFs.. or unit testing .. or load testing.

A recent [post](https://news.ycombinator.com/item?id=23072442) on Hacker News in a thread titled "_Extremely disillusioned with technology. Please help_" resonated with me:
>Don't make enterprise software. Don't write unit tests. Don't accept pull requests. Simply write software for yourself and have fun doing it. Forget refactoring code into modules, just fucking code

 and spurred me into taking action when I needed to solve a problem - my Mom is visiting from India, so due to COVID-19 I'm trying to limit stepping out of the house as far as possible. 
My local Indian grocer actually takes online orders for home deliveries, but as to be expected in these strange times, no delivery windows were open whenever I checked.

 I was curious how easy it'd be to automate the equivalent of refreshing the page and notifying me when a delivery window opened up.
Fortunately for me, it turned out to be pretty straightforward. The grocer's website was built with "Homesome", a platform that helps local businesses sell online (I'd never heard of them prior). I fired up developer tools and found that a single request to their API was fetching pretty much all data related to my order:

<pre>
GET /user/basket HTTP/1.1
Host: partnersapi.gethomesome.com
Connection: keep-alive
Pragma: no-cache
Cache-Control: no-cache
location: [redacted]
Accept: application/json, text/plain, */*
pricelist: C61EB5D2-3C17-4AB9-A326-BD40522DEEDE
<b>apikey: [redacted]</b>
<b>emailid: [redacted]</b>
<b>authToken: [redacted]</b>
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: [redacted]
Accept-Encoding: gzip, deflate, br
Accept-Language: en-US,en;q=0.9
</pre>

The response included all the items in my cart, fees and delivery window information:
<pre>
{
    "basketItems": [
        {
            ...
        }
    ],
    "orderFee": {
        "delivery": {
            "availableTimes": [],
            "unavailable": {
                "reason": "At Capacity"
            },
            ...
        }
    }
}
</pre>

If you notice, the request has no Cookies or indeed any way for the backend to easily expire requests after a certain time (like a nonce)[^1]. All that's needed for authentication are the API key and auth token sent as headers. This meant I could pretty much copy the request verbatim and run it in a loop in a script without worrying about refreshing credentials. So, I copied the request from dev tools and stuck it in [h2c](https://curl.haxx.se/h2c/), a nifty tool that spits out the equivalent `curl` command [^2]. Next, I whipped up a python script that did the following:

1. Call the curl command using `os.system()` - resisted my perfectionist side to use the `requests` library for this
2. Parse the json response and determine availability based on whether `response['orderFee']['delivery']['availableTimes']` is empty or not. Write the availability as a string "true" or "false" to a file `availability.txt`
3. If the availability from the previous run was `false` and the current availability is `true`, send an SMS to my phone using Twilio [^3]

I then fired up a DigitalOcean droplet, and set up a cron job to run the script every minute. In all, it took me about 45 minutes from start to finish, and a lot of that time was spent installing python and looking up cron syntax. I then smugly sat back and relaxed waiting for the notification to arrive.

Sure enough, after a few hours, I received a notification that a window had opened up! (_Just kidding :-) true to its form, the hacky script had a bug that failed to send a notification[^4]. Luckily, I had good logging and caught that it had found an available window in the logs - I'd logged in a few hours later because I was suspicous something was broken_)

Still, it was extremely refreshing to write software to solve my _own_ problem for once, with very little ceremony involved. I highly recommend doing this from time to time - and it just might be the thing that keeps you from being jaded by the software industry.

[^1]: There are multiple security issues with the platform. I'm trying to get in touch with them to get the issues fixed.

[^2]: I later learned you can do this directly from Chrome dev tools - shows you I don't do frontend much.. or at all.

[^3]: I work at Twilio, but this blog is in no way affiliated to my employer, nor does it claim to speak on their behalf.

[^4]: I was opening the `availability.txt` file in `r+` mode and calling `file.truncate(0)` before writing out the current availability. Turns out `truncate()` preserves file position, so calling `truncate(0)` after reading zero-fills the previous bytes - this, of course, threw off my string comparisons.