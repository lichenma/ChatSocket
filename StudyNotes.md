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

> Why do we need STOMP? 

WebSocket is just a communication protocol. It doesn't define things like how to send a message only to
users who are subscribed to a particular topic, or how to send a message to a particular user. We need
STOMP for these functionalities. 


<br> 

In the second method, we are configuring a message broker that will be used to route messages from 
one client to another. 


```
registry.setApplicationDestinationPrefixes("/app");
```

The first line defines that the messages whose destination starts with "/app" should be routed to 
message-handling methods (we will define these methods in the future). 


```
registry.enableSimpleBroker("/topic");
```

And the second line defines that the messages whose destination starts with "/topic" should be 
reouted to the message broker. The message broker broadcasts messages to all the connected clients
who are subscribed to a particular topic. 


This enables a simple in-memory message broker, but we can also use other full-featured message brokers
like `RabbitMQ` or `ActiveMQ`



## Creating the ChatMessage Model 



















