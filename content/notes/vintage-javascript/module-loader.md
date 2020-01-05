---
layout: post
title: Introducing a module loader
area: Vintage JavaScript
---

With [automated deployments](/notes/netlify-integration-with-github-pull-requests/)
in place the next step on our journey was to figure out a better way to manage
our scripts. At the time we explicitly listed these dependencies in the footer
of the HTML file:

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

## Integrate `requirejs` as a module loader

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

## Requiring external scripts

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

## Test `ProjectsService` in NodeJS

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
