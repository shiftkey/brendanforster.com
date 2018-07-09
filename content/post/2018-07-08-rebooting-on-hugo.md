---
title: Spring Cleaning
date: 2018-07-08
summary: This has been limping along for a while, so here's me trying to inject some life into this and address some of the pain points so maybe I'll write some more.
aliases:
  - /2018/06/spring-cleaning/
---

Thanks to [Andrew Best](https://twitter.com/_AndrewB) for sharing [this post](https://www.andrew-best.com/posts/creating-a-personal-website-and-blog-in-an-hour/) earlier today about getting a website setup in an hour (and to [Geoffrey Huntley](https://twitter.com/GeoffreyHuntley) for being Andrew's inspiration), as I've been neglecting this site in a few ways and it was precisely the shot in the arm I needed.

TL;DR:

 - I've been using Jekyll for a while, but I wasn't using GitHub Pages because I wanted to use some syntax highlighing extensions that weren't enabled. There might be a way to do this now, but I'm always stumbling into problems with updating my local Ruby install so revisiting this isn't a priority
 - while GitHub Pages recently rolled out [HTTPS support for custom domains](https://blog.github.com/2018-05-01-github-pages-custom-domains-https/), this site is hosted on Heroku and so I'd need to use their solution (which is still Let's Encrypt-based)
 - it's a static site and was auto-deployed from `master` so it was trivial to push new changes, but running on Heroku required running Rack and having more moving parts than necessary
 - I'd heard about [Netlify](https://www.netlify.com/) but for some reason hadn't connected that I could use it for hosting this blog, and Andrew's post talked about [Hugo](http://gohugo.io/) which I've dabbled with for another project and figured I could port the existing content over (some of it I just want to leave in the past).

Anyway, the new source for this site is available on [GitHub](https://github.com/shiftkey/brendanforster.com) and now supports HTTPS, so that was a couple of hours well spent on a lazy Sunday. I'll likely be tweaking the theme some more as I get settled in and try to write more often.

Some other notes:

 - the use of YAML front matter in Hugo meant I was able to port things over fairly quickly
 - Hugo uses [aliases](https://gohugo.io/content-management/urls/#how-hugo-aliases-work) to handle redirects - each path you specify in the front matter results in a stub page with `http-equiv="refresh"` meta tags to perform the redirect. Not sure how I feel about this approach as serving up the proper `HTTP 30*` response feels More Correct™, and there's probably a way to do it with Netlify if I wanted to push that out of each page, but whatever.
 - I chose a fairly vanilla theme and tweaked the layout a bit, but I'll venture out to something more custom once I figure out what I want to write about and how to organize things
 - I discarded most of the old posts with syntax highlighting, so I'll figure what that experience looks like with an upcoming post.