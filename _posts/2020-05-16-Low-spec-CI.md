---
layout: post
title: Low spec CI with git
---

One of the most annoying things is when I have to compile and publish a version of a dotnet core
project for a docker container in Windows 10. Open visual studio, waiting 10 minutes for it to respond, start docker desktop (pray to get no errors) and try to publish the project without the possibility of doing nothing else.

Maybe is the use of a virtual machine that Windows need to run docker or maybe the lot of resources that Visual Studio take to open and publish a project, basically (and in case isn't enough clear) I hate make the publishes in my limited working desktop computer. 

Well, the first thing that went through my mind was to do a little script to automate the process, 
but that doesn't solve the resource consumption that limit the use of my computer and configuring any
environment with a CI tool, like Travis or Jenkins, can't take too much time for me. The fast 
solution? Doing all the deploy and delivery in a little Virtual Machine and using git hooks.

# Automate CI and deploy with git hooks

The use is really simply:

* Use a git repository in our VM as a remote.
* Make a script to automate and use it as a post-update hook.

## Remote git repository in the VM

In this case I'm using a VM with Ubuntu 18.04 and ZeroTier. With ZeroTier I can access the VM without
public IP nor being in the same network segment.

To use our VM as a git remote, we need to generate a bare git repository and copy it to the server; this can be done using SCP or SFTP, WinSCP can easily help yo to copy the bare git folder. I assume you
have access to the VM with SSH.

Creating a bare git folder:

```shell
$ git clone --bare my_project my_project.git
```

Putting the bare repository on a server:

```shell
$ scp -r my_project.git user@git.example.com:/srv/git
```

Now you can clone or push your changes to the remote repo:

```shell
$ git clone user@git.example.com:/srv/git/my_project.git
```

## Making a post-update hook script

If we explore the bare repository, we can see a folder called hooks

```shell
$ ls my_project.git/hooks/
applypatch-msg.sample      post-update.sample         pre-push.sample
commit-msg.sample          pre-applypatch.sample      pre-rebase.sample
fsmonitor-watchman.sample  pre-commit.sample          pre-receive.sample
post-update                prepare-commit-msg.sample  update.sample

```

Hooks folder contain sample hooks that run in specific events, in this case we have interest in 
post-update hook. For this example I'm gonna show a bash script, but this can be done with any
script language.

## Detecting the branch

Now we can edit the post-update.sample file with our preferred text editor:

```sh
#!/bin/sh
#
# An example hook script to prepare a packed repository for use over
# dumb transports.
#
# To enable this hook, rename this file to "post-update".

exec git update-server-info

```

Lets wipe all of this and add our lines to use bash and detect our branch.

```bash
#!/bin/bash

# Taking branch from the first argument $1 
branch=$(echo $1 | awk -F / ' { print $3 } ')

```

Why do this? We can condition the script execution depending of the branch

```bash
!#/bin/bash

# Taking branch from the first argument $1 
branch=$(echo $1 | awk -F / ' { print $3 } ')

if [[ "$branch" == "master" ]] ; then
	# Things to do
fi

```

## Continuous Integration of dotnet core

This script is specific the project, but you can take some ideas.

We need to assign a directory for our CI; its necessary to clean this directory for each integration
and clone our project to that folder.

```bash

...

if [[ "$branch" == "master" ]] ; then

	# Remove old files
	rm -Rf /CI/* /CI/.* 

	# Cloning the project
	git clone --single-branch --branch master  /srv/git/my_project.git /CI/my_project
	
	...

fi

```

Now we are ready to implement the automation. For this case the project only need to do a  `dotnet publish` command to get the publication. After the publish command, we gonna make a docker image and run it.

```bash

...

if [[ "$branch" == "master" ]] ; then

	...

	dotnet publish /CI/my_project -o /CI/publish/my_project

	# Generate image from the publish

	docker build -t my_project:mytag  /CI/publish/my_project/

fi

```

You can add lines to autodeploy the image:

```bash

...

if [[ "$branch" == "master" ]] ; then

	...

	docker run -d --name myprojectcontainer  -p 80:80  my_project:mytag

fi

```

And thats it! Now is time to save, rename the ".sample" from the hook and make it an executable

```shell
$ mv post-update.sample post-update
$ chmod +x post-update
```

## Using multiple remotes

We can add our new remote. This process is for our working desktop:

```shell
$ git remote add my_ci_remote user@git.example.com:/srv/git/my_project.git
```

and now we can make a `git push` to this remote every time we want to run our CI script


```shell
$ git push my_ci_remote master
```

Thats it! 

