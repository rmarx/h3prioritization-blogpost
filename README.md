HTTP/3 Prioritization demystified
---------------------------------

If you deal with Web Performance, you've probably heard about HTTP resource prioritization. This is especially true since last year, as Chromium added so-called "Priority Hints" with the new [`fetchpriority` attribute](https://web.dev/priority-hints/), which allow you to tweak said prioritizations. You may have also heard that the prioritization system changed between HTTP/2 and HTTP/3. 

However, what exactly does prioritization mean? How does it work under the hood? Why is it important to have some control over it? and, crucially, do all browsers agree on which resources are most important (hint: no, they don't)? This, and much more, down below!

Note: HTTP prioritization is a very expansive topic; I've heard [you can even do a PhD on it :O](https://www.researchgate.net/publication/347519865_Debugging_Modern_Web_Protocols). In this post, I **intentionally** leave out a lot of nuances to focus on the key points. 

# What is prioritization?

HTTP resource prioritization is mainly a concept for HTTP/2 (H2) and HTTP/3 (H3). In HTTP/1.1 (H1), browsers typically open multiple TCP connections (up to 6 per domain) and each connection loads only 1 resource/file **at a time**. Prioritization is implicit by which resources are requested first on the available connections. 

However, in H2 and also H3, we instead aim to improve efficiency by using only **a single TCP/QUIC connection**. If however that single connection could also only have just a single resource "active" at a time like H1, that would be bad for performance. So instead, H2 and H3 can send multiple requests at the same time. Crucially though, this does **not** mean these requests can also be fully responded to at the same time! This is because, at any given time, a connection is limited in how much data it can send by things like [congestion and flow control](https://www.smashingmagazine.com/2021/08/http3-performance-improvements-part2/#congestion-control). 

TODO: congestion control graph

Especially at the start of the connection, we can only send a limited amount of data each network round trip, as the server needs to wait for the browser to acknowledge it successfully received each burst of data. This means the server needs to choose which of the multiple requests to respond to first. 

Let's take an example where the browser has loaded the .html page and now requests 3 resources at the same time on one H2/H3 connection: a `defer`red JS file (100KB), and two .jpg images (300 and 400KB). Let's say the server is limited to sending just 50KB (about 35 packets) in this round trip (the exact numbers depend on various factors). It now has to choose how to fill those 35 packets. What should it send first though? 

You could make the argument that JS is always important (high priority), even though it's deferred. You could also claim instead that there's a high chance that one of the images is a likely Largest Contentful Paint candidate and that one of them should be given preference (but which one?). You could even go as far as to say that jpg images can be rendered progressively, and that it makes sense to give each image 25KB (we call this the "interleaving" or multiplexing of resource data). But what if the deferred JS is the thing that will actually load the LCP image instead...? 

TODO: prio diagram

It's clear that the server's choice here can have a big impact on various Webperf metrics, as some resource data will always be delayed behind "more important" data of a "higher priority". If you send the JS first, the images are delayed by 1 or multiple network round trips, and vice versa. This is especially impactful for things like (render-blocking) JS and CSS, which have to be downloaded in full before they can be applied/executed.  

TODO: multiplex diagram showing resources being done later from holblocking blogpost 

> Note: this is why I hate it when people say that H2 and H3 allow you to send multiple resources **in parallel**, as it's not really parallel at all! H1 is much more parallel than H2/3, as there you do have 6 independent connections. At best, the H2/3 data is **interleaved** or multiplexed on the wire (for example distributing data across the two images above), but usually the responses are still sent sequentially (first image 1 in its entirety, then image 2). As such, as a pedant, I rather speak of **concurrent** resources (or, if you really want, parallel requests and multiplexed responses). 

The problem here is that the server doesn't really have enough information to make the best choice. In fact, it wouldn't even know the JS file was marked as `defer` in the HTML, as that piece of context is not included in the HTTP request by the browser (and servers don't typically parse the HTML themselves to discover these modifiers). Similarly, the server doesn't know if the images are immediately visible (e.g., in the viewport) or not yet visible (user has to scroll down, 2nd image in a carousel, etc.). It also doesn't know about the fancy new `fetchpriority` attribute! If you want some more in-depth examples, see my [FOSDEM 2019 talk](https://www.youtube.com/watch?v=nH4iRpFnf1c) or my [Fronteers 2022 talk](https://vimeo.com/768728308).

It follows that it's really the browser which has enough context to decide which resource should be loaded first. As such, it needs a way to communicate its preferences to the server. This is exactly what HTTP prioritization is all about: **a standardized way for browsers to signal servers in what order requested resource data should be sent**. 

# Prioritization signals

If you look at the priority column in the Network tab of the browser devtools, we see highest -> lowest (or similar) and you might think that's what is sent to the server as well. However, that's (sadly) not the case. 

TODO: devtools prio column


Especially in HTTP/2, there is a much more advanced system at work, called the "Prioritization Tree". Here, the resources are arranged in a tree datastructure. The position of the resource in the tree (what are its parents and siblings) and an associated "weight" influences when it is given bandwidth and how much. When they request a resource, browsers use a special additional HTTP/2 message (the [PRIORITY frame](https://www.rfc-editor.org/rfc/rfc9113.html#name-priority)) to tell the server where in the tree the resource belongs. 

TODO: ff prio tree

This system is very flexible and powerful, but as it turns out, it's also complex; too complex. So complex, that even today [many HTTP/2 implementations have serious bugs in this system](https://github.com/andydavies/http2-prioritization-issues), and some stacks simply don't implement it at all (ignoring the browser's signals). Browsers also [use the system in very different ways](https://speeder.edm.uhasselt.be/www18/files/h2priorities_mwijnants_www2018.pdf).

This is why, when work started on HTTP/3, we decided a simpler system was needed. This is what eventually became [RFC 9218: Extensible Prioritization Scheme for HTTP](https://www.rfc-editor.org/rfc/rfc9218.html). Instead of a full tree, this setup has just 8 numerical priority levels (called "urgency") and one "incremental" boolean flag to indicate if a resource can be interleaved with other resources or not (progressive jpgs: yes please. Render-blocking JS: probably best not). By default, a resource has an urgency of 3 and is non-incremental. 

TODO: priority ranking

The new system is also simpler in how it sends the urgency and incremental signals: instead of a special HTTP/3 message, it can just use a new textual HTTP header called, shockingly, `priority`. The hope is that this overall simpler approach is easier to implement and debug, and that it will lead to much better support and fewer bugs than we had with the H2 system (spoiler for later: that's not exactly true yet). 

TODO: priority header example

> That's at least the idea. In practice, the HTTP header can only be used to signal a resource's **initial** priority. If the priority needs to be updated later (say a lazy loaded image first gets low priority, but needs to switch to high when scrolled into view), this sadly cannot be done using an HTTP header. For that, we do need a special H3 (and H2!) binary message: the [PRIORITY_UPDATE frame](https://www.rfc-editor.org/rfc/rfc9218.html#name-the-priority_update-frame). These details aren't really important for most people, but I mention this because while Firefox and Safari use the HTTP header, Chromium does not. For "reasons", Google only uses the PRIORITY_UPDATE frame, even to signal the initial priority (put differently: it just immediately overrides the default priority). 

It's important to note that this new scheme is not exclusive to HTTP/3. Indeed, the intent is that it would be backported to existing H2 implementations over time (though, by my knowledge, no H2 stack has adopted this today). Additionally, it's called the "extensible" scheme, with a view towards adding additional parameters beyond "urgency" and "incremental" in the future. 


# Browser differences

As I mentioned above, browsers used the complex HTTP/2 system in very different ways. Each of the major browser engines (Chromium, Firefox and Safari) produced radically different prioritization trees and signals. Cloudflare has a quite [extensive blogpost](https://blog.cloudflare.com/better-http-2-prioritization-for-a-faster-web/) on this. 

As such, with the new HTTP/3 setup, I was curious to find out if there were still big differences in the browsers' approaches, as all 3 currently utilize the new system for HTTP/3. Sadly, no one had looked into this yet, so it was time to roll up my sleeves and get dirty! I immediately ran into two problems:

1. Observing the new signals was difficult, as no tools have support for them yet. They don't show up (in their raw form) in the browser devtools, nor in WebPageTest. I decided to [modify the excellent aioquic HTTP/3 server](https://github.com/http3-prioritization/aioquic/tree/priority-logging) to add some additional logging for the prioritization signals.

2. I asked around, but there was no single test page that included all the different resource loading options that can impact prioritization (async/defer, lazy/eager, fetchpriority, preload/prefetch, etc.). I thus [created my own test page](https://github.com/http3-prioritization/prioritization-test-page) that includes a whopping 36 different situations to make it easy to test.  

I then hosted the custom test page on the custom HTTP/3 server and loaded it with the 3 browsers. I stored the HAR files from the browsers, and the logs from the server to find out what the browsers were actually sending on the wire. More details on the results are/will be [available on github](https://github.com/http3-prioritization/prioritization-experiments), and I'll highlight the key points below.


## Raw details

frame vs header
i vs no i
default vs no default
which urgency values 

https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/platform/loader/fetch/resource_fetcher.cc;l=141;drc=727666becab9ab1b26080180df45f043cac5f0aa

## Learnings






TODO: mention chromium plans to change signals soon
TODO: mention that h2 and h3 prioritization differs, another reason to start using H3 :)

# Server differences

No name-and-shame yet: give it anohter 6-12 months to catch up. 

TODO: comparison image 

# Conclusion

TODO: hopefully, this allows you to better understand things discussed in the web.dev fetchpriority article. Hopefully it also shows how important it is to have fetchpriority in all browsers (FF is adding it, imagine that!). 


