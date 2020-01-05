---
layout: post
title: Making vintage JavaScript more modular (without throwing it out)
area: Vintage JavaScript
slug: vintage-javascript-introduction
---

This post explores how I brought some personal sanity to an existing JS codebase
which ended up going in a different direction to what you might have expected.

## The History

[Up for Grabs](https://up-for-grabs.net/) is a little codebase that started in
2013 and grew organically without any real thought or organizing. Since that
time we've seen a lot of change in the JavaScript ecosystem:

 - NodeJS started to get people's attention as a way of building apps using the
   same language on the server as on the client. io.js forked from NodeJS in
   2014, but was merged back into NodeJS in 2015 with the introduction of the
   NodeJS foundation
 - NPM because the de-facto way to publish and consume JavaScript libraries,
   using CommonJS as a format to synchronously load modules. There are some
   other formats out there (AMD, UMD, ESModules) which may be supported
   depending on how popular the library is
 - JavaScript developers moved away from thinking about minifying their
   JavaScript and towards **bundling** because as the complexity of their
   application code increased. Tools like Browserify, Rollup and Webpack started
   to address this need, and there was much rejoicing.
 - A project named **5to6** was started in 2014 to support upgrading existing
   JavaScript from EcmaScript 5 to EcmaScript 6. This later evolved into
   **BabelJS**, a full environment for using newer JavaScript syntax but
   transpiling the code in a way that was backwards-compatible to the user
   running the code.
 - EcmaScript 6 was formalized in 2015, and the TC39 technical committee working
   on standardizing the JavaScript language has continued to introduce updates
   to this annually - ES2016, ES2017 and so on.

This barely scratches the surface of all the things that have influenced modern
JavaScript development, but each topic is relevant to our little project.

We're also limited in some ways that most developers may not be aware of:

 - this is a static site generator and doesn't provide hooks to perform bundling
   of our JavaScript assets (or anything similar)
 - modern browsers have improved their support of standards since 2013, but I'm
   still not ready to block certain user agents unless absolutely necessary
 - we've learned better ways to manage JavaScript as our applications have
   become more complex, but we need to refactor our current code to be able to
   work in this way

## Modern JS without a fancy toolchain

After spending the previous couple of years full-time a Node + TypeScript +
React project, stepping back into this codebase was jarring. I definitely missed
a lot of the things that came with modern web development, and I was curious
how much I could port back to Up for Grabs without having to rewrite large
chunks of it.

Here are my goals:

 - **incremental improvements** - be able to continually ship changes, and have the
   site stay working throughout the process
 - **modularity** - we want to move away from having a monolithic script for the
   logic, and instead compose the app together at runtime from a number of
   modules that are well-maintained
 - **automated tests** - there are some annoying bugs that we've regressed because
   of a lack of test coverage - can we prevent these from reaching users?
 - **good local development experience** - you should be able make a change to a
   file, reload the site and see the changes immediately - no in-between steps
   to transpile to bundle things

Armed with those goals, here's what we've done since:

 - [Enable Netlify Deploy Previews on GitHub PRs](/notes/netlify-integration-with-github-pull-requests/)
 - [Introducing a module loader](/notes/introducing-a-module-loader/)

