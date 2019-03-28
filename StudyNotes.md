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



