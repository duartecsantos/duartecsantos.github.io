---
layout: post
title: Cross-Site Scripting in Go
subtitle: Guessing unknown MIME types
tags: []
---

The other day I was investigating the findings of a certain SAST scanner for a Go project. 
In particular, I was analyzing the Reflected Cross-Site Scripting (XSS) results. 
At first glance, one of these results looked like a True Positive (TP) — it was writing a partially user-controllable value directly to a response stream. 
As I have no experience with Go, I decided to put my assumptions aside and investigate this result a little further. 
In the worst case, I wasted some time, but I learned a bit about a new language. Not bad!

That said, here's a snippet that more or less reflects the case in question:

```go
package main

import "net/http"

func handler(w http.ResponseWriter, r *http.Request) {
    path := r.URL.Path
    w.Write([]byte(path))
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
```

This snippet creates and starts an HTTP Server listening on port 8080, handling requests on server's root.
The `handler` function, which handles requests, simply writes the value of the requested path to the response stream.

Sounds pretty simple, huh?
Would you also say that this snippet is directly vulnerable to XSS, as I initially assumed?

There's an important detail that I haven't told you about, but which makes all the difference.
After some time investigating how this package works, I realized that if the `Content-Type` HTTP response header is not explicitly defined, it will try to guess an appropriate value for that header via `http.DetectContentType`, which follows the [WhatWG](https://mimesniff.spec.whatwg.org/#identifying-a-resource-with-an-unknown-mime-type) specification.
For example, if we write "test" to the response stream, the `Content-Type` HTTP response header will be set to `text/plain`.

You must be thinking, why can't we just visit `https://.../<script>alert(1)</script>` and get XSS?
Script tags are valid HTML elements, so the `Content-Type` should be set to `text/html`, no? Yes and no.
If we look at the [WhatWG](https://mimesniff.spec.whatwg.org/#identifying-a-resource-with-an-unknown-mime-type) specification, we can see that should be the case:

> The case-insensitive string "<SCRIPT" followed by a tag-terminating byte.

But we're not trying to write `<script>alert(1)</script>` to the response stream.
What we're writing is `/<script>alert(1)</script>` — notice the leading forward slash.
That first character is enough for this response to be assigned `text/plain` as its `Content-Type` instead of `text/html` and that makes all the difference. 

I did try to trick this prediction mechanism, but to no avail.
The attempt I found most interesting was to use `head><body><script>alert(1)</script></body></html>` as the path, because in conjunction with the leading forward slash it would make it look like we were interpreting incomplete HTML (i.e. `/head><body>...</body></html>`), but it didn't work.
That said, there may be some way of cheating this mechanism that I haven't thought of.
If anyone wants to exchange ideas, feel free to contact me.

Now, of course this response can still be easily consumed by a user in an insecure way and result in an XSS, but in isolation that's not going to happen.
In product terms, this could even be a good argument for a SAST to mark this as a True Positive, but that kind of debate is outside the scope of this blogpost.
In any case, no one should blindly trust this type of mechanism as a guarantee of security...
Please, if you want to implement similar code, just use the `html/template` package to produce escaped output. It's simple and safe.

Finally, as a bonus, I noticed a very interesting detail in the documentation for [DetectContentType](https://pkg.go.dev/net/http#DetectContentType):

> It considers at most the first 512 bytes of data.

In other words, in this case, our tainted response is prefixed by a non-problematic constant string ("/"), which prevents the `Content-Type` from being interpreted as `text/html`, but rather as `text/plain`.
But if our response were suffixed by that string (or any other type of content) instead of prefixed, it would be possible to simply fill those 512 bytes with our payload plus some padding that corresponds to `text/html` type content and thus trigger XSS!
Another cool idea of a challenge I can implement for a CTF, hmm...

Thanks for reading, I hope it was interesting! 🙂
