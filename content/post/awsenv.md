+++
date = "2015-03-08T21:06:46-04:00"
title = "awsenv"
tags = ["rc", "AWS", "bash"]
author = "Tim Gossett"
+++

## TL;DR

I have this cool `awsenv` function. [Check it out](#tldr)

_Full disclosure: the `awsenv` function in this post is based on a `zsh` script my co-worker [phemmer](https://github.com/phemmer) wrote. I ported it to `bash`._

## Why?

I deal with a few different AWS accounts in the course of a day. We use separate accounts at work for our development and production deploys, and I also have a personal account to kick around.

AWS has a nice suite of command-line utilities, but there's one aspect that I don't agree with: credential management. Suffice it to say that it's not easy to switch between AWS accounts using the provided CLIs. Fortunately, the AWS CLIs can make use of credentials set as environment variables, and environment variables can be swapped rather easily.

## The Credentials

The end goal is a function (we'll call it `awsenv`) that will set a number of environment variables associated with my label for the AWS account I was to use, and drop me in a subshell.

We'll start off by declaring two associative arrays, one for the keypairs and another for the regions.

```bash
declare -A keypairs regions
```

It would be best to keep the secret information in a separate file. I tend to put things like this in `~/.secrets`. Also, it would be convenient if we can just source that file, and it will fill in the associative arrays for us.

```bash
source ~/.secrets/aws
```

The content of that file looks something like this:

```bash
# ~/.secrets/aws

keypairs[mine]='AKIA... <shh-its-a-secret>'
regions[mine]='us-west-2'

keypairs[dev]='AKIA... <shh-its-a-secret>'
regions[dev]='us-east-1'

keypairs[prod]='AKIA... <shh-its-a-secret>'
regions[prod]='us-west-1'
```

So now `keypairs` and `regions` are ready to go.

## Usage

It's not uncommon for me to temporarily forget how to use something I wrote. It's good practice to respond to unexpected or invalid input with some usage tips, just to help the user out. Let's handle that first.

There are two conditions to check that indicate the user needs some help. The first is that no arguments were passed at all, which we check with `[[ -z "$1" ]]`. This evaluates to true if the first argument passed to the function (`$1`) is zero-length. The second condition is that the user passed an invalid account label, which we check with `[[ -z "${keypairs[$1]}" ]]`. This fetches the value associated with the key (`"${keypairs[$1]}"`), and tests if that value is zero-length, which indicates that the key (`$1`) is not a valid account label.

So far we have this:

```bash
if [[ -z "$1" ]] || [[ -z "${keypairs[$1]}" ]]; then
```

Then what? Let's help out the user by printing a list of account labels. We'll start by printing a header:

```bash
printf 'Environments:\n'
```

Then we'll loop through the keys of the `keypairs` array (using `"${!keypairs[@]}"` to get the list of keys), printing each key as we go.

```bash
for name in "${!keypairs[@]}"; do
	printf '  %s\n' "$name"
done
```

Here's what that look like on my machine:

```sh
$ awsenv
Environments:
  mine
  dev
  prod
```

## Creating an Environment

Now to the meaty part. The user has given us a valid account label, so we need to fetch the values for that label and set environment variables accordingly.

First off, let's get those credentials. We'll set a local variable with the content of `${keypairs[$1]}`, which will be a list of strings. This will save some repition later on.

```bash
creds=(${keypairs[$1]})
```

Now let's set environment variables that the AWS CLIs will use for credentials.

```bash
export AWS_ACCESS_KEY="${creds[0]}"
export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY"
export AWS_SECRET_KEY="${creds[1]}"
export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_KEY"
```

Some CLIs use the contents of a credential file, the path to which should be set in `AWS_CREDENTIAL_FILE`. First, let's just set the environment variable with the path to a file we'll create later:

```bash
export AWS_CREDENTIAL_FILE="/tmp/.$USER-$1.awskey"
```

On my machine, that variable ends up looking like this:

```sh
$ echo $AWS_CREDENTIAL_FILE
/tmp/.tim-dev.awskey
```

Now let's create that file and allow only the user to read and write to it. If there are any issues setting the permissions on the file, bail out with an exit code of `1` to indicate an error.

```bash
touch "$AWS_CREDENTIAL_FILE"
chmod 0600 "$AWS_CREDENTIAL_FILE" || return 1
```

Now to write the contents of the file. The access key should be on one line prefixed with `AWSAccessKeyId=`, and the secret should be on another line prefixed with `AWSSecretKey=`. We can handle this with one command:

```bash
printf 'AWSAccessKeyId=%s\nAWSSecretKey=%s\n' "$AWS_ACCESS_KEY" "$AWS_SECRET_KEY" > "$AWS_CREDENTIAL_FILE"
```

That will interpolate `$AWS_ACCESS_KEY` and `$AWS_SECRET_KEY` into their respective `%s` placeholders, and write the result to the path specified by `AWS_CREDENTIAL_FILE`, truncating any content that already exists in the file.

Now let's check if the label the user gave us has an associated region. `[[ -n "${regions[$1]}" ]]` will attempt to fetch the value from the `regions` array using our label (`$1`) as the key. If the result is not null (tested with `-n`), the condition evaluates to true and we're clear to `export AWS_DEFAULT_REGION="${regions[$1]}"`.

```bash
if [[ -n "${regions[$1]}" ]]; then
	export AWS_DEFAULT_REGION="${regions[$1]}"
fi
```

## The Subshell

Now that we have an environment configured for the correct AWS account, we need to drop the user in a subshell to run some commands with that account.

For sanity, I track subshells with the `SHELLSTACK` environment variable. I've added that variable to my prompt so that I can quickly see how many levels deep I am, and if [the top is still spinning](http://en.wikipedia.org/wiki/Inception). Since we're about to drop the user down another level, let's append our label to `SHELLSTACK`.

```bash
export SHELLSTACK="$SHELLSTACK aws:$1"
```

At this point we're done with the label, which has heretofore occupied `$1`. Let's `shift` and be done with it; any following arguments will "shift" up a space, occupying the void left at `$1` and following.

```bash
shift
```

#### Now for a neat trick.

A majority of my use of `ssh` is in the form `ssh <host>`, where I want to start a secure shell session on the remote host. Once the session is established, `ssh` drops me in a shell on the other end.

However, I really like (and sometimes use) the form `ssh <host> <command>`, which establishes a secure connection to the remote host, executes the command, and then terminates the connection. This saves me from having to wait for a shell before I can execute the command, and then exit the shell when it's done.

We can get the same effect with `awsenv`. At this point, `shift` has thrown aside the account label that the user passed. If any arguments remain, then the user has given us a command to run. If no argument remain, then the user is looking for a shell.

`$#` will give us the count of arguments (having been modified by `shift` earlier). If that count is `>= 1`, then we have a command to run, which we can execute with `exec "$@"`. Otherwise, just need a shell, which we can get with `exec "$SHELL"` (`SHELL` being something like `/bin/bash`).

```bash
if (( $# >= 1 )); then
	exec "$@"
else
	exec "$SHELL"
fi
```

Lastly, to make all this happen in a subshell, we need to wrap parens around everything from `creds=(${keypairs[$1]})` to the `fi` after `exec "$SHELL"`.

## Put it all together

<a name="tldr"></a>Here's the complete function in all its glory:

```bash
# ~/.awsrc

function awsenv() {
	declare -A keypairs regions
	source ~/.secrets/aws

	if [[ -z "$1" ]] || [[ -z "${keypairs[$1]}" ]]; then
		printf 'Environments:\n'
		for name in "${!keypairs[@]}"; do
			printf '  %s\n' "$name"
		done
	else
		(
			creds=(${keypairs[$1]})
			export AWS_ACCESS_KEY="${creds[0]}"
			export AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY"
			export AWS_SECRET_KEY="${creds[1]}"
			export AWS_SECRET_ACCESS_KEY="$AWS_SECRET_KEY"
			export AWS_CREDENTIAL_FILE="/tmp/.$USER-$1.awskey"
			touch "$AWS_CREDENTIAL_FILE"
			chmod 0600 "$AWS_CREDENTIAL_FILE" || return 1
			printf 'AWSAccessKeyId=%s\nAWSSecretKey=%s\n' "$AWS_ACCESS_KEY" "$AWS_SECRET_KEY" > "$AWS_CREDENTIAL_FILE"


			if [[ -n "${regions[$1]}" ]]; then
				export AWS_DEFAULT_REGION="${regions[$1]}"
			fi

			export SHELLSTACK="$SHELLSTACK aws:$1"
			shift
			if (( $# >= 1 )); then
				exec "$@"
			else
				exec "$SHELL"
			fi
		)
	fi
}
```

Have fun!