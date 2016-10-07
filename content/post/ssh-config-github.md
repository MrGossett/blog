+++
date = "2016-06-04T22:14:47-05:00"
title = "~/.ssh/config - GitHub"
tags = ["sh","rc"]
author = "Tim Gossett"
+++

_This is part of a series explaining the SSH config I'm using on my machine (OS X 10.11). The config should be portable to most UNIX-based systems; Windows users are on their own._

---------------------------------------

A majority of the SSH traffic produced by a developer is to and from GitHub. My SSH config adds a little tuning specific to `github.com`.

## `Host github.com`

`Host github.com` applies to any SSH traffic (including `git` over SSH) routed to the `github.com` hostname. Here's the full stanza:

```
Host github.com
  User git
  ControlPersist 570
  ForwardAgent no
  IdentitiesOnly yes
  IdentityFile ~/.ssh/github
```

### `User git`

GitHub's SSH URLs all use the `git` user (`git@github.com:...`). I mirrored that in my config for good form, though in practice it probably doesn't need to be there.

### `ControlPersist 570`

My global config uses a 4-hour idle timeout, but GitHub will hang up after 10 minutes. This adjusts the idle timeout to 9.5 minutes. The idea is to leave a connection to GitHub open so that it can be reused, but to close it before it gets closed on GitHub's end due to inactivity.

When the remote side closes a connection, the SSH client's default behavior is to print a message on STDERR saying something like "`connection closed by github.com`". Random and unexpected messages on STDERR can get annoying.

### `ForwardAgent no`

I use a dedicated keypair for GitHub from each machine. There's no need to bring my entire keychain along since I'll never need to use any other key in the chain from GitHub's remote host.

### `IdentitiesOnly yes`

Only use the identity file I specify in config, and don't use the keychain at all.

### `IdentityFile ~/.ssh/github`

This specifies the private key to use when connecting to GitHub. Specifying it here means I don't need to have it in my keychain.

---------------------------------------

This is part of a series explaining the SSH config I'm using on my machine. The full series will be:

1. [General Settings](/post/ssh-config-global-settings/)
2. GitHub
3. Stomping Grounds
4. Development Host
5. Jump Host
