---
layout: post
title: My Code Review Emoji Guide
area: Open Source
---

I do a lot of reviews of pull requests on GitHub, and as soon as they require
more than a single comment I try and prefix the comments with an emoji
character to help categorize each bit of feedback.

Here's [an example](https://github.com/desktop/desktop/pull/7635#discussion_r287833836)
of how it looks in action:

<picture>
  <source type="image/webp" srcset="/images/review-emoji/suggested-change.webp">
  <source type="image/png" srcset="/images/review-emoji/suggested-change.png">
  <img src="/images/review-emoji/suggested-change.png" class="align-center" />
</picture>

I'll save the details and my opinions about code review itself for another post,
as I think this guide can stand on it's own.

I prefix comments with an emoji for a few reasons:

 - Feedback can fall into a number of different categories, and not everything
   should be addressed immediately (or in the same way)
 - Sometimes a piece of feedback doesn't need to be addressed immediately before
   a pull request is merged, but it should be captured somewhere. Once we agree
   its something we should address, and figure out how/when to address it, we can
   move the feedback to a separate tracking issue for future work and merge the
   pull request
 - I believe visual cues help the recipient to navigate the feedback of a
   thorough review containing many comments without being overwhelmed

Here's the current set I use, and what each thing represents:

 - 🎨 `:art:`
    - **Context:** this is a suggestion to improve the code, that shouldn't
      affect the functional aspect but would help with readability/legibility
    - **Action:** contributor may want to accept the change (yay
      [suggested changes](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/commenting-on-a-pull-request#adding-line-comments-to-a-pull-request)),
      may provide extra context about why they're not interested in making the
      change, or may just dismiss the suggestion
    - **Status:** not a blocker on merging
 - 💭 `:thought_balloon:`
    - **Context:** there's something about this change that I'm thinking about,
      and it's not currently clear to me whether something needs to change here
    - **Action:** input about the questions or ideas proposed, and whether there
      is extra relevant information that I've overlooked
    - **Status:** currently not a blocker on merging, pending further discussion
 - ❓ `:question:`
    - **Context:** I'm not clear about something and why it's required
    - **Action:** clarifying remarks from the contributor
    - **Status:** tentatively a blocker on merging, pending more context
 - 🚨 `:rotating_light:`
    - **Context:** I'm a bit nervous about this change and what it might
      introduce after merging, and would like to better understand why it is
      needed
    - **Action:** discussion between contributor and reviewers
    - **Status:** blocker on merging, pending further discussion

### What does "blocker" mean?

With teams that are working asynchronously across timezones, it's often hard to
communicate which feedback that is considered blocking versus non-blocking:

 - **blocking:** feedback must be resolved before merging
 - **non-blocking:** optional feedback but pull request is fine to merge as-is

Sometimes you can submit a quick review with "Request changes" selected to block
a pull request from being merged, but the teams I've been a member of in the
past prefer to only use this in case of an emergency (overriding a previous
approval), as we will protect the default branch to require an approval from
someone else on the team from merging.

On teams containing a mix of senior and junior developers being able to provide
detailed feedback in a review and indicate what is blocking feedback, without
the "Request changes" end result, allows the contributor a chance to own their
review process, discuss things with the reviewers and hopefully feel more in
control of the situation.

### What about proposing larger changes to the code?

You may notice there's no item here for proposing significant changes to the
pull request, and that's a conscious decision for a few reasons:

 - as a reviewer I should be helping guide the contributor rather than getting
   too much into the implementation details
 - if there are significant things to address, it's on me as the reviewer to
   sync up and help the contributor figure out a way forward before too much
   time and effort is spent on the pull request
 - if the contributor gets stuck on something, pairing (physically or virtually)
   is an option here to unblock
 - if there is an opportunity to improve that's not necessarily in scope for the
   pull request, we should confirm it's worth spending time on, identify when it
   should be addressed, and write it up (e.g. as a new issue)

There's definitely a skill to balancing the quality of the codebase versus
empowering the contributor to achieve what they are working towards, but I think
constraining the reviewer to focus on feedback as much as possible helps to keep
things moving for asynchronous teams.
