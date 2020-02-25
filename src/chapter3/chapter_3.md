# The main client interface

First, let's start off with a very basic webpage.  Our end goal with this app is not to create
something beautiful, but simply as a way to get data from some agent (a human, being, an IoT
sensor, or another remote service for example) so that we can work on this data in real time.

Our initial goal is to provide a user interface for a human user (or some AI agent) to post
messages similiar to slack.  It's sort of half slack/gitter, half webchat in that the messages
should be persistenet and editable, but they should also be seen in a public forum (or privately).

A second goal is to provide video camera access to allow for video using WebRTC.

The SPA running on the users browser has 2 initial services:

- Collect messages from an agent and do text classification on it
- Perform face recognition and eye tracking

## General Layout

Before we get to actually coding up anything, let's think about what we want to show the user, and
how we want the interface to be presented.

- Navbar
  - Google Login
  - Settings for app
    - video/audio enumeration
    - chat and appearance
- Chat messages window
  - Public message window
  - Message threads
  - Private message window
- Logged in users
- Video (Draggable and Resizable div component)

## State

What are some of the things we need to keep track of for our app?

- Is user logged in already? (ie, session is current and active?)
- What users have connected and logged in that we can chat with?
- Do we need to show modal for user to sign up?  login?
- Max message/text limit for chat message windows before removing from DOM
- Is there a video chat session?
- SDP data for webrtc peer connection for both offer and reply

## Navbar

TODO: Go over how the navbar is implemented in bulma.

## Chat message window

TODO: Go over how to split up into a main public column, and another column for message threads

### Public messages

TODO: Explain what the public messages are vs. threaded

### Message threads

TODO: Allow replies to a message that follows that specific message

### Private messages

TODO: How do we add new containers for private messages?

## Logged in users

TODO Show logged in users 

## Video

