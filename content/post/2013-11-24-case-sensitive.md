---
title: The Case of The Somewhat Case-Sensitive HTTP Method
date: 2013-10-24T22:30:00+11:00
description: A cautionary tale about debugging and RFC specifications
aliases:
  - /blog/case-sensitive.html
---

## Act 1 - How Did We Get Here?

So I spent this afternoon looking into an issue one of our users was having with
a specific GitHub API.

What do I mean by "issue"? Well, I'm sure you've seen the tweet like this
going around:

 - 1xx: hold on
 - 2xx: here you go
 - 3xx: it's over there
 - 4xx: you f***ed up
 - 5xx: I f***ed up

The user was getting a 502 repeatedly from this API, and the contents were not
JSON like he was requesting, but a very boring HTML page:

```
<html><body><h1>502 Bad Gateway</h1>
The server returned an invalid or incomplete response.
</body></html>
```

The initial theory we had was about certificate validation failing, but I was
suspicious because he'd had other requests succeed inside the same test - it
was just this last one that was failing.

The other details are a boring, but it came down to three things:

1. He had a C# demo project which would fail on a specific request
2. I could run Fiddler in the middle and see the request succeed
3. I could craft the request inside Fiddler and see it succeed

Once I realized that Fiddler's presence was helping the test succeed (yes, that
Futurama quote "Not fair! You changed the outcome by measuring it!" was staring
me in the face for a while), I put on my debugging hat and drilled down into
the issue using some old-fashioned tracing.

For those of you who haven't been in this situation, here's the
[cheat codes](http://msdn.microsoft.com/en-us/library/ty48b824.aspx) for tracing
the network activity. I get angry at how much configuration in .NET can be
driven from configuration files (hello there WCF), but this is one case where
it was tremendously helpful.

So I dropped this into the `app.config` file for my test application and ran it
again:

```
<configuration>
  <system.diagnostics>
    <sources>
      <source name="System.Net" tracemode="includehex" maxdatasize="1024">
        <listeners>
          <add name="System.Net"/>
        </listeners>
      </source>
      <source name="System.Net.Sockets">
        <listeners>
          <add name="System.Net"/>
        </listeners>
      </source>
      <source name="System.Net.Cache">
        <listeners>
          <add name="System.Net"/>
        </listeners>
      </source>
    </sources>
    <switches>
      <add name="System.Net" value="Verbose"/>
      <add name="System.Net.Sockets" value="Verbose"/>
      <add name="System.Net.Cache" value="Verbose"/>
    </switches>
    <sharedListeners>
      <add name="System.Net"
        type="System.Diagnostics.TextWriterTraceListener"
        initializeData="trace-network.log"
      />
    </sharedListeners>
    <trace autoflush="true"/>
  </system.diagnostics>
</configuration>
```

## Act 2 - What did we find?

So I dig into the generated log file and see what's going on...

```
...
System.Net Information: 0 : [3368] HttpWebRequest#7746814 -
  Request: GET /repos/shiftkey-tester/test/branches/master HTTP/1.1

System.Net Information: 0 : [3368] ConnectStream#59408853 - Sending headers
{
Authorization: Basic {snip}
Host: api.github.com
Connection: Keep-Alive
}.
...
System.Net Information: 0 : [3368] HttpWebRequest#26376641 -
  Request: POST /repos/shiftkey-tester/test/git/blobs HTTP/1.1

System.Net Information: 0 : [3368] ConnectStream#46098163 - Sending headers
{
Authorization: Basic {snip}
Host: api.github.com
Content-Length: 79
Expect: 100-continue
}.
...
System.Net Information: 0 : [3368] HttpWebRequest#40635940 -
  Request: Patch /repos/shiftkey-tester/test/git/refs/heads/master HTTP/1.1

System.Net Information: 0 : [3368] ConnectStream#16678014 - Sending headers
{
Authorization: Basic {snip}
Host: api.github.com
Content-Length: 57
Expect: 100-continue
}.
```

Did you spot the issue? Here's a hint:

```
Patch /repos/shiftkey-tester/test/git/refs/heads/master HTTP/1.1
```

![](https://i.imgur.com/0IE7YeS.gif)

The sample code for this request is just some good old `HttpWebRequest` code:

```
var webRequest = WebRequest.Create(sourceUrl);
webRequest.Method = method;
...
using (var webResponse = webRequest.GetResponse())
{
   // read the response
}
```

`method` is actually just a string (in more modern .NET clients it's an enum
but this is the bed we've chosen to sleep in this time).

So when these requests were being created, they were just strings like this:

```
webRequest.Method = "Post";
webRequest.Method = "Patch";
```

The first method is transformed to `POST` correctly, while the second remains
as `Patch` - and then the bad things happen.

I tested this theory by switching to use [Runscope](https://runscope.com/) as
a proxy to the GitHub API and the same thing happened.

```
System.Net Error: 0 : [3312] Exception in HttpWebRequest#32764015::
  - The server committed a protocol violation. Section=ResponseStatusLine.
System.Net Error: 0 : [3312] Exception in HttpWebRequest#32764015::GetResponse
  - The server committed a protocol violation. Section=ResponseStatusLine.
```

Wait, no, silly me. I got a completely different response from the server.
No content, no status code, just a message about committing a "protocol violation"
 - whatever that means.

## Act 3 - Cleaning Up The Murder Scene

So I'll own some of the blame here for not spotting the casing issue earlier,
but after that I got a bit shouty on Twitter about two things.

### The Client

RFC 2616 Section 5.1.1 states that

> The Method token indicates the method to be performed on the resource
> identified by the Request-URI. The method is case-sensitive.

It would have been nice for the client to give some feedback about using a method
which was not uppercase. The fact that it uppercased one method and not another
I suspect is a bug - you can see it here on
[Connect](https://connect.microsoft.com/VisualStudio/feedback/details/806423/httpwebrequest-uppercases-some-http-actions-and-not-others)
for yourself.

EDIT: Someone pointed me to [RFC 5789](http://tools.ietf.org/html/rfc5789)
where `PATCH` was moved from a custom verb to an official verb - which might
explain the differences in behaviour (RFC 2616 was written in June 1999,
  RFC 5789 in March 2010 - that's forever in Internet time).

I've also thrown the repro up [on GitHub](https://github.com/shiftkey/HttpWebRequest-CaseSensitiveMethods)
- I don't care at this stage to test with the newer HttpClient bits, but I
suspect those work as advertised due to using ~~enums~~ classes over plain strings.

EDIT: so the always-friendly [Darrel Miller](https://darrelmiller.com/) pointed out
that the
[System.Net.Http.HttpMethod](http://msdn.microsoft.com/en-us/library/system.net.http.httpmethod.aspx)
used in HttpClient is actually a class in it's own right (and thus extensible).
### The Server

RFC 2616 Section 5.1.1 also states:

> The return code of the response always notifies the client whether a method
> is currently allowed on a resource, since the set of allowed methods can
> change dynamically. An origin server SHOULD return the status code 405
> (Method Not Allowed) if the method is known by the origin server but not
> allowed for the requested resource, and 501 (Not Implemented) if the method
> is unrecognized or not implemented by the origin server. The methods GET and
> HEAD MUST be supported by all general-purpose servers. All other methods are
> OPTIONAL; however, if the above methods are implemented, they MUST be
> implemented with the same semantics as those specified in section 9.

TL;DR:

 - if your server knows about the method, but it's not allowed?
 `405 Method Not Allowed`
 - if your server does not know about the method? `501 Not Implemented`

Getting served a `502 Bad Gateway` in this case didn't align with the
specification and lead us to research avenues that were unrelated to the
problem at hand.

In both examples above, I got two different results - neither of which aligned
with what the specification indicated.


### Appendix: The Vibe Of The Thing

Specifications are important.

They're what we use to define implementations of systems that are supposed to
interop with eachother. If these things behave in slightly different ways for
the same data, then you're gonna have a bad time.

Times like these remind me that the web is an amazingly fragile thing, and that
all the specifications in the world don't matter if we choose to ignore them.
