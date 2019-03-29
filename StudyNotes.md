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

 * A message with destination `/app/chat.sendMessage with be routed to the `sendMessage()` method 
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























