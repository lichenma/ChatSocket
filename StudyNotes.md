# Building a Simple Chat Service with Spring Boot and WebSockets

This tutorial (credits to Singh at callicoder) will cover creating a simple group chat application 
using Websocket API and Spring Boot. 


**WebSocket** is a communication protocol that makes it possible to establish a two-way communication
channel between a server and a client. It works by first establishing a regular HTTP connection 
with the server and then upgrading it into a bidirectional websocket connect by sending an `Upgrade`
header. 


Websocket is supported in most modern web browsers and for those that don't support it we can fallback
to other techniques like comet and long-polling. 



## Creating the Application 

Similar to the TicTacToe game and the Spring REST API, we are going to be using Spring Initializr to
generate the project. This project is going to be a maven project and we are adding `Websocket` and 
`Rabbit MD` as the dependencies. Intellij Idea is my choice for IDE and now we can begin programming. 



## WebSocket Configuration 

The first step is to configure the websocket endpoint and message broker. Let's create a new package 
`config` inside `com.example.websocketchat`, and then create a new class called `WebSocketConfig`
inside it. It will have the following contents: 

```java 
package com.example.websocketchat.config; 

import org.springframework.context.annotation.Configuration; 
import org.springframework.messaging.simp.config.MessageBrokerRegistry; 
import org.springframework.web.socket.config.annotation.*; 


@Configuration 
@EnableWebSocketMessageBroker 
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer { 

	@Override
	public void registerStompEndpoints(StompEndpointRegistry registry) {
		
		registry.addEndpoint("/ws").withSockJS();
	}

	@Override
	public void configureMessageBroker(MessageBrokerRegistry registry) {
		
		registry.setApplicationDestinationPrefixes("/app");
		registry.enableSimpleBroker("/topic");
	}
}
```

The `@EnableWebSocketMessageBroker` is used to enable our WebSocket server. We implement 
`WebSocketMessageBrokerConfigurer` interface and provide implementation for some of its methods to 
configure the websocket connection. 

<br> 

In the first method, we register a websocket endpoing that the clients will use to connect to our 
websocket server. 

Notice the use of `withSockJS()` with the endpoint configuration. **SockJS** is used to enable the 
fallback options for browsers that don't support websocket. 


You might have noticed the word **STOMP** in the method name. These methods come from Spring 
framework's STOMP implementation. STOMP stands for **Simple Text Oriented Messaging Protocol** and is 
a messaging protocol that defines the format and rules for data exchange. 

<br>

> Why do we need STOMP? 

<br>

WebSocket is just a communication protocol. It doesn't define things like how to send a message only to
users who are subscribed to a particular topic, or how to send a message to a particular user. We need
STOMP for these functionalities. 


<br> 

In the second method, we are configuring a message broker that will be used to route messages from 
one client to another. 


```java
registry.setApplicationDestinationPrefixes("/app");
```

The first line defines that the messages whose destination starts with "/app" should be routed to 
message-handling methods (we will define these methods in the future). 


```java
registry.enableSimpleBroker("/topic");
```

And the second line defines that the messages whose destination starts with "/topic" should be 
reouted to the message broker. The message broker broadcasts messages to all the connected clients
who are subscribed to a particular topic. 


This enables a simple in-memory message broker, but we can also use other full-featured message brokers
like `RabbitMQ` or `ActiveMQ`



## Creating the ChatMessage Model 


`ChatMessage` model is the message payload that will be exchanged between the clients and the server. 
Create a new package `model` inside `com.example.websocketchat` and create the `ChatMessage` class with
the following contents: 


```java
package com.example.websocketchat.model;

public class ChatMessage {
	
	private MessageType type;
	private String content; 
	private String sender; 

	public enum MessageType {
		
		CHAT, 
		JOIN,
		LEAVE
	}

	public MessageType getType() {
		
		return type;
	}

	public void setType(MessageType type) {
		
		this.type = type;
	}

	public String getContent() {
		
		return content;
	}

	public void setContent(String content) {
		
		this.content = content; 
	}

	public String getSender() {
		
		return sender;
	}

	public void setSender(String sender) {
		
		this.sender = sender;
	}
}
```



## Creating the Controller for Sending and Receiving Messages 

We will define the message handling methods in our controller. These methods will be responsible for
receiving messages from one client and then broacasting it to others. 

Create a new package called `controller` under `com.example.websocketchat` and then create the 
`ChatController` class with the following content: 



```java 
package com.example.websocketchat.controller; 

@Controller 
public class ChatController {

	@MessageMapping("/chat.sendMessage")
	@SendTo("/topic/public")
	public ChatMessage sendMessage(@Payload ChatMessage chatMessage) {
		return chatMessage;
	}

	@MessageMapping("/chat.addUser")
	@SendTo("/topic/public")
	public ChatMessage addUser(@Payload ChatMessage chatMessage, 
					SimpMessageHeaderAccessor headerAccessor) {
		// Add username in web socket session
		headerAccessor.getSessionAttributes().put("username", chatMessage.getSender());
		return chatMessage;
	}
}
```


Previously from the websocket configuration, all the messages sent from clients with a destination 
starting with `/app` will be routed to these message handling methods annotated with `@MessageMapping`.



 **For Example** :

 * A message with destination `/app/chat.sendMessage` with be routed to the `sendMessage()` method 
 * A message with destintation `/app/chat.addUser` will be routed to the `addUser()` method. 


## Adding WebSocket Event Listeners 

We will use event listeners to listen for socket connect and disconnect events so that we can log these
events and also broadcast them wehn a user joins or leaves the chat room 



```java 
package com.example.websocketchat.cotnroller; 

@Component 
public class WebSocketEventListener { 
	
	private static final Logger logger = LoggerFactor.getLogger(WebSocketEventListener.class)

	@Autowired 
	private SimpMessageSendOperations messagingTemplate; 

	@EventListener 
	public void handleWebSocketConnectListener(SessionConnectedEvent event) {
		
		logger.info("Received a new web socket connected");
	}

	@Event Listener 
	public void handleWebSocketDisconnectListener(SessionDisconnectEvent event) {
		
		StompHeaderAccessor = StompHeaderAccessor.wrap(event.getMessage()); 

		String username = (String) headerAccessor.getSessionAttributes().get("username");
		if (username != null) {
			
			logger.info("User Disconnected: " + username);

			ChatMessage chatMessage = new ChatMessage(); 
			chatMessage.setType(ChatMessage.MessageType.LEAVE);
			chatMessage.setSender(username);

			messagingTemplate.convertAndSend("/topic/public", chatMessage);
		}
	}
} 
```

We are already broadcasting the user join event in the `addUser()` method defined inside 
`ChatController`. So we do not need to do anything in the SessionConnected event. 

In the SessionDisconnect event, we have written code to extract the user's name from the websocket
session and broadcast a user leave event to all the connected clients. 



## Creating the Front-End


We are going to be adding the following folders and files inside `src/main/resources` directory: 

* `static/css/main.css`
* `static/js/main.js`
* `static/index.html`

The `src/main/resources/static` folder is the default location for static files in Spring Boot. 


## Creating the HTML - index.html

The HTML file contains the user interface for displaying the chat messages. It includes `sockjs` and 
`stomp` javascript libraries. 

SockJS is a WebSocket client that tries to use native WebSockets and provides intelligent fallback 
options for older browsers that don't support WebSocket. STOMP JS is the stomp client for javascript.

The following is the complete code for `index.html` : 

```html
<!DOCTYPE html> 
<html>
	<head> 
		<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0"
		<title>Spring Boot WebSocket Chat Application</title> 
		<link rel="stylesheet" href="/css/main.css"/>
	</head>
	<body> 
		<noscript> 
			<h2> Uh Oh, Looks like your browser doesn't support Javascript</h2>
		</noscript>

		<div id="username-page">
			<div class="username-page-container">
				<h1 class="title">Enter your Username</h1> 
				<form id="usernameForm" name="usernameForm">
					<div class="form-group">
						<input type="text" id="name" placeholder="Username" autocomplete="off" class="form-control"/>
					</div>
					<div class="form-group">
						<button type="submit" class="accent username-submit">Start Chatting</button>
					</div>
				</form>
			</div>
		</div>

		<div id="chat-page" class="hidden">
			<div class="chat-container">
				<div class="chat-header">
					<h2>Spring WebSocket Chat</h2>
				</div>
				<div class="connecting">
					Connecting...
				</div>
				<ul id="messageArea">

				</ul>
				<form id="messageForm" name="messageForm">
					<div class="form-group">
						<div class="input-group clearfix">
							<input type="text" id="message" placeholder="Type a Message..." autocomplete="off" class="form-control"/>
							<button type="submit" class="primary">Send</button>
						</div>
					</div>
				</form>
			</div>
		</div>

		<script src="https://cdnjs.cloudflare.com/ajax/libs/sockjs-client/1.1.4/sockjs.min.js"></script>
		<script src="https://cdnjs.cloudflare.com/ajax/libs/stomp.js/2.3.3/stomp.min.js"></script>
		<script src="/js/main.js"></script>
	</body>
</html>

```

## JavaScript - main.js

Now we can add the javascript required for connecting to the websocket endpoint and sending and 
receiving messages. First, add the following code to the `main.js` file, and then we will cover some
of the important methods in the file: 


```javascript 

'use strict';

var usernamePage = document.querySelector('#username-page')
var chatPage = document.querySelector('#chat-page');
var usernameForm = document.querySelector('#usernameForm');
var messageForm = document.querySelector('#messageForm');
var messageInput = document.querySelector('#message');
var messageArea = document.querySelector('#messageArea');
var connectingElement = document.querySelector('.connecting');

var stompClient = null; 
var username = null; 

var colors = [
	'#2196F3', '#32c787', '#00BCD4', '#ff5652', 
	'#ffc107', '#ff85af', '#FF9800', '#39bbb0'
];


function connect(event) {
	
	username = document.querySelector('#name').value.trim();

	if(username) {
		
		usernamePage.classList.add('hidden');
		chatPage.classList.remove('hidden');

		var socket = new SockJS('/ws');
		stompClient = Stomp.over(socket);

		stompClient.connect({}, onConnected, onError);
	}

	event.preventDefault();
}


function onConnected() {
	
	// subscribe to the public topic 
	stompClient.subscribe('/topic/public', onMessageReceived);

	// send username to server 
	stompClient.send("/app/chat.addUser",
		{},
		JSON.stringify({sender: username, type: 'JOIN'})
	)
	
	connectingElement.classList.add('hidden');
}


function onError(error) {
	
	connectingElement.textCOntent = 'Could not connect to WebSocket Server - Please refresh this page and Try Again';
	connectingElement.style.color = 'red';
}


function sendMessage(event) {
	
	var messageContent = messageInput.value.trim();
	if (messageContent && stompClient) {
		
		var chatMessage = {
			sender: username, 
			content: messageInput.value,
			type: 'CHAT'
		};
		
		stompClient.send("/app/chat.sendMessage", {}, JSON.stringify(chatMessage));
		messageInput.value = '';
	}

	event.preventDefault();
}


function onMessageReceived(payload) {
	
	var message = JSON.parse(payload.body);

	var messageElement = document.createElement('li');

	if (message.type === 'JOIN') {
		
		messageElement.classList.add('event-message');
		message.content = message.sender + ' joined!';
	} else if (message.type === 'LEAVE') {
		messageElement.classList.add('event-message');
		message.content=message.sender + ' left!';
	} else {
		messElement.classList.add('chat-message');

		var avatarElement = document.createElement('i');
		var avatarText = document.createTextNode(message.sender[0]);
		avatarElement.appendChild(avatarText);
		avatarElement.style['background-color'] = getAvatarColor(message.sender);

		messageElement.appendChild(avatarElement);

		var usernameElement = document.createElement('span');
		var usernameText = document.createTextNode(message.sender);
		usernameElement.appendChild(usernameText);
		messageElement.appendChild(usernameElement);
	}

	 var textElement = document.createElement('p');
	 var messageText = document.createTextNode(message.content);
	 textElement.appendChild(messageText);

	 messageElement.appendChild(textElement);

	 messageArea.appendChild(messageElement);
	 messageArea.scrollTop = messageArea.scrollHeight;
}


function getAvatarColor(messageSender) {
	
	var hash = 0;
	for (var i=0; i<messageSender.length; i++) {
		hash = 31 * hash + messageSender.charCodeAt(i);
	}
	var index=Math.abs(hash % colors.length);
	return colors[index];
}



usernameForm.addEventListener('submit', connect, true)
messageForm.addEventListener('submit', sendMessage, true)
```




### Import Things to Note


The `connect()` function uses `SockJS` and `stomp` client to connect to the `/ws` endpoint that we 
configured in Spring Boot. 


Upon successful connection, the client subscribes to `/topic/public` destination and communicates the 
user's name to the server by sending a message to the `/add/chat.addUser` destination. 


The `stompClient.subscribe()` function takes a callback method which is called whenever a message 
arrives on the subscribed topic. 


The rest of the code is used to display and format the messages on the screen . 






## CSS - main.css


Finally, here is the styling used in the `main.css` file: 

```css
* {
	-webkit-box-sizing: border-box; 
	-moz-box-sizing: border-box; 
	box-sizing: border-box; 
}


html, body {
	height: 100%;
	overflow: hidden; 
}

body {
	margin: 0;
	padding: 0;
	font-weight: 400; 
	font-family: "Helvetica Neue", Helvetica, Arial, sans-serif; 
	font-size: 1rem;
	line-height: 1.58; 
	color: #333; 
	background-color: #f4f4f4; 
	height: 100%;
}

body:before {
	height: 50%;
	width: 100%;
	position: absolute; 
	top: 0;
	left: 0;
	background: #128ff2;
	content: "";
	z-index: 0;
}


.clearfix:after {
	display: block; 
	content: "";
	clear: both;
}


.hidden {
	display: none;
}


.form-control {
	width: 100%;
	min-height: 38px; 
	font-size: 15px;
	border: 1px solid #c8c8c8;
}


.form-group {
	margin-bottom: 15px; 
}


input {
	padding-left: 10px; 
	outline: none; 
}


h1, h2, h3, h4, h5, h6 {
	margin-top: 20px; 
	margin-bottom: 20px;
}


h1 {
	font-size: 1.7em;
}


a {
	color: #128ff2;
}


button {
	box-shadow: none; 
	border: 1px solid transparent; 
	font-size: 14px; 
	outline: none; 
	line-height: 100%;
	white-space: nowrap; 
	vertical-align: middle; 
	padding: 0.6rem 1rem; 
	border-radius: 2px;
	transition: all 0.2s ease-in-out; 
	cursor: pointer;
	min-height: 38px; 
}



button.default {
	background-color: #e8e8e8;
	color: #333;
	box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.12);
}


button.primary {
	background-color: #128ff2; 
	box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.12); 
	color: #fff;
}


button.accent {
	background-color: #ff4743;
	box-shadow: 0 2px 2px 2 rgba(0, 0, 0, 0.12);
	color: #fff;
}


#username-page {
	text-align: center; 
}


.username-page-container {
	background: #fff;
	box-shadow: 0 1px 11px rgba(0, 0, 0, 0.27);
	border-radius: 2px;
	width: 100%; 
	max-width: 500px; 
	display: inline-block; 
	margin-top: 42px; 
	vertical-align: middle; 
	position: relative; 
	padding: 35px 55px 35px;
	min-height: 250px; 
	position: absolute; 
	top: 50%; 
	left: 0; 
	right: 0; 
	margin: 0 auto; 
	margin-top: -160px; 
}


.username-page-container .username-submit {
	margin-top: 10px; 
}


#chat-page {
	position: relative; 
	height: 100%;
}


.chat-container {
	max-width: 700px; 
	margin-left: auto; 
	margin-right: auto; 
	background-color: #fff;
	box-shadow: 0 1px 11px rgba(0, 0, 0, 0.27);
	margin-top: 30px; 
	height: calc(100% - 60px);
	max-height: 600px;
	position: relative; 
}


#chat-page ul { 
	list-style-type: none; 
	background-color: #FFF:
	margin 0; 
	overflow: auto; 
	overflow-y: scroll; 
	padding: 0 20px 0px 20px; 
	height: calc(100% - 150px); 
}


#chat-page #messageForm {
	padding: 20px;
}



#chat-page ul li {
	line-height: 1.5rem; 
	padding: 10px 20px; 
	margin: 0; 
	border-bottom: 1px solid #f4f4f4;
}


#chat-page ul li p {
	margin: 0;
}


#chat-page .event-message {
	width: 100%;
	text-align: center;
	clear: both; 
}


#chat-page .event-message p {
	color: #777;
	font-size: 14px; 
	word-wrap: break-word;
}


#chat-page .chat-message {
	padding-left: 68px; 
	position: relative; 
}


#chat-page .chat-message i {
	position: absolute; 
	width: 42px; 
	height: 42px; 
	overflow: hidden;
	left: 10px; 
	display: inline-block;
	vertical-align: middle; 
	font-size: 18px; 
	line-height: 42px; 
	color: #fff;
	text-align: center; 
	border-radius: 50%;
	font-style: normal; 
	text-transform: uppercase; 
}


#chat-page .chat-message span {
	color: #333;
	font-weight: 600;
}


#chat-page .chat-message p {
	color: #43464b;
}


#messageForm .input-group input {
	float: left; 
	width: calc(100% - 85px);
}


#messageForm .input-group button {
	float: left; 
	width: 80px; 
	height: 38px; 
	margin-left: 5px;
}


.chat-header {
	text-align: center;
	padding: 15px; 
	border-bottom: 1px solid #ececec;
}


.chat-header h2 {
	margin: 0;
	font-weight: 500;
}


.connecting {
	padding-top: 5px; 
	text-align: center; 
	color: #777; 
	position: absolute; 
	top: 65px; 
	width: 100%;
}


@media screen and (max-width: 730px) {
	
	.chat-container {
		margin-left: 10px; 
		margin-right: 10px; 
		margin-top: 10px; 
	}
}


@media screen and (max-width: 480px) {
	
	.chat-container {
		height: calc(100% - 30px); 
	}

	.username-page-container {
		width: auto;
		margin-left: 15px; 
		margin-right: 15px; 
		padding: 25px; 
	}


	#chat-page ul {
		height: calc(100% - 120px);
	}


	#messageForm .input-group button {
		width: 65px;
	}


	#messageFOrm .input-group input {
		width: calc(100% - 70px);
	}

	
	.chat-header {
		padding: 10px; 
	}


	.connecting {
		top: 60px;
	}

	.chat-header h2 {
		font-size: 1.1em;
	}
}
```



## Running the Application 

We can now test and see the application in action by running the following command in the terminal


```
$ mvn spring-boot:run
```

The application starts on Spring Boot's default port 8080. You can browse the application at 
`http://localhost:8080`. 






## Using RabbitMQ Message Broker 

If you want to use a full featured message broker like RabbitMQ instead of the simple in-memory message
broker then just add the following dependencies in the `pom.xml` file 



```xml 
<!-- RabbitMQ Starter Dependency --> 
<dependency> 
	<groupId>org.springframework.boot</groupId> 
	<artifactId>spring-boot-starter-amqp</artifactId> 
</dependency> 

<!-- Following dependency is also required for full featured STOMP broker relay -->
<dependency> 
	<groupId>org.springframework.boot</groupId> 
	<artifactId>string-boot-starter-reactor-netty</artifactId> 
</dependency> 
```

Once we have added the above dependences, we can now enable RabbitMQ message broker in the 
`WebSocketConfig.java` file like this: 



```java 
public void configureMessageBroker(MessageBrokerRegistry registry) {
	regist.setApplicationDestinationPrefixes("/app");

	// Use this for enabling a Full featured broker like RabbitMQ
	registry.enableStompBrokerRelay("/topic")
		.setReplayHost("localhost")
		.setReplayPort(61613)
		.setClientLogin("guest")
		.setClientPasscode("guest"):
}
```


## Conclusion 


Thanks again to Rajeev at Callicoder for providing an awesome tutorial to follow and through this 
tutorial we have built a fully-fledged chat application from scratch using Spring Boot and WebSocket. 



