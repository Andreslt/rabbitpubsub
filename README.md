

#Scaling real-time apps on Cloud Foundry (using Redis + RabbitMQ)#
<br>

## Chat App ##
One of the most common things people build on Node.js are real-time apps like chat apps, social-networking apps etc. And there are plenty of examples showing how to build such apps on the web. But it's hard to find an example that shows how to deal with real-time apps that are scaled & are running in the cloud/PaaS with multiple instances & deal with issues like app sessions, sticky sessions, scale-up/down, instance crash/restart etc in such an environment.
 
The main objective of this project is to build a simple chat app and tackle such issues. Specifically, we will be building a simple Express (server), Socket.io(browser<->server), Redis(session store) & RabbitMQ(pub-sub) based Chat app that should meet the following objectives:

1. Chat server should run on multiple instance.
2. User's login should be saved in a session.
    * If the user refreshes the browser, he should be relogged in.
    * Socket.io should get user info from the session before sending chat-message 
   * Socket.io should only connect if user is already logged in.
3. While the user is chatting, if the server to which he is connected is restarted / scaled-down, the user should be reconnected to available instance w/o bouncing him. 

***Final architecture:***

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/finalArchitecture.png" height="" width="450px" />
</p>

***Login page:***

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/chatAppPage1.png" height="" width="450px" />
</p>

***Chat page:***
<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/chatAppPage2.png" height="" width="450px" />
</p>

<br>
***Along the way, we will go over:***

1. How to use Socket.io & Sticky Sessions.
2. How to use Redis as session store 
3. How to use Redis as a pubsub service.
4. How to use sessions.sockets.io to get session info (like user info) from Express sessions.
5. How to configure Socket.io client & server to properly reconnect after one or more server instances goes down ( i.e. has been restarted / scaled down / has crashed).



## Socket.io & Sticky Sessions ##

<a href='http://socket.io/' target='_blank'>Socket.io</a> is one of the earliest & most popular Node.js modules to help build real-time apps like chat, social networking etc. in Node.js.  (PS: <a href='https://github.com/sockjs/sockjs-client' target='_blank'>SockJS</a> is another popular library similar to Socket.io).

But when you run such a server in the cloud that has load-balancer/ reverse proxy, routers etc, you need to configure it work properly especially when you scale the server to use multiple instances.

One of the constraints Socket.io and SockJS etc. have is that they need to continuously talk to the <i><b>same instance</b></i> of the server. They work perfectly fine when there is only 1 instance of the server.

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/socketio1Instance.png" height="300px" width="450px" />
</p>


<br>
<br>
<br>
But when you scale your app in a cloud environment, the load balancer will take over and starts to send the requests are sent to different instances causing Socket.io to break.
<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/socketioBreaks.png" height="300px" width="450px" />
</p>

<br>
<br>
<br>
To help in such situations, load balancers have feature called 'sticky sessions' aka 'session affinity'. The main idea is that if its set, then after the first request(load-balanced request), all the following requests will go to the same server instance.

In Cloud Foundry, cookie based sticky sessions are enabled for apps that sets cookie <b>jsessionid</b>.

So all the apps need to do is to set a cookie w/ name <b>jsessionid</b> to make Socket.io work.

```
    /*
     Use cookieParser and session middlewares together.
     By default Express/Connect app creates a cookie by name 'connect.sid'.But to scale Socket.io app,
     make sure to use cookie name 'jsessionid' (instead of connect.sid) use Cloud Foundry's 'Sticky Session' feature.
     W/o this, Socket.io won't work if you have more than 1 instance.
     If you are NOT running on Cloud Foundry, having cookie name 'jsessionid' doesn't hurt - it's just a cookie name.
     */
    app.use(cookieParser);
    app.use(express.session({store:sessionStore, key:'jsessionid', secret:'your secret here'}));
```

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/socketioWorks.png" height="300px" width="450px" />
</p>
In the above diagram, when you open the app, 

1. Express sets a session cookie w/ cookie name <b>jsessionid</b>. 
2. Then when socket.io connects, it uses that same cookie & hits load balancer
3. Load balancer always routes it to the same server that the cookie was set in.

## Sending session info to Socket.io 
Let's imagine that the user is logging in via Twitter or Facebook or we have regular login screen. And we are storing this information in a session after the user has logged in.

```javascript

app.post('/login', function(req, res) {
   //store user info in session after login.
  req.session.user = req.body.user;
  ...
  ...
});
```

And once the user has logged in, we connect to Socket.io to allow chatting. But socket.io doesn't know who the user is & he is actually logged in before sending chat messages to others.

That's where `sessions.sockets.io` library comes in. It's a very simple library, all it does is to grab session information during handshake & gives it to Socket.io's `connection` function.

```javascript
//instead of
io.sockets.on('connection', function(socket) {
 //do pubsub here
 ...
})

// with sessions.sockets.io, you'll get session info

/*
 Use SessionSockets so that we can exchange (set/get) user data b/w sockets and http sessions
 Pass 'jsessionid' (custom) cookie name that we are using to make use of Sticky sessions.
 */
var SessionSockets = require('session.socket.io');
var sessionSockets = new SessionSockets(io, sessionStore, cookieParser, 'jsessionid');

sessionSockets.on('connection', function (err, socket, session) {

  //get info from session
  var user = session.user;
  
  //Close socket if user is not logged in
  if(!user) 
  	socket.close(); 
   
  //do pubsub
  socket.emit('chat', {user: user, msg: 'logged in'});
  ...
});
```

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/sendingSession2SocketIO.png" height="300px" width="450px" />
</p>


## Redis as a session store

So far so good, but all these sesssion information is stored in Socket.io's Memory store. i.e. in-memory. So if one of the instances goes down, it will be lost. 

So we will configure our app to use Redis as session store like below.

```javascript
/*
 Use Redis for Session Store. Redis will keep all Express sessions in it.
 */
var redis = require('redis');
var RedisStore = require('connect-redis')(express);
var rClient = redis.createClient();
var sessionStore = new RedisStore({client:rClient});
    
    
  //And pass sessionStore to Express's 'session' middleware's 'store' value.
     ...
     ...  
    app.use(express.session({store:sessionStore, key:'jsessionid', secret:'your secret here'})); 
     ...

```

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/redisAsSessionStore.png" height="300px" width="450px" />
</p>


With the above configuration, sessions will now be stored in Redis. And also, if one of the server instances goes down, session will still be available for other instances to pick up.
    
<br>
##RabbitMQ as pub-sub server
So far with the above setup our sessions are taken care of but if we are using Socket.io's default pub-sub, it will work only for 1 sever instance.
i.e. if user1 & user2 are on server instance #1, they both can chat with each other. But if they are on different server instances they can't.

So we will update our server to use Redis as PubSub service (along with session-store). PS: Redis natively supports pub-sub operations. All we need to do is to create a publisher, a subscriber & a channel and we will be good. 

```javascript
//instead of using socket.io's default pub-sub..

sessionSockets.on('connection', function (err, socket, session) {
   socket.on('chat', function(data){
        socket.emit('chat', data);
        socket.broadcast.emit('chat', data);
   });

   socket.on('join', function(data){
        socket.emit('chat', {msg: 'user joined'});
        socket.broadcast.emit('chat', {msg: 'user joined'});
  });
}


//We will use Redis to do pub-sub

/*
 Create two redis connections. A 'pub' for publishing and a 'sub' for subscribing.
 Subscribe 'sub' connection to 'chat' channel.
 */
var sub = redis.createClient();
var pub = redis.createClient();
sub.subscribe('chat');


sessionSockets.on('connection', function (err, socket, session) {
   socket.on('chat', function(data){
        pub.publish('chat', data);
   });

   socket.on('join', function(data){
        pub.publish('chat', {msg: 'user joined'});
  });
  
   /*
     Use Redis' 'sub' (subscriber) client to listen to any message from Redis to server.
     When a message arrives, send it back to browser using socket.io
   */
    sub.on('message', function (channel, message) {
        socket.emit(channel, message);
    });
}

```


So the app's architecture will now look like this:
<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/finalArchitecture.png" height="" width="450px" />
</p>
<br>
<br>
## Handling server scale-down /crashing / restarting
In theory, our app will work fine as long as all the server instances are running. But what happens if the server is restarted or scaled down or one of the instances crash? How do we handle that?

But let's first understand what happens in that situation.

The below code simply connects a browser to server and listens to various Socket.io's events.

```javascript
    /*
         Connect to socket.io on the server (***BEFORE FIX***).
         */
        var host = window.location.host.split(':')[0];
        var socket = io.connect('http://' + host);

        socket.on('connect', function () {
            console.log('connected');
        });
        socket.on('connecting', function () {
            console.log('connecting');
        });
        socket.on('disconnect', function () {
            console.log('disconnect');
        });
        socket.on('connect_failed', function () {
            console.log('connect_failed');
        });
        socket.on('error', function (err) {
            console.log('error: ' + err);
        });
        socket.on('reconnect_failed', function () {
            console.log('reconnect_failed');
        });
        socket.on('reconnect', function () {
            console.log('reconnected ');
        });
        socket.on('reconnecting', function () {
            console.log('reconnecting');
        });


```
<br>
<br>
While the user is chatting, if we restart the app **on localhost or single host**, Socket.io attempts to reconnect multiple times (configuration) to see if it can connect. If the server comes up w/in that time, it will reconnect. So we see the below logs:

<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/reconnectOn1server.png" height="300px" width="600px" />
</p>

<br>
But, if the user is chatting on the same app that's running ***on Cloud Foundry AND with multiple instances***, and if we restart the server (say using `vmc restart rabbitpubsub`)
then we'll see the following log:
<p>
<img src="https://github.com/rajaraodv/rabbitpubsub/raw/master/pics/reconnectOnMultiServer.png" height="400px" width="600px" />
</p>

You can see that in the above logs, after the server comes back up, Socket.io -client(that's running in the browser) isn't able to connect to Socket.io-server(that's running in the server). 

That is because, once the server is restarted on Cloud Foundry, ***instances are brought up as if they are brand-new server instances w/ different IP addresses and different ports and so `jsessionid` in no-longer valid***  
And that in-turn causes Socket.io's reconnection requests to be ***load balanced*** (i.e. they are sent to different server instances) causing socket.io-server piece not to properly handshake and consequently to throw `client not handshaken` error!

<br>
###OK, let's fix that reconnection issue.

First, we will disable Socket.io's default "reconnect" feature. And then implement our own reconnection feature. 

In our custom reconnection function, When the server goes down, we'll make a dummy HTTP-get call to index.html every 4-5 seconds.
	      And if the call succeeds, we know that (Express) server has already set ***jsessionid*** in the response. So then we'll call socket.io's reconnect function. And this time because jsessionid is set, socket.io's handshake will succeed and the user will get to continue chatting happily.
	      
```javascript

    /*
      Connect to socket.io on the server (*** FIX ***).
    */
 	var host = window.location.host.split(':')[0];
 	
 		//Disable Socket.io's default "reconnect" feature
        var socket = io.connect('http://' + host, {reconnect:false, 'try multiple transports':false});
        var intervalID;
        var reconnectCount = 0;
		...
		...
        socket.on('disconnect', function () {
            console.log('disconnect');
            
            //Retry reconnecting every 4 seconds
            intervalID = setInterval(tryReconnect, 4000);
        });
       ...
       ...
      
        

	    /*
	      Implement our own reconnection feature. 
	      When the server goes down we make a dummy HTTP-get call to index.html every 4-5 seconds.
	      If the call succeeds, we know that (Express) server sets ***jsessionid*** , so only then we try socket.io reconnect.
	    */
        var tryReconnect = function () {
            ++reconnectCount;
            if (reconnectCount == 5) {
                clearInterval(intervalID);
            }
            console.log('Making a dummy http call to set jsessionid (before we do socket.io reconnect)');
            $.ajax('/')
                .success(function () {
                    console.log("http request succeeded");
                    //reconnect the socket AFTER we got jsessionid set
                    socket.socket.reconnect();
                    clearInterval(intervalID);
                }).error(function (err) {
                    console.log("http request failed (probably server not up yet)");
                });
        };
        
```


In addition, on the server, when the dummy-http request comes in, we will ***regenerate*** session to remove old session & sessionid and ensure everything is afresh before we serve the response.

```javascript
//Instead of..
exports.index = function (req, res) {
    res.render('index', { title:'rabbitpubsubApp',  user:req.session.user});
};

//Use this..
exports.index = function (req, res) {
    //Save user from previous session (if it exists)
    var user = req.session.user;
    
    //Regenerate new session & store user from previous session (if it exists)
    req.session.regenerate(function (err) {
        req.session.user = user;
        res.render('index', { title:'rabbitpubsubApp',  user:req.session.user});
    });
};


```

## Running / Testing it on Cloud Foundry ##
* Clone this app to `rabbitpubsub` folder
* ` cd rabbitpubsub`
* `npm install` & follow the below instructions to push the app to Cloud Foundry

```

[~/success/git/rabbitpubsub]
> vmc push rabbitpubsub
Instances> 4       <----- Run 4 instances of the server

1: node
2: other
Framework> node

1: node
2: node06
3: node08
4: other
Runtime> 3  <---- Choose Node.js 0.8v

1: 64M
2: 128M
3: 256M
4: 512M
Memory Limit> 64M

Creating rabbitpubsub... OK

1: rabbitpubsub.cloudfoundry.com
2: none
URL> rabbitpubsub.cloudfoundry.com  <--- URL of the app (choose something unique)

Updating rabbitpubsub... OK

Create services for application?> y

1: blob 0.51
2: mongodb 2.0
3: mysql 5.1
4: postgresql 9.0
5: rabbitmq 2.4
6: redis 2.6
7: redis 2.4
8: redis 2.2
What kind?> 5 <----- Select & Add RabbitMQ 2.4v service (for pub-sub)

Name?> rabbit-e1223 <-- This is just random name for RabbitMQ service

Creating service rabbit-e1223... OK
Binding rabbit-e1223 to rabbitpubsub... OK

Create another service?> y

1: blob 0.51
2: mongodb 2.0
3: mysql 5.1
4: postgresql 9.0
5: rabbitmq 2.4
6: redis 2.6
7: redis 2.4
8: redis 2.2
What kind?> 6 <----- Select & Add Redis 2.6v service (for session store)

Name?> redis-e9771 <-- This is just random name for Redis service

Creating service redis-e9771... OK
Binding redis-e9771 to rabbitpubsub... OK

Bind other services to application?> n

Save configuration?> n

Uploading rabbitpubsub... OK
Starting rabbitpubsub... OK
Checking rabbitpubsub... OK

```
* Once the server is up,o open up multiple browsers and go do `<servername>.cloudfoundry.com`
* Start chatting.

#### Test 1 ####

* Refresh the browser.
* You should automatically be logged in.


#### Test 2 ####

* Open up JS debugger (On Chrome, do `cmd + alt +j` )
* Restart the server by doing `vmc restart <appname>`
* Once the server restarts, Socket.io should automatically reconnect
* You should be able to chat after the reconnection.



## General Notes ####
* If you don't have Node.js, download it from <a href='http://nodejs.org' target='_blank'>here</a>
* If you don't have a Cloud Foundry account, sign up for it <a href='https://my.cloudfoundry.com/signup' target='_blank'>here</a>  
* Check out Cloud Foundry getting started <a href='http://docs.cloudfoundry.com/getting-started.html' target='_blank'>here</a> & install `vmc` Ruby command line tool to push apps.

* To install ***latest alpha or beta*** `vmc` tool run: `sudo gem install vmc ---pre`


####Credits####
<p>
PS: Front end UI: <a href="https://github.com/steffenwt/nodejs-pub-sub-chat-demo">https://github.com/steffenwt/nodejs-pub-sub-chat-demo</a></p>
