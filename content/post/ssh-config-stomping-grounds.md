+++
date = "2016-06-011T18:04:08-05:00"
title = "~/.ssh/config - Stomping Grounds"
tags = ["sh","rc"]
author = "Tim Gossett"
+++

_This is part of a series explaining the SSH config I'm using on my machine (OS X 10.11). The config should be portable to most UNIX-based systems; Windows users are on their own._

---------------------------------------

A majority of the SSH traffic produced by an operator is to and from running VMs. My SSH config makes it easier to quickly and securely connect to VMs.

## `Host jump*`

`Host jump*` applies to any hostname that starts with "jump". Here's the full stanza:

```
Host jump*
  IdentityFile ~/.ssh/jump
  User ec2-user
  HostName %h.mrgossett.com
```

### `User ec2-user`

I don't have a dedicated user for these hosts, so I use the default `ec2-user`. Specifying `User` here means I don't have to type it as part of the `ssh` command.

### `IdentityFile ~/.ssh/jump`

Use the SSH key that I generated specifically for use with jump boxes. The public key for this pair would have been added to the `~/.ssh/authorized_keys` file on the target host. Specifying it here means that it's sure to be used, even if it's not part of my keychain.

### `HostName %h.mrgossett.com`

This saves some typing by expanding the hostname I provided at the command line using this format string. If I type `jump1` as the hostname, SSH will expand it to `jump1.mrgossett.com`.

---------------------------------------

# What's a jump host?

It's generally not good security practice to allow direct SSH access from the public internet to every host in an environment. A "jump host" (a.k.a., "bounce box" or "jump box") acts as a proxy. The protected hosts will allow SSH traffic coming from a known set of jump hosts (usually one or two). The jump hosts allow SSH traffic from the public internet and are (should be) secured as much as possible. This way, the protected hosts only need to worry about trusting one or two jump hosts, and the jump hosts only need to be worried about authenticating SSH traffic on a single port.

## Jump Host example config

Below is an example config stanza for working with a jump host:

```
Host target-host
  ProxyCommand ssh -q jump-user@jump-host nc %h %p
  Compression no
```

### `ProxyCommand ssh -q jump-user@jump-host nc %h %p`

SSH usually connects directly to a host and starts talking SSH protocol. The `ProxyCommand` instead will handle the connecting part. SSH executes `ProxyCommand` and pipes in its SSH protocol via `STDIN`. `ProxyCommand` then is responsible for shipping that data off to an SSH daemon somewhere, and returning response data to `STDOUT`. Note that `%h` and `%p` will be expanded to the target hostname and port, respectively.

Here, `ProxyCommand` first is establishing an SSH connection to `jump-host` (`ssh -q jump-user@jump-host`). The `-q` flag puts that first SSH session in quiet mode so that warnings and diagnostic messages are suppressed. Once the connection to the jump host is established, we need to make the next hop to the target machine.

The next part of `ProxyCommand` points `STDIN` at the target hostname and port using `netcat` (a.k.a `nc`), a standard UNIX utility for reading from and writing to network connections. `STDOUT` is also routed back for the return trip from the target host to the client's machine, fulfilling the requirement for `ProxyCommand` to return response data on `STDOUT`.

### `Compression no`

My global settings turn on compression for any SSH connection. In this scenario, we're connecting to the jump host before we connect to the target host, so the connection to the jump host will be compressed. Leaving compression on for the target host would result in double-compressed data. The connection from the jump host to the target host should be reasonably fast since they're likely on the same LAN, or otherwise in the same data center, so compressing traffic between them won't really speed things up that much. This directive turns off compression for the target host, avoiding double compression.

## Another Revision

The following two stanzas have the same effect as the Jump Host example config above, but a bit cleaner.

```
Host target-host
  ProxyCommand ssh -q jump-host nc %h %p
  Compression no

Host jump-host
  User jump-user
  HostName 127.0.53.53
```
The settings for `Host jump-host` are applied as usual in the `ProxyCommand` for `Host target-host`.

### `User jump-user`

Whenever I connect to this jump host, I want to do so as a user named `jump-user`.

### `HostName 127.0.0.53`

This is using `HostName` to create an alias so that SSH knows `jump-host` is really IP `127.0.53.53` (just an example).

---------------------------------------

This is part of a series explaining the SSH config I'm using on my machine. The full series will be:

1. [General Settings](/post/ssh-config-global-settings/)
2. [GitHub](/post/ssh-config-github/)
3. Stomping Grounds
4. Development Host
5. Jump Host
