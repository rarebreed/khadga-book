# Using Docker

In order to test our new code changes as well as deploy our final app, we need a way to deliver
a usable artifact that we can either run tests against, or actually use in production.

Although rust is very nice in that it produces standalone binaries, our product here is actually a
set of services.  For example, khadga needs a database and we can't stuff a database inside our
binary (well, we _could_, but where would you persist the data?).  Other parts that probably aren't
good to stuff inside the binary are configuration files.  Many times you want to dynamically
configure how an application runs based on a file that it can read.  If we statically link in the
file to the binary, it's stuck with that configuration.  If you allow your app to read the config
file at runtime, where and how do you bring along the configuration file(s)?

To solve these kinds of problems, we will use docker and docker-compose.  If you are not that
familiar with docker, that's fine.  Here, I will explain what is going on, and why we are doing what
we are doing.

## Installing docker

Depending on your operating system, this can be a tricky affair, so I will direct you to the docs
on the [docker site][-docker-install].

## Creating a Dockerfile

The Dockerfile will be our recipe to create a container that contains our backend server and the
configuration files that it needs.

First, we create a file named Dockerfile in the root of our project.  It will look like this:

```dockerfile
FROM ubuntu:bionic

RUN apt update \
    && apt upgrade -y \
    && apt install -y curl \
    && mkdir -p /apps/vision/dist

# TODO: create a khadga user and run as that user.  We don't need to run as root

# Copy the dist that was generated from wasm-pack and webpack to our working dir
# then, copy the executable to the vision directory.  This is because the binary
# is serving files from ./dist
WORKDIR /apps/vision
COPY vision/dist ./dist
COPY khadga/config ./config
COPY target/debug/khadga .

CMD [ "./khadga" ]
```

## Explanation

So what's this Dockerfile file doing?  For those not familiar with docker, the Dockerfile is the
default name for the file that the docker CLI will look for when running certain subcommands like
build.  It is a file that contains instructions for how to build a docker image step-by-step.

As a quick aside, do not confuse docker images, from docker containers.  A container is really a
kind of fancy process with special restrictions.  An image is like an executable binary.  From one
binary, you can launch many processes.  The docker image is the _executable_ that a container is
spawned from.  The command `docker ps` will list running containers, while `docker images` will list
docker images on your system.

Each line of code is actually creatig a new layer in a docker image (which is why you frequently see
certains commands like RUN using && to concatenate commands to a single layer).  This layering is
also important to understand when you make changes to your Dockerfile.  As each command in the
Dockerfile is encountered, docker will create a layer which it caches so that it doesn't need to be
done again.  If you only ever add new steps at the bottom of the Dockerfile, that's fine.  However,
if you change a step at the beginning (for example), all steps after your new change must be
executed again, since the layer system is immutable.  This can increase your build times
significantly.

### Our base image

So our first step is a `FROM` command, which basically specifies what base image to use.  Here, we
are saying that the base image we will use is an ubuntu bionic release.  Notice the use of the colon
here.  To the left of the colon is the base image name (ubuntu in this case), and to the right of
the colon is a tag.  The colon and tag name are optional, but if you do not use them, there is
typically a default tag (usually `:latest`).

What this does is download from the docker hub repository an image named ubuntu, with the tag of
bionic. This is our base image.  From this base image, we will add more to it.

### Adding new components to our base image

Next, we encounter the `RUN` command.  `RUN` takes a shell command and executes it within a docker
daemon (which is actually executing our base image).  If you look at the shell commands it is
running, you can see it is updating the operating system, adding curl and creating some directories.

### Copying files from dev machine to docker image

A critical thing to understand in docker is the difference between our development machine and the
docker image itself.  We need to get the files from our dev machine onto the docker image.  Another
important thing to understand is the idea of a context which is like a build directory.

When you actually build your docker image, you specify a build context argument.  For example:

```sh
sudo docker build -t <username></tag> .
```

The . at the end tells docker "use the current directory as the build context".  The build context
is the point of view from your development machine.  If you use your current directory as your build
context, any `COPY` operations in the Dockerfile will assume that the source is from that build
context.

For the sake of the example, let's assume that you are currently in `/path/to/my-project`.  And that
you then run 

The `WORKDIR` command tells the docker daemon "use this directory as our working directory from the
docker image point of view".  In the Dockerfile, we are using `WORKDIR /app/vision`.  Any command
that copies files for example, will use this directory as the destination directory by default.

So the next command `COPY vision/dist ./dist` is basically saying "Copy the files from
${BuildContext}/vision/dist to ${WORKDIR}/dist".  So, if you set your build context to "." during
the `docker build` command, and that you are currently in the `/path/to/my-project` directory, that
this will copy our project's `vision/dist` directory to the docker image's `/app/vision/dist`
directory.

Hopefully the next `COPY` commands are clear now.  We also copy our local khadga/config on our dev
machine to the image's  /app/vision/config directory.  The last `COPY` is interesting.  Here, we
will copy the binary generated from cargo (in this case the debug version) to our docker image in
/app/vision.  Eventually, we'll want to somehow dynamically set that based on whether we want to use
the docker container for testing or release it to production.

### Executing our process

A docker container executes what's in the `CMD` operation.  The first arg is our command or
executable, and subsequent elements in the array are any arguments the executable needs.

## Setting up mongodb with docker-compose

This chapter is about mongodb, but so far we haven't touched mongodb at all.  Creating the
Dockerfile was necessary however for our next piece, the docker-compose file, which will be
explained in the next section

[-docker-install]: https://docs.docker.com/install/