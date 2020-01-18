---
layout: post
title: How many files did I add since a given date?
area: Git
---

One of my [resolutions for 2020]({{< ref "post/2020-01-06-some-resolutions.md" >}})
was to write more, and rather than manually checking this I figure I have enough
details in [the repository](https://github.com/shiftkey/brendanforster.com) to
write a tiny Bash script to validate this myself.

I'll walk through how I designed this as it was a fun little exercise to
recall some corners of Git that readers might not be familiar with, but you can
skip to [the end](#wrap) if you want to see the completed script.

## Find the baseline commit

The first thing I knew I needed was a starting point to compare, and it felt
appropriate to find whatever commit was there at January 1, 2020. `git-log`
supports providing start and end date ranges, so we can log all commits on
the default branch before that date.

```shellsession
$ git log master --before="2020-01-01"
commit 92411d5f3798994b3423c8ab5c7aaed2f10bd8ca
Author: Brendan Forster <github@brendanforster.com>
Date:   Mon Sep 9 12:01:59 2019 -0300

    update second post to v2 workflow file

commit fdbc087e2ce411d9d1136ee2b95cac45321d8261
Author: Brendan Forster <github@brendanforster.com>
Date:   Mon Sep 9 11:59:56 2019 -0300

    update first post to v2 workflow file format

commit a258cab485008dc677298c71aab3e085bbe419e9
Author: Brendan Forster <github@brendanforster.com>
Date:   Tue Aug 20 08:56:47 2019 -0300

    added note about v2 and v1 being slightly different
...
```

`git-log` will display all commits that satisfy this criteria, from the latest
through to the first commit in the repository. That's unnecessary for our
situation - we only want the first commit.

Thankfully `git-log` has the `-n` parameter which tells Git to stop logging
after a specific number of commits:

```shellsession
$ git log master --before="2020-01-01" -n 1
commit 92411d5f3798994b3423c8ab5c7aaed2f10bd8ca
Author: Brendan Forster <github@brendanforster.com>
Date:   Mon Sep 9 12:01:59 2019 -0300

    update second post to v2 workflow file
```

Now it's down to one commit, but for the purposes of diff-ing we only need the
commit id.

`git-log` and other commands that show details about the Git repository support
allow you to format the output of Git objects as they're shown to users. This
can be one of the supported aliases, or you can specify your own format using
the tokens that Git defines. I don't want to go over this in too much detail,
because the [documentation](https://git-scm.com/docs/git-log#_pretty_formats)
for `git-log` does a good job of showing the options available and what they
look like.

For this script I only need the commit id, and we can use the shortened version
rather than the full 40 bytes. This is represented by the `%h` placeholder in
the formatting output:

```shellsession
$ git log master --before="2020-01-01" -n 1 --pretty="%h"
92411d5
```

With this available, I can assign the output from the command to a bash variable
for use in later steps of the script.

## Find the current commit

I want to compare the base commit to the latest commit published to the site.
Because this gets deployed whenever I push a new commit to the default branch,
I can use the branch name and `git-rev-parse` to obtain the commit id.

```shellsession
$ git rev-parse master
e349927bf67262e40b16c1ef6fac365a1d2ad617
```

I settled on this command rather than `git-log` because it caters for my "turn a
ref into a commit id" case without needing to traverse the history of the
repository.

Like above, I can also grab the short commit id by providing a `--short`
parameter:

```shellsession
$ git rev-parse --short master
e349927
```

## Compare two commits and find the added files

The simplest way to diff these two commits would be to use the `git-diff`
command:

```shellsession
$ git diff 92411d5 e349927
diff --git a/config.toml b/config.toml
index 92d2484..0ac3759 100644
--- a/config.toml
+++ b/config.toml
@@ -26,3 +26,4 @@ pygmentsUseClasses=true

 [params]
     description = "Just a site about me and what I've been up to"
+
diff --git a/content/notes/maintainer-lessons-proposing-changes.md b/content/notes/maintainer-lessons-proposing-changes.md
new file mode 100644
index 0000000..e22e053
--- /dev/null
+++ b/content/notes/maintainer-lessons-proposing-changes.md
@@ -0,0 +1,226 @@
+---
+layout: post
+title: Maintainer Lessons - Proposing Changes to a Project
+area: Open Source
+---
+
+This article came from some interactions I had back in May last year in an open
+source project. The specifics are unimportant, but I felt were an opportunity to
+share some insight from my experiences as a maintainer.
+
+### Target Audience
```

But we have several problems with this approach:

 - the default output is very verbose - *can we hide the diff and header information?*
 - changes unrelated to posts are included - *can we focus the diff only on specific paths?*
 - all changes are displayed by default - *can we only focus on added files?*

It turns out all of these options are controllable for `git-diff`.

### Don't display the full diff

`git-diff` supports outputting the diff in some common "modes". If you're
interested in looking at the summary of changes, you can include
`--numstat` which shows the lines added and deleted in the diff:

```shellsession
$ git diff --numstat 92411d5 e349927
1       0       config.toml
226     0       content/notes/maintainer-lessons-proposing-changes.md
102     0       content/notes/my-code-review-emoji-guide.md
7       11      content/now/index.md
5       5       content/post/2013-11-24-case-sensitive.md
4       3       content/post/2016-06-26-burnout.md
115     0       content/post/2020-01-06-some-resolutions.md
3       0       layouts/notes/list.html
30      0       layouts/partials/todo.html
6       0       layouts/shortcodes/picture.html
2       2       netlify.toml
-       -       static/images/on-burnout/beach.png
-       -       static/images/on-burnout/beach.webp
-       -       static/images/on-burnout/contributions.png
-       -       static/images/on-burnout/contributions.webp
-       -       static/images/on-burnout/tweets.png
-       -       static/images/on-burnout/tweets.webp
-       -       static/images/review-emoji/suggested-change.png
-       -       static/images/review-emoji/suggested-change.webp
1       1       themes/hugo-natrium-theme/layouts/index.html
10      0       themes/hugo-natrium-theme/static/css/fonts.css
4       0       themes/hugo-natrium-theme/static/css/main.css
```

The `-` entries are for paths that are binary files, as those cannot be diffed
directly by Git.

This is handy to see how files have changed, but as we don't need the stats we
can instead provide `--name-only` to print out just each path for the diff:

```shellsession
$ git diff --name-only 92411d5 e349927 
config.toml
content/notes/maintainer-lessons-proposing-changes.md
content/notes/my-code-review-emoji-guide.md
content/now/index.md
content/post/2013-11-24-case-sensitive.md
content/post/2016-06-26-burnout.md
content/post/2020-01-06-some-resolutions.md
layouts/notes/list.html
layouts/partials/todo.html
layouts/shortcodes/picture.html
netlify.toml
static/images/on-burnout/beach.png
static/images/on-burnout/beach.webp
static/images/on-burnout/contributions.png
static/images/on-burnout/contributions.webp
static/images/on-burnout/tweets.png
static/images/on-burnout/tweets.webp
static/images/review-emoji/suggested-change.png
static/images/review-emoji/suggested-change.webp
themes/hugo-natrium-theme/layouts/index.html
themes/hugo-natrium-theme/static/css/fonts.css
themes/hugo-natrium-theme/static/css/main.css
```

### Only diff for specific paths

It looks like we have a number of unimportant files included in the diff
currently:

 - config changes
 - images added alongside posts
 - theme changes

We want to focus on just the content changes, so let's specify the directory we
want to diff at the end of the command:

```
$ git diff --name-only 92411d5 e349927 -- content/
content/notes/maintainer-lessons-proposing-changes.md
content/notes/my-code-review-emoji-guide.md
content/now/index.md
content/post/2013-11-24-case-sensitive.md
content/post/2016-06-26-burnout.md
content/post/2020-01-06-some-resolutions.md
```

The `content/` path here can also support wildcards based on your configured
shell if you wanted to filter on specific file extensions or a more complex
setup, but for my situation I know `content/` is full of markdown files, so I'm
happy with that.

One note about the `--` separator - this is defined for several Git sub-commands
to represent "end of options" and separate the known arguments for the
sub-command from pathspecs, refs and revisions that users might define in a
repository that could start with `--` (as unlikely as this might be) to avoid
ambiguities and potential security risks with handling these inputs.

### Only diff on added files

Looking at the files shown in that last step, it includes some older posts which
I was tidying up that shouldn't count towards my goal. Let's confirm this by
switching back to `--numstat`:

```
$ git diff --numstat 92411d5 e349927 -- content/
226     0       content/notes/maintainer-lessons-proposing-changes.md
102     0       content/notes/my-code-review-emoji-guide.md
7       11      content/now/index.md
5       5       content/post/2013-11-24-case-sensitive.md
4       3       content/post/2016-06-26-burnout.md
115     0       content/post/2020-01-06-some-resolutions.md
```

Notice we have some entries where the deleted lines (second column) are greater
than zero. That suggests these files have been modified (lines added and
deleted in a file) which existed at the start of 2020.

`git-diff` has a `--diff-filter` parameter that lets you specify which changes
are displayed in the diff. This is helpful if you only wanted to diff specify
types of changes - added, removed, modified, copied, etc, or some combination of
these states.

[The docs](https://git-scm.com/docs/git-diff#Documentation/git-diff.txt---diff-filterACDMRTUXB82308203)
are worth reading here if you're curious, but for now I'm just going to focus on
added files.

```shellsession
$ git diff --numstat --diff-filter=A 92411d5 e349927 -- content/
226     0       content/notes/maintainer-lessons-proposing-changes.md
102     0       content/notes/my-code-review-emoji-guide.md
115     0       content/post/2020-01-06-some-resolutions.md
```

This now excludes all those modified files from earlier, so I think we're in a
good place with this query. Switching back to  `--name-only` and we have our new
posts listed:

```shellsession
$ git diff --name-only --diff-filter=A 92411d5 e349927 -- content/
content/notes/maintainer-lessons-proposing-changes.md
content/notes/my-code-review-emoji-guide.md
content/post/2020-01-06-some-resolutions.md
```

## Caveats

This works great for my case, but I suspect there are some corner cases that 
need to be handled:

 - if you've renamed a file while also changing the content of that file, the
   output here may depend on your rename detection settings. The default for Git
   should be `50%` similarity index, but it could be disabled via config 
   using `git config diff.renames false`.

## Wrap

This is the completed script with comments to explain each step:

```bash
# change this if you have a different default branch
DEFAULT_BRANCH=master
# get the final commit id from 2019
BASE=$(git log $DEFAULT_BRANCH --before="2020-01-01" -n 1 --pretty="%h")
# get the current commit id from the default branch
HEAD=$(git rev-parse --short $DEFAULT_BRANCH)
# list the files added in HEAD that don't exist in in BASE (under a given directory)
FILES=$(git diff $BASE..$HEAD --name-only --diff-filter=A -- content/)
# convert to a count of files and process output
COUNT=$(echo "$FILES" | wc -l | xargs echo)
# pretty print the result
echo "$COUNT files added to blog in 2020"
```

It's been tested on macOS, and for the sake of convenience it depends on `wc`
and `xargs` also being accessible - these should exist in common Linux distros.

Running this up against my local blog (not including this post) shows I've made
some progress for 2020 already against my goal of 20 posts:

```shellsession
$ ./script.sh
3 files added to blog in 2020
```
