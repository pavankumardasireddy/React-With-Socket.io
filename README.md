## Combining React with Socket.io for real-time goodness
This post isn’t heavy on React, so the core concepts should translate easily to other view frameworks like Vue and Angular. That said, for the rest of the post, I’ll assume you are using React or React-Native.

So you need to make your app respond to events from the server. Usually this is referred to as real-time, meaning you aren’t going to rely purely on the client to decide when to get data from the server, instead the server will stream data down to the client as it becomes available. Data gets pushed to the client from the server, instead of being pulled from the client.

This sort of model is perfectly suited for a variety of applications, for example chat, games, trading, etc. That said, more and more teams are adopting it for use cases where it’s not required for the product to work, but rather to make the product feel more responsive.

Regardless of what your reasons are, I think it’s safe to assume that you are reading this post because you are interested in seeing how to combine sockets and React to make your app more real-time.

This post will use a super simple example, where we publish the current time from the server socket, subscribe to it on the client and render the value as it gets updated.


## Setting up the basics
To get started, I’m going to assume that you already have the basics installed like node and npm.

If you don’t have it already, you’ll have to install create-react-app to get started. This is easily done by running the following command to install it      ``` globally: npm --global i create-react-app```

Now you can go ahead and generate the app we’ll use to play around with sockets by running he following command: create-react-app socket-timer

After the app has been generated, open up the folder with your favorite text editor. To run the project, all you need to do is run npm start in the app folder.

We’re going to be running both the server and client in the same codebase, which you would probably not be doing in a production app, but it’ll make it easier to follow along.

## Socket.io on the server
Let’s create a websocket service quickly. To do this, drop into a terminal in your app folder, and install socket.io: npm i --save socket.io

Now that you have socket.io installed, create a file called server.js in the root folder of the app you generated earlier. And inside of this file, you can type up the following code to import and construct the socket.


Ok, so we just import it and construct it. We’re going to be using the io variable to do our socket magic.

Websockets are long running duplex channels between the client and server. So the most important thing to do on the server is to handle a connection of a client, so that you can publish (emit) events to that client. So let’s do that:


Once you’ve done that, you also need to tell socket.io to start listening for clients.


Great! So now you can drop into your terminal and start up the server by running node server . You should see a message that it started up successfully: *`listening on port 8000`* .

At the moment, the socket service isn’t doing anything. You have access to client sockets, but you’re not emitting anything to them yet. Because you have access to a connected client in this scope, you can respond to events being emitted from the client. Think of it as a server side event handler pertaining to a specific event from a specific client.

In the end, we want the service to start an interval (timer) and emit the current date back to the client. We want the service to start a timer per client, and the client can pass in the desired interval time. That’s an important point, because it means that clients can send data through to the server socket. Change your code to add the following:


Ok, so now you have coded up the basic structure to handle a client connection and for handling said client emitting an event to start a timer for it. Now you can actually start up the timer, and start emitting events back to the client containing the current time, so change your code to look like this:


Goody! So you’ve fired up a socket and started listening on it for clients. When a client connects, you have a closure where you can handle events from a particular client. You also handle a specific event, subscribeToTimer , being emitted from the client where you start a timer. When your timer fires, you emit an event, timer , back to the client.

At this point, your server.js file should look like this:


That’s it for the server side of our socket.io setup! Let’s move on to the client. Just make sure that you have your latest server code running by running node server in a terminal. If you started this up before you made your latest changes, just restart it.

## Socket.io on the client
You started up the React app earlier by running npm start on the command line. So you should be able to go into your code, make changes, and see the browser reloading your app with the changes.

First thing you want to do now is wire up the client socket code that’ll communicate with the server side socket to start a timer, by emitting subscribeToTimer , and consume the timer events emitted by the server.

To do this, create a file in the src folder called api.js. In the api.js file, you want to create a function that can be called to emit the subscribeToTimer event to the server and feed back the results of the timer event to the consuming code.

You can start by defining the function, and exporting it from the module:
```
        function subscribeToTimer(interval, cb) {
        } 
        export { subscribeToTimer }
```
So we’re basically opting to use a node style function, where the caller of this function can pass in an interval for the timer on the first parameter, and a callback function on the second parameter.

Because we want to communicate to the socket on the server, we need to install the socket.io client library. On the command line,
 install it using npm: 
         npm i --save socket.io-client .

And now that you have installed it, you can go ahead and import it. Here we can use the ES6 module syntax, because we are running client side code that’ll get transpiled with Webpack and Babel. You can also construct the socket by calling the main export function from the socket.io-client module, providing a port, which in our case is 8000 .
```
        import openSocket from 'socket.io-client';
        const socket = openSocket('http://localhost:8000');
```
On the server, we wait for a client to emit the subscribeToTimer event to the server before we start a timer and then emit the timer event to the client every time the timer fires.

So what we need to do now is first to subscribe to the timer event coming from the server, and then emit the subscribeToTimer event. Every time we receive a timer event from the server, we’ll call the callback function with the result:
```
        import openSocket from 'socket.io-client';
        const  socket = openSocket('http://localhost:8000');
        function subscribeToTimer(cb) {
          socket.on('timer', timestamp => cb(null, timestamp));
          socket.emit('subscribeToTimer', 1000);
        }
        export { subscribeToTimer };
```
Note that we subscribe to the timer event on the socket, before we emit the subscribeToTimer event. We did this in case we run into a race condition where timer events are being emitted from the server, but the client hasn’t shown it’s interest in it yet, causing events to go missing.

## Using the events in a React component
Now you should have a file called api.js on the client side that exports a function that you can call to subscribe to timer events. Next, you’ll see how you can use this function in your React component to render the timer value every time the server emits it.

At the top of the App.js file that was generated by create-react-app, import the api that we created earlier.

        ```
        import { subscribeToTimer } from './api';
        ```
Now that you’ve done that, you can go ahead and add a constructor for the component and inside of this constructor, call the subscribetoTimer function that you get from the api file. Every time that you get an event, just set a value called timestamp on state using the value that came through from the server.

        ```
        constructor(props) {
          super(props);
          subscribeToTimer((err, timestamp) => this.setState({ 
            timestamp 
          }));
        }
        ```
Because you are going to use the timestamp value on state, it makes sense to add a default value for it. Add this bit of code just below your constructor to set some default state:

        ```
        state = {
          timestamp: 'no timestamp yet'
        };
        ```
And now you can update your render function to render the timestamp you’ve set on state in the event handler.

        ```
        render() {
          return (
            <div className="App">
              <p className="App-intro">
              This is the timer value: {this.state.timestamp}
              </p>
            </div>
          );
        }
        ```
And that’s it! In the browser, you should be able to see the events coming from the server and being rendered inside of your React component.


## Summary
I’ve used this pattern a lot in my code, and it works well, and you can scale it to handle more complexity, without running into issues. That said, there is quite a bit more to developing real-time apps with React and Socket.io. I’ll be sharing a bunch of articles with you on this topic, even some on using RethinkDb to handle scaling out your sockets.


## Follwed this link:
    https://medium.com/dailyjs/combining-react-with-socket-io-for-real-time-goodness-d26168429a34