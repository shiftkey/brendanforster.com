---
title: "Displaying SQL queries in Rails console"
date: 2020-03-16
area: Rails
---

I keep using this command but also forgetting it when working in the Rails
console so I'm publishing this here to ensure it's easily accessible:

```rb
ActiveRecord::Base.logger = Logger.new(STDOUT)
```

What this does is ensure that any SQL commands that are performed with
ActiveRecords are emitted to the console. This helped me out a bunch of times
recently as I was implementing a feature and needed to see what database queries
resulted from each ActiveRecord call.

There might be a way to ensure this is always run when you, like [this post](https://medium.com/@sulabhjain/how-to-extend-rails-console-on-initialization-14ac79011332) but
I'll look into that another day.
