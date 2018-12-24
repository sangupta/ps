* Deployment strategies for Java based web applications

There are multiple ways in which a Java based web application can be
deployed. The factors that influence the choice are:

1. Is front-end code static or dynamic?
2. 

## Front-end code

There are 3 different ways the front-end of a Java application can
be generated:

1. Directly via Java code - using JSPs or otherwise
2. Using HTML static code being served directly by JVM 
3. Decoupled code like React, Angular etc

In the first-case, as the front code is being emitted directly
by the JVM, the only thing we could do is to place a reverse proxy
in front of the Java server. The proxy shall serve spoon feeding
purposes.

In the second case, we can have