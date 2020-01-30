# Introduction to khadga

This book is a small guide on how to create a web application using react, typescript, and
webassembly generated from rust.  It will take you from knowing nothing about how to create wasm to
how to generate both the front and back end code for your app.

The application that is generated here is a small chat application that will set up WebSocket
connections between two or more clients and the central web server.

If you're wondering where the name khadga comes from, it's a Sanskrit word meaning sword.  It is
often referred to in spiritual or mystical concepts as the sword that cuts away illusion.

## Why?

One might ask why go through this?  If the main point of this app is to do a chat application with
video, there's a dime-a-dozen ready made apps for that.

Basically, I wanted to write a non-trivial application from top to bottom.  A truly _full-stack_
application where the front end, the back end, the database, the CI deployments, integration with
IoT sensor data, and the data analysis is all done by one developer (that'd be me, and hopefully you
the reader as well).

Yes, it's a tall order.  Although the primary purpose is to use this project as a vehicle for
learning how to do deep learning, my intention is to use khadga as a learning tool.  Not just for
myself but for others as well.  What I have learned is that most books don't walk you through a
non-trivial project from beginning to end.  Or, they might show you how to use some framework, but
not always why.

My hope is that by me forging ahead and suffering the learning pains, others can follow along and
avoid the mistakes I made.  Because of the pedantic nature of this project, I will endeavor to do
the following:

- Write documentation for all my code
- Keep this book up to date
- Show you every step of building a project (as much as possible)
  - Writing unit and integration tests
  - Using CI/CD to build, test, and deploy an application (on openshift)
- Try not to cut corners (ie, I will try to use `.unwrap()` or `.expect()` as little as I can)

## What you will learn

This guide will walk you through everything required to develop, test and deploy both the front and
backend application. This includes:

- How to write an asynchronous web server using rust's warp framework
- How to use wasm-bindgen, web-sys and js-sys crates to create a wasm npm module and publish it
- How to serve your single page app from the async web server
- How to create a react+redux front end, using WebRTC and WebSockets modules written in webassembly
- How to deploy your app to Openshift Online using docker
- How to set up unit and integration tests for the front and back end
- How to set up a CI pipeline between your deployment and your tests using travis-ci

### What the app does

We will build the functionality of the app slowly in order to make it useful early on, but to build
up functionality as we go.

- Chat application
- WebRTC for cam-to-cam video conferencing
- Video recognition

First, we will start with a relatively simple chat application.  It will also cover things like
setting up a database of users and saved chat history.  It will also showcase how to do user
authentication and authorization.

Next, we will write a webassembly module that interacts with WebRTC and WebSocket APIs.  We will use
this so that the webassembly can quickly and efficiently store data into tensor format that we can
hand back to tensorflowjs.

Then, we will enhance the app so that it will do video as well as text based chatting.  In this
step, we will add a signaling service to the bacnd, so peers can find one another.  Chats can either
be saved locally or stored on the discovery server. This step will also show how to encrypt the
streams for end to end encryption.

Lastly, we will build on the video streams enabled by the WebRTC to do image recognition.  We will
use this as a project to detect faces that are displayed and see if it is a known user.

## Prerequisites

This chapter will cover how to scaffold and generate the initial wasm-pack code as well as a basic
asynchronous web server but it assumed that the reader is already familiar with:

- basic rust
- basic html
- basic css
- basics of docker

The reader's rust skills should be at a basic level.  One of the goals of the book is to explain
some of the more tricky rust concepts (or at least tricky to the author).  A basic understanding of
rust ownership and lifetimes, and how closures work will be required.  Understanding how Traits
work, especially AsRef and From/Into will also be useful. The async/await portions will be described
in detail however.

The reader should have a basic level of HTML5 understanding.  This includes knowing what the DOM is
and the basics of common tags.  Most of the front end will be calling the web-sys bindings to the
Web APIs, so it is useful but not required to know javascript.  Because we are creating a single
page app, knowledge of CSS will be useful, as the book will not spend a lot of time explaining much
of the CSS content.

For deployment, we will be using Openshift Online.  There is a free tier available and you can use
this to deploy the application.  In order to get the server to the cloud, we will need to create a
container and therefore a docker image. This will not be advanced, but there will not be a lot of
explanation of what the docker file does.

## Caveats

The biggest caveat is the author is new to this himself.  The decision to write the book was to help
others so they do not have to learn the hard way like the author did.  Also, the author is at a
basic to intermediate level understanding of rust.  There could very well be a better way to write
the code and if so, please make a PR and contribute!

The second caveat was that this project made an opinionated stance on the technology used.  First
and foremost was the desire to use the new async/await syntax.  This lead to several _problems_.
For example, since async/await is still new, documentation is scarce.

### Why not yew?

Seasoned developers might ask why [yew][-yew] was not used for this project.  The simple answer here
is that there was a desire to use wasm-bindgen, web-sys and js-sys crates to create the app.

The author has no working knowledge of yew, and it was considered initially.  Afterall, it seems to
tick off a lot of the right boxes:

- Built in virtual DOM
- Macros to generate JSX like code
- Concurrency
- Safe state management

However, yew is built on a crate called stdweb instead of wasm-bindgen, web-sys and js-sys.  The
main difference is that those libraries are created and maintained by the official Web Assembly
Working Group.  It will therefore be more up to date and have "official" support.  It was also
designed to be language agnostic and the bindings are auto-generated from the WebIDL schema.

### Why not percy?

There is another framework that looked promising called [percy][-percy].  It also has a virtual DOM
and macro generator to create some JSX-like code. Unlike yew, it is using wasm-bindgen and web-sys
and js-sys crates. The problem with percy is that because of some of the macros, it required a
nightly toolchain.

Although nightly is great for individual learning and experimentation, it's not the best for
teaching others.  The brittleness of nightly means that what may compile one day for one person may
not compile for another person (or the same person!) on another day.  It can also be hard sometimes
to find a nightly build that allows all dependency crates to be built successfully.

### Why not seed?

I discovered [seed][-seed] very recently, and while it seems to fit the bill for writing the entire
application in rust, upon some consideration, I felt that it would be more beneficial to write the
front end in react and typescript, and use wasm only for speed (or safety) sensitive areas of the
code.

This also has the advantage of being able to slowly convert existing applications to use wasm,
rather than have an all-in-one greenfield project created from scratch.

[-yew]: https://github.com/yewstack/yew
[-seed]: https://seed-rs.org/
[-percy]: https://github.com/chinedufn/percy