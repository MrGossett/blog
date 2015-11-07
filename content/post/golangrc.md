+++
date = "2015-03-25T14:05:03-04:00"
title = "~/.golangrc"
tags = ["rc", "go", "bash"]
author = "Tim Gossett"
+++

## TL;DR

I wrote some bash functions that are a regular part of my workflow. [Check them out](#tldr)

## Go source

I have Go's source cloned in `~/goroot`. Since the source is now managed with `git`, I can switch between versions very easily, and I can browse the source for Go's standard library using my editor.

To clone the source, I did this:

```sh
git clone https://go.googlesource.com/go ~/goroot
```

Since this constitutes ["installing [Go] to a custom location"](https://golang.org/doc/install#tarball_non_standard), Go will need a few environment variables set:

```bash
export GOROOT=$HOME/goroot
export GOPATH=$HOME/go
export GOBIN=$GOROOT/bin
export PATH=$GOBIN:$PATH
```

## Helper functions

I wrote a few helper functions which are basically aliases for commands I execute often.

### Go me!

Take me to my personal projects in `GOPATH`:

```bash
function gome() {
	cd $GOPATH/src/github.com/MrGossett
}
```

### Go work.

Take me to my work projects in `GOPATH`:

```bash
function gowork() {
	cd $GOPATH/src/github.com/StealthNinjaOrgName
}
```

### Check it before code reviewers wreck it

We have pretty high standards for code quality at work. It's not a direct mapping, but if any of the popular Go code quality tools complain about my code, then so will my co-workers. As a last step before I check in my code, I run `style-check`, which executes `golint`, `go vet`, `gofmt`, `gocyclo`, and `go build`:

```bash
function style-check() {
	golint ./...
	go vet ./...
	find . -path ./Godeps -prune -o -name '*.go' -exec gofmt -l -s {} +
	find . -path ./Godeps -prune -o -name '*.go' -exec gocyclo -over 10 {} +
	go build -o /tmp/a
}
```

## All together now

<a name="tldr"></a>Here's my full `~/.golangrc`

```bash
# ~/.golangrc

export GOROOT=$HOME/goroot
export GOPATH=$HOME/go
export GOBIN=$GOROOT/bin
export PATH=$GOBIN:$PATH

function gome() {
	cd $GOPATH/src/github.com/MrGossett
}

function gowork() {
	cd $GOPATH/src/github.com/StealthNinjaOrgName
}

function style-check() {
	golint ./...
	go vet ./...
	find . -path ./Godeps -prune -o -name '*.go' -exec gofmt -l -s {} +
	find . -path ./Godeps -prune -o -name '*.go' -exec gocyclo -over 10 {} +
	go build -o /tmp/a
}
```