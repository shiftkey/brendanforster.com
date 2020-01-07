---
layout: post
title: Maintainer Lessons - Proposing Changes to a Project
area: Open Source
---

This article came from some interactions I had back in May last year in an open
source project. The specifics are unimportant, but I felt were an opportunity to
share some insight from my experiences as a maintainer.

### Target Audience

The audience for this article is mostly people who are new contributors to open
source, who aren't as familiar with the "tribal knowledge" or unspoken customs
that occur in open source communities. The first interactions are often the most
important, and I want to help new contributors navigate their way through this
world and hopefully get some rewards out of their experiences with open source.

## So what happened?

I'm omitting any names and details from this because I don't fault the person on
the other side of this. I want to use this as a learning experience for those
who are new to open source communities.

### Timeline

The timeline of events for one of the projects I maintain went like this:

- a new contributor sent a pull request to a project I maintain to add a bunch
  of changes, along with a brief summary about the change
- I get a notification, which falls into the endless pile of notifications I
  have to manage
- 10 days later, I finally get a chance to look at it properly
- I close the pull request with a message explaining why I didn't feel it was a
  suitable change
- 5 hours later, I get an email from someone (who I believe was the pull request
  author, I'm not 100% sure) asking to re-evaluate my decision

At first glance this might not seem so interesting, but the email itself talked
about a bunch of things that were fascinating to hear after-the-fact.

Here were the key points of the email:

> We need this change in before [our launch]

Deadlines. I've got them. You've got them. Everyone has them, even the ones they
choose to ignore until you feel them \*whoosh\* past. But we're talking about a
deadline that was several months away, and wasn't mentioned in the pull request
itself.

> We already reviewed the changes internally [in $company].

This is also great to hear, but this is news to me again. How was it reviewed?
How did you come to make the various decisions made?

It's great to hear large companies using the things I work on, and it's great to
see them engage more directly with the project - either through contributions,
feedback or other sorts of participation. In this situation there were some
differences between how I was maintaining the project and what was proposed
in the pull request. I tried to make those decisions clear when I closed the
pull request, but maybe I can do better in the future with some documentation
improvements.

> [The teams] are waiting for [the pull request to be merged] to pull their
changes.

You may think my review is a blocker on your job, but it shouldn't be. There
are plenty of ways to be able to work with your changes independent of the pull
request review and result. I'll expand on this later.

> Our telemetry [shows some interesting insight about our audience].

This is great and all but again, it wasn't mentioned in the pull request. I'm
making decisions with the best information I have on hand at the time.

### The Aftermath

The contents of the email left me bemused, exasperated and a bit annoyed. It got
to me enough that I couldn't put it aside and focus on other things, as it
contained plenty of information that would have been helpful while I was
reviewing the change.

I made a joke on Twitter about not being a mind-reader, and then proceeded to
unpack this whole interaction as a Twitter thread as it was a great opportunity
for me to think more about it. This draft has been sitting on my machine for
several months, so I'm glad [Mark Gravell](https://twitter.com/marcgravell)
wrote a blog post about a related topic recently that reminded me of this.

## Proposing a change to a project

So you've been working with a project but it would be so much better with just a
small change to some feature. You've got the code written and it works great for
you, and you also think it's something other people would benefit from. Before
you go and open that pull request, here are some things to think about.

### Who are you and how did you find this project?

After working in open source projects for many years it all kind of blurs
together, and I've come across countless pull requests with just a basic title.
Or an incorrect title. Or a minimal description. Or no description at all.

It's pretty easy to take a couple of minutes here and boost your chances of
landing the change by writing a good introduction to the pull request. Even if
you're not new to open source, or are already a contributor, providing context
behind the change will be very valuable in the future.

If this is the first time you're submitting a pull request to a project (or
anywhere for that matter), introduce yourself. Tell the maintainer about how
you came across the project, how you've gotten use out of their work, and where
the inspiration for your proposal came from.

This might seem like bragging a bit, but it's a great help to the maintainer
to understand the context behind a change. As much as people say "the code is
all that matter", that's most often the easy part. The motivation for a change
is what's really important here - elaborating on the problem, how the proposed
solution came to be, what alternatives were tried and discarded, what unknowns
there are about adopting the solution. All of these things matter and help with
reviewing the change.

### You're selling something, even if you don't believe you are

Once you're setup to contribute to a project, submitting a pull request is
hopefully a straight-forward task. Make some commts, add some comments, push,
create the pull request, voila. But consider things from the other side - you're
asking someone else to accept your changes into their project.

This is not a small thing. Open source projects of all sizes have to deal with
supporting their current userbase, and your change is something on top of that.
When you're proposing a change, make sure you think about how it benefits the
project.

 - bugfixes are easy to propose, but if the changes required are complex it
   might take a bit of discussion to ensure everyone is fine with the changes
   and that they don't introduce any unexpected side-effects elsewhere
 - documentation improvements are very underrated, and as long as they are
   applied in the right spot (some documentation is generated using tooling)
   and that a find-and-replace found all the places that needed to be changed,
   these can be much easier to approve
 - changes to APIs should be thoughtful and deliberate. Was this API already in
   use? How will this change impact users of the library?

### You may think it's minor, but it may be more complicated

Change is almost certain with any sort of software artifact, and evolving the
software to add features, address bugs, improve performance or make it more
flexible are all things that maintainers should strive for. But change also need
to be managed while keeping the software in a working state. This is typically
done by using tests to verify the behaviour of the software, but there are many
cases where tests aren't enough of a confidence net and need to fall back to a
manual review.

This is a topic that I don't want to dig too deep into as it's already a
lengthy post, but some cases I can think of that are relevant to
a maintainer of a project that go beyond tests:

 - a bugfix that requires changing lots of code - Is there a smaller version of
   the bugfix that can be proposed? Are there architectural changes or test
   coverage improvements that can help give confidence that we haven't broken
   the software?
 - adding functionality - it could be something that they don't see as valuable
   to include for everyone, something they feel requires a lot of work from them
   to maintain, or something they don't believe fits with the existing
   functionality provided by the software
 - a behaviour change to the software that's already in use could affect
   existing users - What's the benefit of this new behaviour over the old one?
   Are there risks for others who are relying on the old behaviour? How should
   users of the library handle this change when upgrading?

There are plenty of ways to manage changes in software, and the maintainer
probably has some opinions on how they want to handle this, so you may need to
work within their situation if you want to see your changes merged.

### This might not have a happy ending

These are a lot of words for me to essentially detail my feeling that **software
development is more of a human process than it is a technical process**. The
code is often the easy part, and while working software is the end result it's
still very much reliant on all the work of humans collaborating on the software
to achieve this result that we easily forget about.

I've had my share of discussions and contributions to projects declined, and
often these situations are learning experiences for me:

 - by sharing my insights and interacting with the maintainer I get a better
   understanding about the software that isn't often covered in documentation
 - proposing contributions in a public forum allows others to see the work,
   and potentially learn things themselves
 - collaborating with the maintainer and other reviewers about a proposed
   contribution could lead to a better solution for my problem, even if it
   doesn't get accepted by tha maintainer

## So it didn't work out...

In the original Twitter thread I proposed several different options for being
able to continue to use your changes in the event they were declined by the
maintainer.

*Please confirm the license on the project permits these, as I cannot guarantee
every project is using a sufficiently-permissive license.*

 - for source code you are consuming directly, forking the repository to a
   location you control and linking that as a submodule into your project means
   that you can continue to use your customizations while the upstream project evolves
 - if the repository is too complex to use as a submodule in the main project,
   building the source with the patched changes in version control and then
   using the compiled binaries is also an option. [This guide](https://mijingo.com/blog/creating-and-applying-patch-files-in-git)
   on creating and applying patch files is a good introduction for working with
   Git repositories
 - if the software is distributed via a package manager, you may be able to fork
   the package and publish it under a namespace you control. NPM supports [scopes](https://docs.npmjs.com/about-scopes)
   as a first-class concept, and other package managers may support unofficial
   ways to create branded namespaces. This is a simpler than working with the
   source directly as you can use typical development tools to manage
   dependencies, and it becomes easier for others to also consume your version
   of the project.

Each option on this list may look unappealing to you because it feels like more
work, and that's because **maintenance of open source projects is an ongoing
burden**, and the pain is acute for small or solo projects that accidentally get
successful and find themselves with a lot of users.

We can automate things to save time on repetitive tasks like updating
dependencies and testing proposed changes, but most of the time I believe there
needs to be a human involved to ensure the changes move the project in the right
direction.
