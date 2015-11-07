+++
date = "2015-05-14T14:28:56-04:00"
title = "~/.gitconfig aliases"
tags = ["rc", "bash", "git"]
author = "Tim Gossett"
+++

I use `git` from the command line as part of my regular workflow. Configuring aliases saves just a few keystrokes, but those keystrokes add up (My bash history says I've executed `git` commands 121 times in my current shell session, which has spanned about two workdays).

I have `git` itself aliased to `g` in my `~/.bashrc`. `git` allows aliasing subcommands in the `[alias]` section of `~/.gitconfig`. Below are my aliases in workflow order:

## Fetch & Checkout

To kick it off, I typically `git fetch` from remote to make sure `git` is aware of the branches that exist on the remote end. Then I `git checkout` the branch that I want to work on, whether the branch already exists on the remote end or I create it locally. Those two commands look like `g f` and `g co` with the following aliases:

```
f = fetch
co = checkout
```

## Status & Diff

To see what's changed, I use `git status` and `git diff` often.

`git status` will just list the files that have changed since the last commit (both staged and unstaged). I have it aliased as `g s`

Sometimes I want to see changes that have not yet been staged for a commit (plain `git diff`), and other times I want to see only the changes that have been staged for a commit (`git diff --staged`). With the following aliases, those will end up looking like `g d` and `g ds`.

```
s = status
d = diff
ds = diff --staged
```

## Add

I typically stage changes wholesale instead of adding file-by-file; that's the behavior of `git add --all`, which I aliased as `g a`. Sometimes, though, it's better to add only some of the parts of a file that have changed and not others. That's possible with `git add --patch`, aliased as `g ap`.

```
a = add --all
ap = add --patch
```

## Commit

Once I've staged everything I want to include in a changeset, it's time to `git commit`. My first alias, `g c`, expands to `git commit -v`. The behavior of the `-v` flag is described in `git`'s manpage like this:

```
-v, --verbose
   Show unified diff between the HEAD commit and what would be committed at the bottom of the commit message template to help the user describe the commit by reminding what changes the commit has.
   Note that this diff output doesn't have its lines prefixed with #. This diff will not be a part of the commit message.
```

Occasionally, I make a typo in the commit message, or I forget a small change that I want to slip into the commit I just made. Both scenarios involve `git commit --amend`, which I've alased as `g ca`.

```
c = commit -v
ca = commit --amend
```

## Trim

Over the course of a few weeks (or a couple of intense days), I may end up with a few dozen feature branches in one repo, most of which have been merged into `master`. To clean up the branches that have been merged, I use the following alias, which executes a shell command:

```
trim = "!sh -c 'git branch --merged | grep  -v \"\\*\\|master\\|production\" | xargs -n1 git branch -d'"
```

This lists all of the branches that have been merged, drops `master`, `production`, and the current branch from that list, and then passes the final list to `git branch -d` to delete them.

## All Together

Here's the whole thing:

```
[alias]
	f = fetch
	co = checkout
	d = diff
	ds = diff --staged
	a = add --all
	ap = add --patch
	s = status
	c = commit -v
	ca = commit --amend
	trim = "!sh -c 'git branch --merged | grep  -v \"\\*\\|master\\|production\" | xargs -n1 git branch -d'"
```