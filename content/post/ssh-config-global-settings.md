+++
date = "2016-05-14T20:12:53-05:00"
title = "~/.ssh/config - Global Settings"
tags = ["sh","rc"]
author = "Tim Gossett"
+++

_This is part of a series explaining the SSH config I'm using on my machine (OS X 10.11). The config should be portable to most UNIX-based systems; Windows users are on their own._

---------------------------------------

## Quick Primer

Every time the SSH client is invoked, it looks for settings in `~/.ssh/config` (as well as some other places). The same is true for utilities that use SSH as a transport such as `scp` and `sftp`.

The config file is organized into stanzas separated by the `Host` directive; that is, a line beginning with `Host` starts a stanza, and the stanza includes all of the following lines until the next line beginning with `Host`.

The `Host` directive itself contains a whitespace-separated list of patterns. The settings in a stanza apply to hostnames that match one of the patterns. There are three pattern specifiers:

* `*` will match zero or more characters
* `?` will match exactly one character
* `!` at the start of a pattern will negate its match

The SSH client uses the first obtained value for each parameter, so more host-specific declarations should be given near the beginning of the file, and general defaults at the end. This post examines the last stanza in my `~/.ssh/config` which contains general configuration.

Full documentation for SSH config is available on most machines with `man ssh_config`.

## `Host *`

`Host *` applies to all hosts, which makes this the Global Settings stanza. Here's the full thing:

```
Host *
  CheckHostIP no
  StrictHostKeyChecking no
  ConnectTimeout 5
  ForwardAgent yes
  ControlMaster auto
  ControlPath /tmp/.ssh_control-%h-%p-%r
  ControlPersist 4h
  StreamLocalBindUnlink yes
  Compression yes
```

### `CheckHostIP no`

Don't worry if the IP address of a host has changed since the last time I connected. More often than not, I target machines that are part of a pool behind a hostname. Since that hostname may be routed to one of a pool of hosts, and hosts may come and go from the pool, I don't expect the IP to be static.

### `StrictHostKeyChecking no`

Strict host key checking expects that the host's key will not change. Since a given hostname may be routed to a pool of hosts and hosts in the pool will have different keys, I've turned off this check.

### `ConnectTimeout 5`

Wait 5 seconds to establish the connection; the default is the system-set TCP timeout value. On my machine, that's 75 seconds, as shown (in milliseconds) here:

```
$ sysctl net.inet.tcp.keepinit
net.inet.tcp.keepinit: 75000
```

### `ForwardAgent yes`

When connecting to a host, bring my keyring along. This is important when using a jump host, where I typically use one keypair to authenticate with the jump host, and a different keypair to authenticate with the target host.

### `ControlMaster auto`

Allow multiple concurrent SSH sessions to the same host to share a single network connection. For example, two concurrent `git pull`s connecting to GitHub over SSH can share a single TCP connection—[which is more efficient](http://chimera.labs.oreilly.com/books/1230000000545/ch02.html#SLOW_START)—instead of establishing a new connection for each. Also, two sessions to separate hosts that use the same jump host can share a single TCP connection to that jump host. The `auto` option here tells SSH to look for an existing connection and create one if it doesn't already exist.

### `ControlPath /tmp/.ssh_control-%h-%p-%r`

This specifies a format string SSH should use for the filename when creating a Unix domain socket file for new TCP connections. The `%h`, `%p`, and `%r` placeholders are for the hostname, port, and remote username, respectively. As an example, the filename for a connection to GitHub will be `/tmp/.ssh_control-github.com-22-git`.

### `ControlPersist 4h`

This sets a 4-hour idle timeout for SSH connections. As long as the remote host doesn't hang up, the SSH connection will remain open in the background, ready for reuse, for 4 hours after the last activity. Network communication is more [efficient when existing TCP connections are reused](http://chimera.labs.oreilly.com/books/1230000000545/ch02.html#SLOW_START).

### `StreamLocalBindUnlink yes`

Will remove an existing Unix domain socket file before attempting to create a new one. If the old one hangs around, SSH won't be able to use it.

### `Compression yes`

Compresses data before putting it on the wire. Network communication is [faster when there are less bytes to send](http://chimera.labs.oreilly.com/books/1230000000545/ch02.html#_tuning_application_behavior). For latent or long-distance connections, the time spent compressing data will usually be less than the time it would take to transmit extra bytes; fast or local connections won't benefit from compression. Since I typically connect to remote machines, having compression on by default is helpful.

---------------------------------------

This is part of a series explaining the SSH config I'm using on my machine. The full series will be:

1. General Settings
2. GitHub
3. Stomping Grounds
4. Development Host
5. Jump Host
