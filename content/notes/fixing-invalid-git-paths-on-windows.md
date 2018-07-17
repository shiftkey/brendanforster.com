---
layout: post
title: Fixing Invalid Git Paths on Windows
date: 2018-07-17
---

One of the _fun_ things about working with Git on Windows is that you're often
at the mercy of the filesystem underneath you, and this crops up in some
entertaining ways:

 - even though NTFS supports running as a case sensitive filesystem, the Win32
   subsystem does not support filenames that differ by case sensitivty only
 - some characters are not permitted by Win32 subsystem APIs - and I refer back to
   [this link](https://docs.microsoft.com/en-us/windows/desktop/fileio/naming-a-file)
   regularly to remind myself of this.

Even in the age of [running Linux on Windows](https://docs.microsoft.com/en-us/windows/wsl/install-win10),
these compatibility issues are not going away any time soon, and taking a
repository and running it on Windows can be problematic, so the rest of this
post is about repairing these repositories so they are not tied to a particular
filesystem.

#### Note

 - All of these operations are being run inside Git Bash that ships with Git
   for Windows - if you are trying to perform the same actions in a different
   shell then they're likely to fail.
 - These instructions are tailored to one repository. You'll need to adapt
   these based on the problem file paths you encounter in your situation.
 - This repository should be fixed soon, so you probably be able to reproduce
   the same behaviour by literally running the same commands.

## The Scene - A Failing Clone

You might have stumbled upon an error like this when cloning down someone else's
repository:

```sh
$ git clone https://github.com/MCanterel/Dual_Brains
Cloning into 'Dual_Brains'...
remote: Counting objects: 12023, done.
remote: Compressing objects: 100% (19/19), done.
remote: Total 12023 (delta 9), reused 14 (delta 4), pack-reused 12000
Receiving objects: 100% (12023/12023), 56.77 MiB | 6.63 MiB/s, done.
Resolving deltas: 100% (5891/5891), done.
error: unable to create file test-files/SavedData\OpenBCI-RAW-aaron+eva.txt: No such file or directory
error: unable to create file test-files/SavedData\OpenBCI-RAW-friday_test.txt: No such file or directory
Checking out files: 100% (9136/9136), done.
fatal: unable to checkout working tree
warning: Clone succeeded, but checkout failed.
You can inspect what was checked out with 'git status'
and retry the checkout with 'git checkout -f HEAD'
```

So here is where Git is actually falling over:

```
error: unable to create file test-files/SavedData\OpenBCI-RAW-aaron+eva.txt: No such file or directory
error: unable to create file test-files/SavedData\OpenBCI-RAW-friday_test.txt: No such file or directory
```

Did you read [the link](https://docs.microsoft.com/en-us/windows/desktop/fileio/naming-a-file)
above about valid file path characters? I'll quote the relevant part here:

> Use any character in the current code page for a name, including Unicode characters and characters in the extended character set (128–255), except for the following:
>
>The following reserved characters:
>
> - < (less than)
> - > (greater than)
> - : (colon)
> - " (double quote)
> - / (forward slash)
> - \ (backslash)
> - | (vertical bar or pipe)
> - ? (question mark)
> - * (asterisk)


So we can't use `\` as a path in Windows, but we have a repository that has some
existing content using this character. That's what we need to fix here.

At this point you _could_ look inside the newly created repository on disk and
try to salvage things. I wouldn't recommend this approach because of this error
message further down.

```
fatal: unable to checkout working tree
```

The working tree is the files on disk associated with the current branch (either
the default branch or whatever branch you wanted to initially checkout). And
with that in an unknown state, there is only pain ahead. Let's try a different
way.

## Part 1 - Fix The Problem Checkout

Because `git clone` fails I instead went and emulated the clone operation:

```
$ git init test-repo
$ cd test-repo
$ git remote add origin https://github.com/MCanterel/Dual_Brains -f
$ git checkout origin/master -f
```

This gets me to a usable repository state, but I still see these problem files
that need addressing:

```
$ git checkout origin/master -f
error: unable to create file test-files/SavedData\OpenBCI-RAW-aaron+eva.txt: No such file or directory
error: unable to create file test-files/SavedData\OpenBCI-RAW-friday_test.txt: No such file or directory
Checking out files: 100% (9136/9136), done.
Note: checking out 'origin/master'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b <new-branch-name>

HEAD is now at 0cbfb99 Merge remote-tracking branch 'refs/remotes/upstream/master'
```

So I'm going to just remove those so I have _something_ to work with:

```
$ git status
On branch fix-path-issue-with-invalid-files
Changes not staged for commit:
  (use "git add/rm <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

        deleted:    "test-files/SavedData\\OpenBCI-RAW-aaron+eva.txt"
        deleted:    "test-files/SavedData\\OpenBCI-RAW-friday_test.txt"

no changes added to commit (use "git add" and/or "git commit -a")

$ git add .

$ git status
On branch fix-path-issue-with-invalid-files
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        deleted:    "test-files/SavedData\\OpenBCI-RAW-aaron+eva.txt"
        deleted:    "test-files/SavedData\\OpenBCI-RAW-friday_test.txt"

$ git commit -m "removed files with invalid paths"
[fix-path-issue-with-invalid-files 726d35a] removed files with invalid paths
 2 files changed, 131880 deletions(-)
 delete mode 100644 "test-files/SavedData\\OpenBCI-RAW-aaron+eva.txt"
 delete mode 100644 "test-files/SavedData\\OpenBCI-RAW-friday_test.txt"
```

So we now have no problems checking out this branch, but we did this by removing
the paths from the repository. If we want to be able to access those files
in the repository we have more work to do.

## Part 2 - Put The Files Back

Git keeps a history of all the file contents for all commits in it's `.git`
folder, and we can explore this using various Git plumbing commands.

We're going to use `git ls-tree` to read the tree contents of a given hash and
find the original contents of the files we just removed.

```
$ git ls-tree origin/master
100644 blob 4fda0922d502719782c70810a6d5ddb1c9bdd14d    .gitignore
040000 tree b39c2cabf1b81dba6e7b625ffa2d9e4535e7421f    Dual_Brains_Visualization
100644 blob fa4410c60ac7d6becebdbe43f5560976ae1d207b    README.md
040000 tree 8974c3cb0ae7f8bc05fc891cf481d58235ae5c83    aaron_test_data
040000 tree 68d65ff67f213859f124ed492428eebdb7a5d1ec    drafts
040000 tree 0ca5bae4b0173c9b2f47893136a6d0b299006378    images
040000 tree 7fe38978bb6d446ee4ac3d68a2a3ff006685c8ed    python
100644 blob ca9d99d426f059fdf0c377552d196a22ae2ff47a    requirements.txt
040000 tree e2a3b7648b57746e68ced4f881f193df05a2341f    test-files
```

Git stores each commit as a tree, and a tree can contain blobs (the file
contents) or other trees. Each entry in a tree also needs a mode (for representing
permissions) and a path. If you squint at this, it's not unlike a filesystem
representation as a data structure.

The two problem file paths are under `test-files` so we'll use the `e2a3b7648b57746e68ced4f881f193df05a2341f`
hash from the previous command and run `git ls-tree` again:

```
$ git ls-tree e2a3b7648b57746e68ced4f881f193df05a2341f
040000 tree 35266afd270f2cf429dd753ab695aacf03ee68d8    Filtered_Data
040000 tree 97ab85ca7728e41e080fe9a9128227d2828108d9    RAW_data_only
040000 tree f55abe3ef151d7209c6cce61878823dc6b2f8801    RAW_output
100644 blob 5020200c10e545c19002f4878c2b6686035f10b0    "SavedData\\OpenBCI-RAW-aaron+eva.txt"
100644 blob 0680d8432628eb071aa2c643f11fe8984df6e972    "SavedData\\OpenBCI-RAW-friday_test.txt"
100644 blob becc5ab499fb0954c72026fdd78c3517694f5ffa    eeg_filter.m
```

Excellent, we can see the problem file paths and they correlate to two blobs `5020200c10e545c19002f4878c2b6686035f10b0` and `0680d8432628eb071aa2c643f11fe8984df6e972`. These identifiers can be used to find the file contents.

The next step is to read out the contents of those blobs and write them to new
files. I'm going to also create the `SavedData` directory name again because I
think  that was the intent here and someone just used the wrong slash and got
away with it.

```
$ mkdir test-files/SavedData/

$ git cat-file -p 5020200c10e545c19002f4878c2b6686035f10b0 > test-files/SavedData/OpenBCI-RAW-aaron+eva.txt

$ git cat-file -p 0680d8432628eb071aa2c643f11fe8984df6e972 > test-files/SavedData/OpenBCI-RAW-friday_test.txt
```

Now we have two new files to commit:

```
$ git status
On branch fix-path-issue-with-invalid-files
Untracked files:
  (use "git add <file>..." to include in what will be committed)

        test-files/SavedData/

nothing added to commit but untracked files present (use "git add" to track)

$ git add .
warning: LF will be replaced by CRLF in test-files/SavedData/OpenBCI-RAW-aaron+eva.txt.
The file will have its original line endings in your working directory.
gwarning: LF will be replaced by CRLF in test-files/SavedData/OpenBCI-RAW-friday_test.txt.
The file will have its original line endings in your working directory.
```

I'm not worried about the line ending warning here, and I can confirm this later
on.

Now we can commit these files to the new path:

```
$ git status
On branch fix-path-issue-with-invalid-files
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

        new file:   test-files/SavedData/OpenBCI-RAW-aaron+eva.txt
        new file:   test-files/SavedData/OpenBCI-RAW-friday_test.txt


$ git commit -m "add the new files with the correct paths"
[fix-path-issue-with-invalid-files d2a39b6] add the new files with the correct paths
 2 files changed, 131880 insertions(+)
 create mode 100644 test-files/SavedData/OpenBCI-RAW-aaron+eva.txt
 create mode 100644 test-files/SavedData/OpenBCI-RAW-friday_test.txt
```

Now we have a branch that can be cloned on Windows safely.

## Part 3 - Verification

To confirm that line ending warning was a non-issue, I can run the same `git ls-tree`
command from earlier and confirm the new blobs are unchanged:

```
$ git ls-tree HEAD
100644 blob 4fda0922d502719782c70810a6d5ddb1c9bdd14d    .gitignore
040000 tree b39c2cabf1b81dba6e7b625ffa2d9e4535e7421f    Dual_Brains_Visualization
100644 blob fa4410c60ac7d6becebdbe43f5560976ae1d207b    README.md
040000 tree 8974c3cb0ae7f8bc05fc891cf481d58235ae5c83    aaron_test_data
040000 tree 68d65ff67f213859f124ed492428eebdb7a5d1ec    drafts
040000 tree 0ca5bae4b0173c9b2f47893136a6d0b299006378    images
040000 tree 7fe38978bb6d446ee4ac3d68a2a3ff006685c8ed    python
100644 blob ca9d99d426f059fdf0c377552d196a22ae2ff47a    requirements.txt
040000 tree 7544e5996262a7fc5017f182790550ab51f92429    test-files

$ git ls-tree 7544e5996262a7fc5017f182790550ab51f92429
040000 tree 35266afd270f2cf429dd753ab695aacf03ee68d8    Filtered_Data
040000 tree 97ab85ca7728e41e080fe9a9128227d2828108d9    RAW_data_only
040000 tree f55abe3ef151d7209c6cce61878823dc6b2f8801    RAW_output
040000 tree 8974c3cb0ae7f8bc05fc891cf481d58235ae5c83    SavedData
100644 blob becc5ab499fb0954c72026fdd78c3517694f5ffa    eeg_filter.m

$ git ls-tree 8974c3cb0ae7f8bc05fc891cf481d58235ae5c83
100644 blob 5020200c10e545c19002f4878c2b6686035f10b0    OpenBCI-RAW-aaron+eva.txt
100644 blob 0680d8432628eb071aa2c643f11fe8984df6e972    OpenBCI-RAW-friday_test.txt
```

The two files are at the expected new location but still have the old blob hash.

To confirm the clone has been fixed, I cloned down my fork and ask it to
checkout this branch:

```
$ git clone -b fix-path-issue-with-invalid-files https://github.com/shiftkey/Dual_Brains my-fork
Cloning into 'my-fork'...
remote: Counting objects: 12028, done.
remote: Compressing objects: 100% (23/23), done.
remote: Total 12028 (delta 11), reused 17 (delta 5), pack-reused 12000
Receiving objects: 100% (12028/12028), 56.77 MiB | 7.52 MiB/s, done.
Resolving deltas: 100% (5893/5893), done.
Checking out files: 100% (9136/9136), done.

$ cd my-fork/

$ git status
On branch fix-path-issue-with-invalid-files
Your branch is up to date with 'origin/fix-path-issue-with-invalid-files'.

nothing to commit, working tree clean
```

Now, you just need to merge this branch into the default branch and everyone
will benefit from this change.