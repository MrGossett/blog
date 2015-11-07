+++
date = "2015-02-20T21:06:46-04:00"
title = "~/.dockerrc"
tags = ["rc", "bash", "docker"]
author = "Tim Gossett"
+++

We use [Docker](https://www.docker.com/) at work as part of our deployment strategy. Since we ship dockerized apps, Docker is part of our development environment. I develop in OS X, so I use [Boot2Docker](http://boot2docker.io/).

One of the original core contributors to Docker, [@creack](https://github.com/creack), was part of our team for a while. It was a great opportunity to get familiar with Docker as a technology. He helped me put together a `~/.dockerrc` file that ensures my development environment is always ready to go. I source that `~/.dockerrc` from my `~/.bashrc` so that each new shell I open is wired up with `docker`.

Here's the full `~/.dockerrc`:

```bash
if [ ! `boot2docker status` = "running" ]; then
	boot2docker start > /dev/null
fi

if [ -z "$DOCKER_HOST" ]; then
	eval $(boot2docker shellinit 2>/dev/null)
	export DOCKER_IP=$(boot2docker ip 2> /dev/null)
fi

function rmi-docker() {
	docker images -q --filter 'dangling=true' | xargs docker rmi
}

function drop-containers() {
	docker ps -aq | xargs docker rm -fv
}
```

I'll break down each section.

## Auto-start

If `boot2docker` is not running for some reason, I want to start it.

```bash
if [ ! `boot2docker status` = "running" ]; then
	boot2docker start > /dev/null
fi
```

 I'm not really interested in the output of `boot2docker start`, since this happens every time I open a new shell; so I pipe its `STDOUT` to `/dev/null`.

## Environment Variables

The `docker` client relies on a few environment variables being set so that it knows where the docker daemon is running, and how to talk to it. At the moment, those variables are `DOCKER_HOST`, `DOCKER_TLS_VERIFY`, and `DOCKER_CERT_PATH`. `boot2docker` provides a handy `shellinit` command that spits out some contextual information on `STDERR`, and `export` commands for the environment variables it needs on `STDOUT`. Here's what it looks like in my shell:

```sh
Writing /Users/tim/.boot2docker/certs/boot2docker-vm/ca.pem
Writing /Users/tim/.boot2docker/certs/boot2docker-vm/cert.pem
Writing /Users/tim/.boot2docker/certs/boot2docker-vm/key.pem
    export DOCKER_CERT_PATH=/Users/tim/.boot2docker/certs/boot2docker-vm
    export DOCKER_TLS_VERIFY=1
    export DOCKER_HOST=tcp://192.168.59.103:2376
```

If I were to just `eval $(boot2docker shellinit)`, those first three lines would still show up in `STDERR`, so I silence those three lines with `2>/dev/null`.

`boot2docker` also provides an `ip` command, which prints just the IP address from value it returns for `DOCKER_HOST`. It's convenient to have it around, so I also assign it to `DOCKER_IP`.

This only needs to happen if `DOCKER_HOST` has not been set, so I put it in a conditional.

```bash
if [ -z "$DOCKER_HOST" ]; then
	eval $(boot2docker shellinit 2>/dev/null)
	export DOCKER_IP=$(boot2docker ip 2> /dev/null)
fi
```

## Prune Dangling Images

After running `docker build` a number of times for the same project (making changes along the way), I tend to end up with a number of "dangling"—that is, untagged—Docker images. A majority of those images started `FROM google/golang:1.4`, and contain plenty of code vendored in `Godeps`... so they take up a lot of space. I wrote a function that just wraps a command to drop all of those dangling images.

```bash
function rmi-docker() {
	docker images -q --filter 'dangling=true' | xargs docker rmi
}
```

`docker images` lists all of the images on my machine, in a human-friendly table layout. The `-q` flag (for "quiet") prints just the IDs of the images. `--filter 'dangling=true'` filters the list to show only the untagged images. I pipe that list to `docker rmi`, which deletes them.

## Drop ALL THE CONTAINERS!

I've gotten better about cleaning up containers after they've stopped, but I still let a few fall through the cracks. I wrote a wrapper function that forcefully deletes all of the containers on my machine—even the running ones.

```bash
function drop-containers() {
	docker ps -aq | xargs docker rm -fv
}
```

`docker ps` will list the running containers on the machine in a table layout. The `-a` flag (for "all") includes stopped containers, and the `-q` flag (for "quiet") reduces the table layout to just a list of container IDs. I pipe that list to `docker rm`, which will delete stopped containers. I added the `-f` flag (for "force") to delete the running containers as well, and I added the `-v` flag (for "verbose") so that it lists the container IDs as it deletes them.