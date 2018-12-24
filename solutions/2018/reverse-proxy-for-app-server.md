# Reverse Proxy in front-of Application Servers

Most of the production deployments in the application server 
world always use a reverse proxy (for example, nginx) in front 
of the application server (for example, Tomcat for Java, NodeJS,
ASP.NET server, PHP etc). These notes discuss the gains of
using a reverse-proxy and how they translate into a MUST HAVE
for our deployments.

## Connection Handling, Firewall

As the proxy server handles all client connections, our application
server is thus saved from being exposed to internet. The proxy server
lives in the DMZ, where as the application server lives inside and
only accepts connections from the proxy server. This provides a layer
of security to the app-server and code/data residing on the same.

Malicious clients (IP addresses analyzed from previous logs, or
bad search-bots, or crawlers) can be restricted by a simple 
configuration change in the proxy server. This prevents a restart of
the application server for such changes to take effect, and the values
can be modified at runtime.

Ensuring file upload limits is a good candidate to be done at the
proxy so that the app-server is 

![Connection Handling & Firewall](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/firewall.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Server
Browser->>Proxy: https://domain/path.html
Note over Proxy: Blacklisted IP?
Proxy-->>Browser: reject request
Note over Proxy: Bad search bot?
Proxy-->>Browser: reject request
Note over Proxy: Request length?
Proxy-->>Browser: reject request
Note over Proxy: All checks pass?
Proxy->>App Server: http://appserver/path.html
App Server-->>Proxy: Return page
Proxy-->>Browser: Return page
```

## Load Balancing

The proxy server can serve as a soft load balancer serving requests
from multiple app-servers to distribute the workload on to multiple
servers.

![Load Balanacer](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/load-balancer.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Node 1
Note left of Browser: Parallel Request 1
Browser->>Proxy: https://domain/path.html
Proxy->>App Node 1: http://appserver/path.html
App Node 1-->>Proxy: Return page
Proxy-->>Browser: Return page
Note left of Browser: Parallel Request 2
participant App Node 2
Browser->>Proxy: https://domain/path.html
Proxy->>App Node 2: http://appserver/path.html
App Node 2-->>Proxy: Return page
Proxy-->>Browser: Return page
```

It can ensure to re-route the client request internally to a different
node in case the first node goes down, thus, preventing a bad/dropped 
request and acting as a fail-over. This allows us for easy rolling-release 
of an application.

![Failover](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/failover.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Node 1
Note left of Browser: Fail over
Browser->>Proxy: https://domain/path.html
Proxy->>App Node 1: http://appserver/path.html
Note right of Proxy: App Node 1 is down
Proxy->>App Node 2: http://appserver/path.html
App Node 2-->>Proxy: Return page
Proxy-->>Browser: Return page
```

It can also be used to direct to specific app-nodes based on an incoming
cookie (stateful balancing), or based on the geography a user is coming
in from. Usually in such cases the app-server is stateful and is meant
to serve requests for a sub-set of users.

![Stateful Proxy](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/stateful-proxy.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Node (A..M)
Note left of Browser: User name ABC
Browser->>Proxy: https://domain/user/abc.html
Proxy->>App Node (A..M): http://appserver/home.html
App Node (A..M)-->>Proxy: Return page
Proxy-->>Browser: Return page
Note left of Browser: User name STU
participant App Node (S...Z)
Browser->>Proxy: https://domain/user/stu.html
Proxy->>App Node (S...Z): http://appserver/home.html
App Node (S...Z)-->>Proxy: Return page
Proxy-->>Browser: Return page
```

## Geo-identification

The proxy servers can figure out the geography of the incoming user
request and help set the correct language (for app localization) or
bringing in the correct country data (such as news site home page).

The proxy servers can cache these values based on connections and
help standardize these values as being sent to app-server. Fallbacks
for non-supposted geo, or locales can also be easily implemented by
mere simple configuration changes.

This again frees up app-server developers to this in app code saving
precious development minutes and runtime cycles.

![Geo Identification](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/geo-identification.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Node
Browser->>Proxy: https://domain/home.html
Note over Proxy: Add geo-information
Proxy->>App Node: /home.html?country=US&locale=en
App Node-->>Proxy: Return page
Proxy-->>Browser: Return page
```

## Spoon Feeding

Many of the clients that will connect to the application server
are slow. This may happen if the connection from client to server
is too slow due to bandwidth capacity at client end. Or, the
client machine is too over-loaded/band-width limited for a given
particular request. Or, a malicious client is attempting a
denial-of-service attack (DOS) on your server.

In the legitimate case, we need to feed response to the clients
at the rate they can read at. This involves the server to wait
before the next set of data can be sent. This is called as "spoon
feeding".

The same also happens for these slow clients sending a large request
(file upload, POST packets etc). The proxy server can buffer the
request at its end, and then send the complete request to the
app-server in one shot saving precious CPU cycles. The proxy can
also ensure basic checks: ensuring request length is within allowed
limits to prevent app-server overload. 

The reverse proxy server reduces resource usage caused by slow 
clients on the web servers by caching the content the web server 
sent and slowly "spoon feeding" it to slow clients. This especially 
benefits dynamically generated pages.

![Spoon Feeding](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/spoon-feeding.svg)

```mermaidjs
sequenceDiagram
participant Slow User
participant Fast User
participant Proxy
participant App Thread 1
Slow User->>Proxy: https://domain/home.html
Proxy->>App Thread 1: /home.html
App Thread 1-->>Proxy: Return page
Note over App Thread 1: Free for request
Proxy-->>Slow User: Return page (feed)
Proxy-->>Slow User: Return page (feed)
Proxy-->>Slow User: Return page (feed)
Fast User->>Proxy: https://domain/home.html
Proxy->>App Thread 1: /home.html
App Thread 1-->>Proxy: Return page
Note over App Thread 1: Free for request
Proxy-->>Slow User: Return page (feed)
Proxy-->>Fast User: Return page
Proxy-->>Slow User: Return page (feed)
Proxy-->>Slow User: Return page (feed)
```

If this is done using the actual app server, the application thread
generating the response will be blocked till the client is done
receiving the response. This wastes precious CPU time as the same
thread can be used to process another user's request. On the other
hand, the proxy server can free this precious thread. This becomes
particularly useful when the proxy and the app server are on 
different physical machines.

## SSL Termination

SSL handshakes are expensive in terms of CPU cycles. Proxy servers
take that load off the app-server and help with SSL termination of
the incoming request. The request can then flow between various app
server nodes on non-SSL giving a slight better performance.

Changing SSL private keys, protocols or ciphers does not require
restart of app-servers. This allows for faster response to security
issues without any downtime of the application.

![SSL Termination](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/ssl-termination.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Server
Browser->>Proxy: https://domain/path.html
Note over Proxy: Terminate SSL
Proxy->>App Server: http://appserver/path.html
App Server-->>Proxy: Return article
Proxy-->>Browser: Return article
```

## Security

## Caching

## Compression

## Logging and Auditing

## URL Rewriting

Proxy servers can help rewrite URLs to convert them from user-friendly
ones to the ones recognized by the app-server. For example, a URL like

```
https://myapp.com/@myuser/reverse-proxy-for-app-server-7fbe5d2a34.html
```

can be converted to:

```
https://myapp.com/content?user=myuser&article=7fbe5d2a34
```

They also benefit adding/removing prefix/suffix from the incoming URL.

![URL Rewriting](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/url-rewriting.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Server
participant Admin Server
Note left of Browser: Request Article
Browser->>Proxy: GET /@myuser/reverse-proxy-for-app-server-7fbe5d2a34.html
Proxy->>App Server: get /content?user=myuser&article=7fbe5d2a34
App Server-->>Proxy: Return article
Proxy-->>Browser: Return article
```

## Domain aggregation

A proxy server can help aggregate different services (applications)
on a to a single domain based on the path of the incoming request.
For example, a request to https://myapp.com/admin/home.html can be
sent to the administration website, whereas, the request 
https://myapp.com/home.html can be sent to the website home page.

The can also be used for cookie-less serving as explained later.

![Domain Aggregation](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/domain-aggregation.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Server
participant Admin Server
Note left of Browser: Request Home
Browser->>Proxy: /home.html
Proxy->>App Server: get /home.html
App Server-->>Proxy: Return home page
Proxy-->>Browser: Return home page
Note left of Browser: Request Admin
Browser->>Proxy: /admin/home.html
Proxy->>Admin Server: get /home.html
Admin Server-->>Proxy: Return home page
Proxy-->>Browser: Return home page
```

## Cookie-less serving

Serving static assets (images, css, javascript, html) from another
cookie-less domain helps in performance (usually on older browsers).
Older browsers had a limitation on number of concurrent connections
that they made to a single domain. Thus, hosting static resources
on a different domain allowed more connections to fetch website
resources thus leading to faster page load times.

Years back, the size of cookies associated with a page was large.
This also slowed down requests sent to server as the cookie payload
had to be sent everytime. Moving them to a different domain reduced
this cookie payload speeding up the requests.

Proxy server helps with setting up such static domains.

![Cookieless Static Serving](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/cookie-less.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Server
participant Static
Note left of Browser: Request Home
Browser->>Proxy: /home.html
Proxy->>App Server: get /home.html
App Server-->>Proxy: Return home page
Proxy-->>Browser: Return home page
Note left of Browser: Request CSS
Browser->>Proxy: /static/website.css
Proxy->>Static: get /website.css
Static-->>Proxy: Return CSS
Proxy-->>Browser: Return CSS
```

## Support for newer-protocols

Usually, support for newer protocols like SPDY, QUIC, HTTP 2.0 lands
up later in an application server than the proxy-servers. Thus, with
a proxy server front, we can support them in production much earlier
than directly serving from app-servers.

![New Protocols](https://github.com/sangupta/ps/blob/master/solutions/2018/resources/new-protocols.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant Proxy
participant App Server
Note right of Browser: using HTTP 2.0
Browser->>Proxy: https://domain/path.html
Note over Proxy: Terminate SSL
Note right of Proxy: using HTTP 1.1
Proxy->>App Server: http://appserver/path.html
App Server-->>Proxy: Return article
Proxy-->>Browser: Return article
```

# TL;DR

The proxy server can be deployed on a commodity server as opposed
to app-servers which require better CPU/higher RAM for operations.

# References

* [Revisiting the “Cookieless Domain” Recommendation](https://news.ycombinator.com/item?id=8765368)