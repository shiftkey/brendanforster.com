---
layout: post
title: Making working JavaScript more modular without rewriting the world
area: JavaScript
---

This post is a series of experiments and explorations about how to bring some
personal sanity to an existing JS codebase that I need to support. I've had some
initial wins, so I figure this is a good chance to write up some notes and share
what I'm up against and where I'm heading.

## The History

[Up for Grabs](https://up-for-grabs.net/) is a little codebase that started in
2013 and grew organically without us really thinking about it. Since that time
we've seen a lot of change in the JavaScript ecosystem:

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
   BabelJS, a full environment for using newer JavaScript syntax but
   transpiling the code in a way that was backwards-compatible to the user
   running the code.
 - EcmaScript 6 was formalized in 2015, and the TC39 technical committee working
   on standardizing the JavaScript language has continued to introduce updates
   to this annually - ES2016, ES2017 and so on

This barely scratches the surface of all the things that have influenced modern
JavaScript development, but all of these topics are relevant to our little
project.

We're also limited in some way from being able to adopt new things:

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

Armed with those goals, here's what we've done since.

### Automate deployments of pull requests to Netlify

Netlify is a great product for deploying websites like ours, and their GitHub
integration means that each PR that's opened against the project gets a public
URL that you can view and verify the changes with.

This has been a huge timesaver for reviewers because they don't need to build
and verify the changes locally - they can just browse to the Preview Deploy URL
instead.

### Find a module loader for the web

With automated deployments in place the next step on our journey was to figure
out a better way to manage our scripts. At the time we explicitly listed these
dependencies in the footer of the HTML file:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/underscore.js/1.5.2/underscore-min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/sammy.js/0.7.4/sammy.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/chosen/1.0/chosen.jquery.min.js"></script>
<script src="{{ site.github.url }}/javascripts/projectsService.js"></script>
<script src="{{ site.github.url }}/javascripts/main.js"></script>
```

This works fine, but has some downsides for maintenance:

 - `script` tag loading blocks on parsing until the script is executed
 - we're mixing together external scripts as well as application scripts
 - scripts that are isolated like this need to use the global JS object to share
   things between them - this makes them harder to test
 - we need to be careful about how we'd add new scripts here - have we got them
   in the right order?

I really enjoy the experience of NodeJS and how it's module system works:

 - assign functions to `module.exports` to export them
 - `var something = require('./some-module');` to use them in your code

But CommonJS (the spec that NodeJS and others have implemented) was never
implemented in the browser for a few reasons. The [RequireJS](https://requirejs.org/docs/why.html) docs and [this blog post](http://tagneto.blogspot.com/2010/03/commonjs-module-trade-offs.html) talk about the limitations of CommonJS, and I
won't rehash them here.

So I went exploring for a module loader for the web and ended up with [Asynchronous Module Definitions](https://requirejs.org/docs/whyamd.html), which have a simple
pattern:

 - the script defines the modules it needs to work
 - the callback is invoked when those modules are available, and the return
   value of the function can be used elsewhere

This is a simple example from the AMD documentation for something that needs
jQuery to work:

```js
define(['jquery'] , function ($) {
    return function () {};
});
```

### Integrate `requirejs` as a module loader

To use AMD in your web application, you need to do three things:

 - import the [`require.js`](https://requirejs.org) script on your page, and
   include a `data-main` attribute indicating where the JS setup file lives for
   your site

```html
<script src="{{ site.github.url }}javascripts/lib/require.js" data-main="javascripts/app"></script>
```

  - setup your config in the JS setup file from the previous step. Here's a
    minimal version for the Up for Grabs site. This won't work as-is, for
    reasons I'll explain later

```js
requirejs.config({
  baseUrl: "javascripts"
});
```

 - after invoking `requirejs.config` with the required config, load the module
   that is your entry point to the rest of the application code:

```js
requirejs(["main"]);
```

**Note:** Yes, I've mixed up `app` and `main` here in a potentially confusing
way. I might fix that in the future.

#### Requiring external scripts

Remember that list of external modules listed as `script` tags before our
application code? We can move those into the requirejs config so our module
loader handles loading and caching:

```js
requirejs.config({
  baseUrl: "javascripts",
  paths: {
    underscore: "//cdnjs.cloudflare.com/ajax/libs/underscore.js/1.9.1/underscore-min",
    jquery: "//cdnjs.cloudflare.com/ajax/libs/jquery/1.12.4/jquery.min",
    sammy: "//cdnjs.cloudflare.com/ajax/libs/sammy.js/0.7.6/sammy.min",
    chosen: "//cdnjs.cloudflare.com/ajax/libs/chosen/1.8.7/chosen.jquery.min",
  }
});
```

You'll note that I've dropped the `http/s` prefix and `js` suffix on each file
here - the former is so that the browser will use the same protocol as what the
website is using, so I don't have to manually do this myself.

_I honestly can't recall the reason for the latter right now._

These scripts all work with AMD, but my application code does not, so these
files need to be updated to work with AMD too:

```html
<script src="{{ site.github.url }}/javascripts/projectsService.js"></script>
<script src="{{ site.github.url }}/javascripts/main.js"></script>
```

For `projectsService.js`, it was written as an [Immediately Invoked Function Expression](https://developer.mozilla.org/en-US/docs/Glossary/IIFE) and attaching itself to the `window` global:

```js
(function(host, _) {
  // code goes here...

  host.ProjectsService = ProjectsService;
})(window, _);
```

This was easy to refactor into a module that required `underscore` and returned
the `ProjectsService` function:

```js
define(["underscore"], function(_) {
  // code goes here...

  return ProjectsService;
});
```

The `main.js` script was a bit more complicated, despite looking like it only
required jQuery:

```js
(function($) {

})(jQuery);
```

From experience I knew the `main.js` script depended on all four external
libraries, as well as `projectsService.js`. The way to ensure all of these are
loaded before the main script runs is to list them as dependencies:

```js
define([
  "jquery",
  "projectsService",
  "underscore",
  "sammy",
  // chosen is listed here as a dependency because it's used from a jQuery
  // selector, and needs to be ready before this code runs
  "chosen",
], ($, ProjectsService, _, sammy) => {
  // ...
});
```

There were some smaller changes to make during this process, but that was
related to our usage of third-party libraries that was impacted by the module
loader changes. Aside from that it was a pretty smooth transition.

### Test `projectsService` in NodeJS

As the [RequireJS docs](https://requirejs.org/docs/node.html) mention you need
to insert an AMD loader (`amdefine` was the recommendation) at the start of the
script if you wish to load an AMD module within a Node script:

```js
// required for loading into a NodeJS context
if (typeof define !== "function") {
  var define = require("amdefine")(module);
}

define(["underscore"], function(_) {
  // code goes here...

  return ProjectsService;
});
```

This affects the naming of dependencies in certain ways:

 - unless you also provide your `requirejs.config` setup in your tests, the
   default behaviour in the Node context is to load a module named after your
   dependency

I recommend naming third-party dependencies to match how they're published on
NPM, so that you can leverage those dependencies in a test runner directly.

There's probably some side-effects I'll uncover here as I go further with this,
but for now it's working great.

