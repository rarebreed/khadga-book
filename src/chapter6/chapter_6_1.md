# Websocket chat service

One of the important duties of the khadga backend is to provide a websocket connection from the
browser client.  This allows khadga to act as a kind of messaging broker.  Here, we will describe
how the warp websocket chat service works

## The warp framework

We will be using the warp web framework for rust.  Originally the code was written using actix-web
but after all the turmoil around that project, I decided to to go with the warp framework instead.

The warp framework centers around the idea of Filters, which are similar to other frameworks concept
of a `middleware`.  A Request comes to the warp server, upon which it will dissect the http request.
Depending on how you code up your Filter, it can do things like inspect the url path (ie. the
route), strip off query params, or grab a POST's message body.  It can also do things like create a
http websocket upgrade.

To create a traditional route in warp, you compose Filters together.  What do I mean by _compose_?
In functional programming, composition is very similar to the unix concept of piping.  The output of
one function or command then becomes the input to another function/command.  There is a slight trick
here in warp.  Since Filter's can be asynchronous, and async functions in rust are lazy, when you
compose your functions, what you are really doing is creating one bigger function.  This function
which is made up of the composed functions acts like a pipeline.  However, you don't actually feed
anything into the pipeline, until you run it inside a task.

The task in this case for warp is a task executor.  Let's show an example:

```rust
use warp::{ws,
           ws::Ws,
           Filter};

/// This is a map of users to a tokio mpsc channel
///
/// It is wrapped in an Arc so that we can share it across different runtime executors which might
/// happen since we are using tokio.  The Mutex makes that only one Executor thread can access the
/// HashMap storing the data at a time.
///
/// The key is a username, and the value is a mpsc transmit side channel 
type Users = Arc<Mutex<HashMap<String, mpsc::UnboundedSender<Result<ws::Message, warp::Error>>>>>;

#[tokio::main]
pub fn main() {
	  // Create our shared state
	  let users: Users = Arc::new(Mutex::new(HashMap::new()));
		let users2 = warp::any().map(move || users.clone());  // Turn the state into a Filter
		
	  // This is the main chat endpoint.  When the front end needs to perform chat, it will call
    // this endpoint.
    let chat = warp::path("chat")  // First Filter that looks for /chat in url
        .and(warp::ws())           // If it has /chat, then return a Filter that uses a Websocket
        .and(warp::path::param().map(|username: String| username)) // gets user in /chat/<user>
        .and(users2)  // Pass the state Filter to our composed function
        .map(|ws: Ws, username: String, users: Users| {  // This is where we do something
            println!("User {} starting chat", username);
            ws.on_upgrade(move |socket| user_connected(socket, users, username))
				});
				
		let log = warp::log("khadga");  // For logging
    let app = chat.with(log);       // compose our logging Filter with logging

    warp::serve(app).run(([127, 0, 0, 1], 7001)).await;
    println!("Ended service");
}
```

We start by creating some shared state in our server's main function.  We will pass this along to
any other service endpoints that need to make use of it.  The chat composed Filter starts off with a
`warp::path` Filter.  When the server gets an HTTP request, it will look at the path of the url.  If
the path contains `/chat` then it will pass the request on down to the next Filter.  Next, we see
that we compose the resulting Filter with a `.and` method.  Filters have several composition
methods, and the ones you will use the most often are _and_, _or_, _with_, and _map_.  The _and_ and
_or_ methods are just like their boolean logic equivalents.  Since we used _and_, that means the
http request must start with `/chat` in the URL, otherwise this chat composed Filter function will
stop and we won't use it.

So, assuming the request does have `/chat` in the route, then we create a WebSocket Filter.  This
will pass a warp WebSocket down so that when we get to the `.map` method, it becomes the first
argument to the `./map` closure.  Next, we try to pull off a path parameter (as opposed to a query
parameter).  If this succeeds, we continue down the composition chain, and this path param will be
passed as the second argument to `.map`.  Next, we pass our shared state which was turned into a
Filter earlier.

Finally, assuming all the above matched and was successful (the server created the WebSocket, and
there was a path param to extract), we get to the `.map` method.  As mentioned above, the arguments
that get passed to the closure are the same as the arguments in our chain...and in the same order.
So the first arg is the websocke, the second is the path parameter, and the third arg is the shared
Users state.  In the body of the closure, we perform an HTTP upgrade request to convert from http to
ws protocol.

This `ws.on_upgrade` method takes a closure which takes a socket as an argument and some function
that returns a Future that returns `Future<Output = (), ()>`.  Next we will go over how the
`user_connected` function works.

## User connection handler