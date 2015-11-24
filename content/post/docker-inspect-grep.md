+++
date = "2015-10-19T21:09:16-04:00"
tags = ["docker", "bash"]
author = "Tim Gossett"
title = "docker inspect | grep"
+++

We've dockerized all of the apps the comprise our platform at work. Almost all of the configuration data that our apps need is provided via environment variables. Sometimes I need to verify that the correct configuration was provided for a particular environment variable.

I've executed the following command enough times now that I figured I should write it down somewhere.

```bash
# $cid is the target container's ID, and $var is some part of the environment variable's name
docker inspect -f '{{range $_, $e := .Config.Environment}}{{println $e}}{{end}}' $cid | grep $var
```

This executes [`docker inspect`](https://docs.docker.com/engine/reference/commandline/inspect/) with a Go template that will loop through the container's environment variables (which are stored as a flat array of strings like `["VAR1=value1", "VAR2=value2"]`) and print each variable on a new line. That output is piped to `grep` which searches for the environment variable I'm after.

To spice it up a bit: if there's only one container running, then there's no need to know its CID.

```bash
docker ps -q | xargs docker inspect -f '{{range $_, $e := .Config.Environment}}{{println $e}}{{end}}' | grep $var
```

[`docker ps -q`](https://docs.docker.com/engine/reference/commandline/ps/) will list just the CIDs of the running containers. Pipe this to `xargs` followed by the `docker inspect` command above, and the CID of the one running container will effectively be added as the last argument to the command.
