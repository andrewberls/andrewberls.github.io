---
layout: post
title: "Git tricks: Unstaging files"
date: "2013-05-06"
permalink: /blog/post/git-tricks-unstaging-files
---

<em>This post dives a bit into the git reset command. If you want to jump straight to the good stuff, click <a href="#alias">here</a>.</em>

I wanted to share a handy alias I use for removing files from the staging area in git. Often I'll be working and adding files to the staging area with `git add`, and then decide (for example), that I don't want to commit some files with the others and get them out of the staging area (but keep my work intact). Let's look at an example case - running `git status` after staging a file might look like this:

<pre><code>$ git add example.txt
$ git status
# On branch test
# Changes to be committed:
#   (use "git reset HEAD \<file\>..." to unstage)
#
#	modified:   example.txt
#</code></pre>

<break />

Hey, that's neat - it tells us right there how to unstage files! This is accomplished using the [git reset](https://www.kernel.org/pub/software/scm/git/docs/git-reset.html) command. From the docs:

<pre><code>git reset [-q] [&lt;commit&gt;] [--] &lt;paths&gt;…
This form resets the index entries for all &lt;paths&gt to their state at &lt;commit&gt;. 
(It does not affect the working tree, nor the current branch.)

This means that git reset &lt;paths&gt is the opposite of git add &lt;paths&gt.</code></pre>

`git reset` can be used for several things -it can point the current HEAD at a specified state, and we'll be using it to copy entries from HEAD to the index, or staging area, thus removing our file from the staging area and leaving our working tree unchanged. Also important is that `<commit>` defaults to HEAD, so we can leave that bit out. Let's give it a shot!

<pre><code>$ git reset -- example.txt
Unstaged changes after reset:
M	example.txt
$ git status
# On branch test
# Changes not staged for commit:
#   (use "git add \<file\>..." to update what will be committed)
#   (use "git checkout -- \<file\>..." to discard changes in working directory)
#
#	modified:   example.txt
#
no changes added to commit (use "git add" and/or "git commit -a")</code></pre>

And our changes are left intact but removed from the staging area! Mission accomplished! However, you might be wondering about the strange `--` that shows up in the command. Those are referred to as the 'bare double dashes', and they prevent ambiguity by ensuring that we're resetting a file and not specifying a branch. For example, if you have a branch and a file both named `example` and you run `git reset example`, you might see the following:

<pre><code>fatal: ambiguous argument 'example': both revision and filename
Use '--' to separate filenames from revisions</code></pre>

Git can't tell if you're talking about the branch or the file (each of which has very different consequences), so the `--` makes it clear we're not talking about a branch and that we want to reset our file.

So we've seen that we can use `git reset` to unstage our files. However, it'd be nice to be able to say `git unstage <paths>`, so let's define that using a git alias:

<a name="alias"></a>
<pre><code>git config --global alias.unstage 'reset --'</code></pre>

And there you have it! Now we can run `git unstage example.txt` to our heart's content.
