# Using kubernetes for testing and deployment

Ultimately we are creating a Google Cloud Product, and therefore we need to create a kubernetes
project.  As we develop the project and have something to actually test and deploy, we will go into
more detail.

However, we can go over the actual build container.  At the beginning of the chapter, we went over
how to setup all your local development dependencies.  However, we also need a reproducible way to
build our project.  While installing local tools is nice, what we really want to do in a production
environment are many things:

- Automate the process of building all the artifacts our project needs
- As the artifacts are built, unit test them
- Once all the artifacts have been built, run integration tests
- If all the unit and integration tests pass, create a docker image
- Build your Kubernetes Objects (deployments, services, etc) locally
- Run minikube and apply all your config files to run your app
- Run end to end tests against the app
- If the tests pass
  - Tag the build(s) of your image(s)
  - push the image to a container repo like docker hub or GCR
- Tell GCP to use the latest images and reploy
  
Yes, that's a rather large set of things to do!!  This is called CD or continuous deployment.  This
is slightly different from CI which stands for continuous integration.  The main difference is that
the latter will actually push out the end application/project to a public release, whereas the
former is more about building and testing the artifacts, but not necessarily pushing the artifact to
deployment.

Where does kubernetes fit in?

## Why kubernetes?

We want a way to make reproducible builds.  In the olden days, developers would have their local
build environments, which often was some configuration of an IDE.  When the developer was done with
their feature or bug fix, they would click some button on their IDE to build the application, and
maybe they might run some tests (yeah, this was before TDD, BDD, CI, etc).  Assuming it looked good,
said developer of yore would make a CVS or perforce commit and announce to all the coworkers that
new source code was ready.

Very often, a coworker would checkout a copy of the new source code, and lo and behold, he couldn't
even get the project to build!  What trickery was this?  Were gremlins sabotaging code in the
ancient CVS revision control system?  The fastidious engineer might have compared the source code to
make sure it was indeed correct and identical to what the developer who originally built it
successfully had on his or her system.

Well, the problem was often that one developer's configuration of his build tools was a little bit
different.  Or perhaps the build required configuration files to be present in a particular
location, and in a particular state.  It was also highly likely that the local dependencies on one
developers workstation was different from another.  For you youngsters who have only been
programming a few years, you may have wondered where the terms DLL Hell, Classpath Hell, npm hell
(pre yarn or package-lock.json days) etc all came from.  Well, one of the symptoms of the above
problems is when either your program has some kind of dependency conflict (package A requires
package B of version 1.0, while package C requires package B at version 2.0), or it could also mean
that the dependencies on your system are missing or the incorrect version.

Whole dependency management systems were written to try to tackle this problem.  For example,
pkg-config for C(++) shared libraries/object, ant/maven/gradle/ivy for Java dependency loading and
classpath/modulepath configuration, etc etc.

When I first started learning docker, I first struggled with the concept until I realized it was
just a kind of abstraction around a process which carried around a file system with it.  But I also
tended to think of it in terms of running applications.  Afterall, one of the selling points was
that an image was self-contained: it had all the dependencies and runtimes needed to run the
application.  Ever built a python, java, javascript, etc app at work or school, and wanted to have
other people use it?  Well, first, you had to tell them "oh yeah, before you run my application, you
need to install python.  How do you do that?  Well, first you have to....".  Now imagine some of
your dependency libraries are tucked away in a private repository.  Like maybe you have JFrog
serving up a private npm, maven, pypi etc repo.  Now you have to tell your users how to access the
private repo.

Ughhh, what a pain.

So that's the first thing that containers solved.  Of course, it's also why we are seeing a
resurgence of "build to native" languages, like rust, go, nim, crystal and even haskell.  To be
honest, I always felt like docker containers were a bit of a hack to get around the popularity of
interpreted languages (and yes, I am looking at you java, your bytecode still needs to be fed into a
JVM to spit out the actual machine code).  But containers still have their uses even with "build to
native" languages.

There are 2 other use cases that come to mind:

- Creating a container that handles building a project
- Applications that require more than just a binary

### Your build environment as a container

In order to build a rust project, what do you need?

- The rustc compiler
- Tools like cargo

You may think, ha! That's all I need.  But is it?  Some rust libraries link to native libraries.
The most notable example of this is the openssl crate which leverages the system openssl.lib.  Some
crates will also attempt to link or build C code.  In that case, you now need a C(++) compiler and
set of tools like make, autoconf, pkg-config, ld, nm, ar, etc.

Or, what if you have a webassembly project?  Now you need to install wasm-pack (well, technically
you don't, but good luck setting it up manually).  If you have a webassembly project, you will need
npm installed too.  Oops, to install npm, I need node.

Maybe you're getting the picture.  There's a lot of stuff that you have to install and get right.
We can automate this process by creating a Dockerfile that builds our...well, build environment.

Let's do that now by creating a Dockerfile, but we're going to put it somewhere special:

```Dockerfile
FROM fedora:31 as builder

RUN dnf update -y \
    && dnf upgrade -y \
    && dnf install -y curl openssl-devel \
    && dnf groupinstall -y "Development Tools" \
    && dnf groupinstall -y "C Development Tools and Libraries" \
    && dnf clean all -y \
    && mkdir -p /apps/vision

# TODO: create a khadga user and run as that user.  We don't need to run as root

# Install rust tools
# Strictly speaking this isn't necessary for runtime.  Adding these will create 
# a container that lets us build khadga
RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs > rustup.sh \
    && sh rustup.sh -y \
    && source ~/.cargo/env \
    && rustup update \
    && cargo install wasm-pack

# Install node tools
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.2/install.sh | bash
RUN source ~/.bashrc \
    && nvm install 13 \
    && nvm use 13 \
    && mkdir -p /apps/bundle \
    && mkdir -p /src

# TODO: either clone the remote repo, or copy the entire khadga project over.
# In the "build locally" version, we assume that we've built khadga on our dev machine
# but we could also build it in the docker container itself.  Instead of doing a COPY
# we could do a MOUNT
COPY ./khadga /src/khadga
COPY ./vision /src/vision
COPY ./noesis /src/noesis
COPY ./build-docker.sh /src
WORKDIR /src

# TODO: If we need to publish a new noesis library, we need to 

RUN source ~/.bashrc \
    && source ~/.cargo/env \
    && ./build-docker.sh \
    && ls -al

RUN rm -rf /src

FROM fedora:31

RUN mkdir -p /apps/vision

COPY --from=builder /apps/vision /apps/vision

WORKDIR /apps/vision

CMD [ "./khadga" ]
```

We want to put this Dockerfile in a `dockers/build` folder.  So your folder should look something like
this:

```text
khadga/
- docker/
	- build/
		- Dockerfile
- khadga/
- noesis/
- vision/
```

We don't actually have any source code yet so we can't run this yet.  But I'll show you how to run
this.

```bash
cd /path/to/top-level/khadga
sudo docker build -f ./docker/builder/Dockerfile -t stoner/khadga .
```

So for example, if my khadga is in `/home/stoner/Projects/khadga`, I would do:

```
cd /home/stoner/Projects/khadga
sudo docker build -f ./docker/builder/Dockerfile -t stoner/khadga .
```

Running this command will tell docker to use the special Dockerfile we just created and to tag it
with stoner/khadga.  If you don't use the -f option, docker will look in the current directory for a
Dockerfile.  Since we are going to have another Dockerfile in the root khadga directory, that's why
we created and saved this Dockerfile somewhere special.

### What does it do?

Basically the above Dockerfile will create a Fedora31 base image, update it with the latest goodies,
and then install our tooling needed to build all the khadga subprojects (including the khadga
backend, the noesis webrtc wasm library, and the vision front end).

Of note is that we use the `FROM <base-image> as <phase-name>`.  Usually, when you see the `FROM`
command in a Dockerfile, you don't see the `as` part.  We use this when we have a multiphase build.
Notice that further down almost at the bottom, we say again `FROM fedora:31`.  This means that at this
point docker can throw away the preceeding layers and start with the resulting image as a base
image.  However, docker still "remembers" the old image.  That's why you later see the command:

```dockerfile
COPY --from=builder /apps/vision /apps/vision
```

The `--from=builder` part says, "from our previous phase called builder, copy the /apps/vision
directory to our current image's /apps/vision directory.  If we didn't use this phased approach, the
resulting image would have been about 2.6GB in size.  But by using the phased approach, the size was
less than 1/10th the size at 203MB.

The other thing to note is the `build-docker.sh` script that gets called.  This is what actually
builds the subprojects.  It will look like this:

```bash
ls -al
cd noesis
wasm-pack build
if [ "${PUBLISH}" = "true" ]; then
  wasm-pack test --firefox
	echo "Deploying noesis to npm"
	npm login
	wasm-pack publish
fi

cd ../vision
rm -rf node_modules
npm install
npm run clean
npm run build
npx jest

cd ../khadga
cargo build --release
# KHADGA_DEV=true cargo test

# Copy all the artifacts
cd ..
echo "Now in ${PWD}"
ls -al khadga
cp -r vision/dist /apps/vision/dist
cp -r khadga/config /apps/vision/config
cp -r khadga/target/release/khadga /apps/vision
```

As we build up our code, we will go into more detail on what's happening here.