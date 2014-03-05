![yeoman-vnc-angular][1]

In this blog post I’m going to show you how to build a VNC client using 
[AngularJS][2] and [Yeoman][3]. The source code used in the post is available
at my[GitHub][4]. Click [here][5] to see the final result.

It seems I have affinity to the remote desktop protocols, because this is my
third project at[GitHub][6], which implements one (
[VNC client on 200 lines of JavaScript][7], [VNC client for Chrome DevTools][8]
[VNC client with AngularJS][4]).

Anyway, lets begin with the tutorial.

## Architecture

First, lets take a look at our architecture:

![angular-vnc][9]

We should have a VNC server on the machine we want to control. This machine
provides interface accessible through the[RFB protocol][10]. The proxy in the
middle has RFB client, which knows how to talk to the RFB server. The proxy also
provides HTTP server, which is responsible for serving static files to the 
client and also allows communication through[socket.io][11]. The last component
in our diagram is the “AngularJS VNC client”, which consists few HTML and 
JavaScript files provided to the browser by the proxy. This is what actually the
user of our VNC client sees. He or she use the form provided in the “AngularJS 
VNC client” in order to enter connection details and connect to the machine he 
or she wants to control

## Proxy

*If you’re not interested in the Node.js stuff, you can [skip it][12] and just
copy and paste the code for the proxy from[GitHub][13].*

We can now continue with our proxy server. Create a directory called 
`angular-vnc`. Inside it create another one called `proxy`:

|  | mkdirangular-vnc

Now lets install all the dependencies required for our proxy server:

|  | # Required for the communication with the AngularJS client

All dependencies are installed, now your `package.json` should look something
like this:

|  | "name":"js-vnc",

We will put our proxy in folder called `lib`, located inside 
`angular-vnc/proxy`:

|  | mkdirlib

Note that we created one more file in the root directory of our proxy, we will
use it to start the proxy server.

Add the following content inside `index.js`:

|  | varserver=require('./lib/server');

Using CommonJS we require our server, located inside the `lib` folder and call
its`run` method. In order to make this code work, we should define `run` method
in`./lib/server.js`:

|  | varclients=[],

In the snippet above we create new Express application, with static directory
`/../../client/app` and also start HTTP server, listening on port `8090`.  
As next step we wrap the HTTP server with socket.io, add event handler for
incoming socket.io connections and initialize variable called`clients`. In the
connection handler we add one more event handler, which is invoked when the 
AngularJS application sends`init` message. When `init` message is received, we
add three more handlers, which are responsible for handling the incoming mouse, 
keyboard and disconnect events.

Lets take a quick look at `createRfbConnection`:

|  | functioncreateRfbConnection(config,socket){

With the received by the client configuration we initialize new RFB connection
and add some event handlers. Note that you should require`RFB` in `server.js`,
in order to make the script work:

`addEventHandlers` adds event handlers, which should handle RFB events, like
errors and new incoming frames.

|  | functionaddEventHandlers(r,socket){

Lets take a look at the handler of the `raw` event. It takes very important
role in the initialization of the connection. When we receive a`raw` event for
first time the value of`initialized` is `false`, because of the way boolean
expressions are being evaluated (see[Lazy evaluation][14]), `!initialized` is
evaluated to`true`, which leads to the call of `handleConnection`. 
`handleConnection` is responsible for notifying the AngularJS client that now
we are connected to the VNC server. It also adds the client to the`clients`
collection and changes the value of`initialized` to `true`, so when next time
we receive new frame this won’t lead to call of`handleConnection`.

In other words, 
`!initialized && handleConnection(rect.width, rect.height);` is short
version of:

|  | if(!initialized){

The next method of our proxy, we are going to look at is `encodeFrame`:

|  | functionencodeFrame(rect){

Don’t forget to require `node-png`:

|  | varPng=require('../node_modules/png/build/Release/png').Png; |
||

All that `encodeFrame` does is reordering the received pixels and converting
them to PNG. The handler of the`raw` event converts the binary result, received
by`encodeFrame` to base64, in order to make it readable for the AngularJS
client.

And the last method in the proxy is `disconnectClient`:

|  | functiondisconnectClient(socket){

As its name states it disconnects clients. This method is called with socket.
First, it finds the client, which corresponds to the given socket, ends its RFB 
connection and removes it from the`clients` array.

And now we are done with the proxy! Lets continue with the fun part, AngularJS
and Yeoman!

## AngularJS & Yeoman VNC client {#angular-vnc}

First, of all you will need to install Yeoman, if you don’t already have it
on your computer:

|  | # Installs Yeoman

Now we can begin! Inside the directory `angular-vnc` create a directory called
`client`:

|  | cdangular-vnc

Yeoman will ask you few questions, you should answer as follows:

![Yeoman AngularJS VNC configuration][15]

We are going to use Bootstrap and `angular-route.js`. Wait few seconds and all
required dependencies will be resolved.

Look at: `app/scripts/app.js`, its content should be something like:

|  | 'use strict';

Now in the `client` directory run:

After the command completes, the content of `app/scripts/app.js`, should be
magically turned into:

|  | 'use strict';

For next step, replace the content of `app/views/main.html` with:

|  | <div class="container">

You should also insert some CSS at `app/styles/main.css`:

|  | .colorgraph {

This defines the markup and styles for simple Bootstrap form.

After you start the proxy:

and open <http://localhost:8090>, you should see something like this:

![VNC Login Form][16]

The awesome thing is that we already have validation for the form! Did you
notice that we added selector`form.ng-invalid.ng-dirty input.ng-invalid`?
AngularJS is smart enough to validate the fields in our form by seeing their 
type (for example`input type="number"`, for the port) and their attributes (
`required`, `ng-minlength`). When AngularJS detects that any field is not valid
it adds the class:`ng-invalid` to the field, it also adds the class 
`ng-invalid` to the form, where this field is located. We, simply, take
advantage, of this functionality provided by AngularJS, and define the styles:
`form.ng-invalid.ng-dirty input.ng-invalid`. If you’re still not aware how
the validation works checkout[Form Validation in NG-Tutorial][17].

We already have attached controller, to our view (because of Yeoman), so we
only need to change its behavior.

Replace the content of `app/scripts/controllers/main.js` with the following
snippet:

|  | 'use strict';

The most interesting part of `MainCtrl` is the `login` method. In it, we first
check wether the form is invalid
(`form.$invalid`), if it is we make the it “dirty”. We do this in order to
remove the`ng-pristine` class from the form and force the validation. This
scenario will happen if the user does not enter anything in the form and press 
the “Login” button. If the form is valid, we call the`connect` method of the
service`VNCClient`. As you see it returns promise, when the promise is resolved
we redirect the user to the page<http://localhost:8090/#/vnc>, otherwise we
show him or her the message:`'Connection timeout. Please, try again.'` (
checkout`<div class="form-error" ng-bind="errorMessage"></div>`).

The next component we are going to look at is the service `VNCClient`. Before
that, lets create it using Yeoman:

|  | yo angular:service VNCClient |
||

Now open the file: `app/scripts/services/vncclient.js` and place the following
content there:

|  | 'use strict';

I know it is a lot of code but we will look only at the most important methods
. You might noticed that we don’t follow the best practices for defining 
constructor functions – we don’t add the methods to the function’s prototype. 
Don’t worry about this, AngularJS will create a single instance of this 
constructor function and keep it in the services cache.

Lets take a quick look at `connect`:

|  | this.connect=function(config){

`connect` accepts a single argument – a configuration object. When the method
is called it creates new socket using the service`Io`, which is simple wrapper
of the global`io` provided by socket.io. We need this wrapper in order to be
able to test the application easier and prevent monkey patching. After the 
socket is created we send new`init` message to the proxy (do you remember the
init message?), with the required configuration for connecting to the VNC server.
We also create a connection timeout. The connection timeout is quite important, 
if we receive a late response by the proxy or don’t receive any response at all.
The next important part of the`connect` method is the handler of the response
`init` message, by the proxy. When we receive the response within the
acceptable time limit (remember the timeout) we resolve the promise, which was 
instantiated earlier in the beginning of the`connect` method.

This way we transform a callback interface (by socket.io) into a promise based
interface.

This is the implementation of the `addHandlers` method:

|  | this.addHandlers=function(success){

Actually we add a single handler, which handles the `frame` events, which
carries new (changed) screen fragments. When new frame is received we invoke the
`update` method. It may look familiar to you – this is actually the 
[observer pattern][18]. We add/remove callbacks using the following methods:

|  | this.addFrameCallback=function(fn){

And in `update` we simply:

|  | this.update=function(frame){

Since we need to capture events in the browsers (like pressing keys, mouse
events…) and send them to the server we need methods for this:

|  | this.sendMouseEvent=function(x,y,mask){

The [VNC screen][19] directive is responsible for calling these methods. In the
`sendKeyboardEvent` we transform the `keyCode`, received by handling the
keydown/up event with JavaScript, to one, which is understandable by the RFB 
protocol. We do this using the array`keyMap` defined above. 

Since we didn’t create the `Io` service, you can instantiate it by:

And place the following snippet inside `app/scripts/services/io.js`:

|  | 'use strict';

Don’t forget to include the line:

|  | <script src="/socket.io/socket.io.js"></script> |
||

in `app/index.html`.

And now, the last component is the VNC screen directive! But before looking at
it, replace the content of`app/views/vnc.html` with the following markup:

|  | <div class="screen-wrapper">

as you see we include our VNC screen completely declaratively: 
`<vnc-screen></vnc-screen>`. In the markup above, we have few
directives:
` ng-show="connected()", ng-click="disconnect()", ng-hide="connected()"`, they
has expressions referring to methods attached to the scope in the`VncCtrl`:

|  | 'use strict';

`VncCtrl` is already located in `app/scripts/controllers/vnc.js`. You don’t
have to worry about it because when we instantiated the`vnc` route, Yeoman was
smart enough to create this controller for us.

Now lets create the VNC screen directive:

|  | yo angular:directive vnc-screen |
||

…and now open `app/scripts/directives/vnc-screen.js`. This is our directive
definition:

|  | varVNCScreenDirective=function(VNCClient){

The show is in the link function in, which we will look at later. Now lets take
a quick look at the other properties of the directive. The template of our 
directive is simple canvas with class`vnc-screen`, it should replace the
directive. We define that the user of the`vnc-screen` directive should use it
as element. It is also quite important to notice that we have a single 
dependency – the`VNCClient` service, we described above.

Now lets look what happens in the link function:

|  | if(!VNCClient.connected){

As first step the link function checks whether the `VNCClient` is connected, if
it isn’t the directive simply adds the text`"No VNC connection."` and hides the
template (actually now a DOM element). We can also take more advanced approach 
here, we can watch the`connected` property and undertake different actions
depending on its value. Doing this will make our directive more dynamic. But for
simplicity lets stick to the current implementation.

The line 
`var bufferCanvas = createHiddenCanvas(VNCClient.screenWidth, VNCClient.screenHeight)`
`VNCClientScreen` with parameter the hidden canvas. The constructor function 
`VNCClientScreen` wraps the canvas and provides method for drawing on it:

|  | functionVNCClientScreen(canvas){

As next step we create new instance of `Screen`. This is the last component we
are going to look at in the current tutorial but before taking a look at it lets
see how we use it.

We instantiate the `Screen` instance by passing our “visible” canvas and
the VNC screen buffer (the wrapper of the “hidden” canvas) to it. For each 
received frame we are going to draw the buffer canvas over the VNC screen. We do
this because the VNC screen could be scaled (i.e. with size different from the 
one of the remote machine’s screen) and we simplify our work by using this 
approach. Otherwise, we should calculate the relative position of each received 
frame before drawing it onto the canvas, taking in account the scale factor.

In the `frameCallback` we draw the received rectangle (changed part of the
screen) on the buffer screen and after that draw the buffer screen over the
`Screen` instance.

In the link function we also invoke the methods:

*   `addKeyboardHandlers`
*   `addMouseHandler`

They simply delegate handling of mouse and keyboard event to the `VNCClient`.
Here is the implementation of the`addKeyboardHandlers`:

|  | Screen.prototype.addKeyboardHandlers=function(cb){

Now you also see why we used `VNCClient.sendKeyboardEvent.bind(VNCClient)`,
because in`keyDownHandler` and `keyUpHandler` we invoke the callback with
context`null`. By using `bind` we force the context to be the `VNCClient`
itself.

And we are done! We skipped some of the methods of `Screen` because I think
their consideration is not that essential for the purpose of this tutorial. 
Anyway, here is the whole implementation of the`Screen` constructor function:



| 100

The last step is to run the VNC client! Make sure you have a computer with VNC
server on it.

Run the following command:

|  | cdangular-vnc

Now open the url: <http://localhost:8090>, and rock!

## Demo {#vnc-demo-video}

 [1]: img/yeoman-vnc-angular.png
 [2]: http://angularjs.org/
 [3]: http://yeoman.io/
 [4]: https://github.com/mgechev/angular-vnc

 [5]: http://blog.mgechev.com/2014/02/08/remote-desktop-vnc-client-with-angularjs-and-yeoman/#vnc-demo-video
 [6]: https://github.com/mgechev
 [7]: https://github.com/mgechev/js-vnc-demo-project
 [8]: https://github.com/mgechev/devtools-vnc
 [9]: img/angular-vnc.png
 [10]: https://en.wikipedia.org/wiki/RFB_protocol
 [11]: http://socket.io/

 [12]: http://blog.mgechev.com/2014/02/08/remote-desktop-vnc-client-with-angularjs-and-yeoman/#angular-vnc
 [13]: https://github.com/mgechev/angular-vnc/tree/master/proxy
 [14]: https://en.wikipedia.org/wiki/Lazy_evaluation
 [15]: img/Screen-Shot-2014-02-08-at-19.29.28.png
 [16]: img/Screen-Shot-2014-02-08-at-20.43.44.png

 [17]: http://ng-tutorial.mgechev.com/#?tutorial=form-validation&step=basic-validation
 [18]: https://en.wikipedia.org/wiki/Observer_pattern

 [19]: https://github.com/mgechev/angular-vnc/blob/master/client/app/scripts/directives/vnc-screen.js