---
title: Spring Cleaning
date: 2018-06-26T23:00:00-03:00
summary: This has been limping along for a while, so here's me trying to inject some life into this and address some of the pain points so maybe I'll write some more.
---

Thanks to [Andrew Best](https://twitter.com/_AndrewB) for sharing [this post](https://www.andrew-best.com/posts/creating-a-personal-website-and-blog-in-an-hour/) earlier today about getting a website setup in an hour (and to [Geoffrey Huntley](https://twitter.com/GeoffreyHuntley) for being Andrew's inspiration), as I've been neglecting this site in a few ways and it was precisely the shot in the arm I needed.

TL;DR:

 - I've been using Jekyll for a while, but I wasn't using GitHub Pages because I wanted to use some syntax highlighing extensions that weren't enabled. There might be a way to do this now, but I'm always stumbling into problems with updating my local Ruby install so revisiting this isn't a priority
 - while GitHub Pages recently rolled out [HTTPS support for custom domains](https://blog.github.com/2018-05-01-github-pages-custom-domains-https/), this site is hosted on Heroku and so I'd need to use their solution (which is still Let's Encrypt-based)
 - it's a static site, but running on Heroku required running Rack and having more moving parts than necessary
 - I'd heard about [Netlify]() but for some reason hadn't connected that I could use it for hosting this blog, and Andrew's post talked about [Hugo](http://gohugo.io/) which I've dabbled with and figured I could port the existing content over (some of it I just want to leave in the past).

