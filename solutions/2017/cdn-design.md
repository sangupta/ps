# Design of a Content Delivery Network - CDN

These notes are supposed to help in designing a content delivery network, aka CDN,
from scratch. The solution needs to be scalable for millions of clients to use, with
billions of objects being used at the edges. It should also support features like
cache expiration, forced cache-expiration, custom time-to-live values (TTLs) and more.
It needs to support authenticated requests as well.

Before we being on the design of a CDN, let's see what is a CDN and how does a CDN
works to understand it better.

## What is a CDN

A Content-Delivery-Network is a proxy server, between the user's browser and the
actual server that is intended to speed-up serving resources by reducing the latency
between the browser and server. Thus, a CDN has servers across many countries (called
as edge locations) that serve browser clients from within their own geographical
fence. As the edge server is close to browser, the network latencies are reduced
drastically.

Sometimes CDN networks are also meant to take load-off the actual server, specially
for blob-storage kind of usages (images, thumbnails, binaries and all) - but this is
now known as blob-stores. CDNs may be deployed along with blob-stores to still reduce
the load and latencies between actual blob server and the end-user.

The source of truth in CDN world is called the origin server. This is the server from
where a CDN edge can always fetch the latest and greatest source of content, and is
always supposed to be available for a smaller number of requests. Remember, CDN is also
meant to take-off load of the origin servers.

A typical request flow with CDN edges can be visualized as:

```
                                        Actual Server
                                        Origin Server
                                              |
	   +----------------------------------+------------------------------------+
	   |                                  |                                    |

	Edge US 			 Edge Europe				Edge Asia

           |                                  |                                    |
     +-----+-+                        +---------------+                     +-------------+
     |       |                        |               |                     |             |

    CA    New-York                 England         Turkey                 India         Japan


```

There are two modes of working of a CDN server:

### 1. Push CDN

In a push CDN the origin server knows the CDN system and wants its files to be cached 
eagerly - like Apple OS updates, Android OS updates, Adobe creative cloud binary 
images - here we pay first and make sure that there is no bottleneck for the user. So 
origin itself pushes the content ahead-of-time to the CDN system, which is turn propogates 
it to multiple edge servers. This propogation usually takes a few minutes to replicate 
around the world (also depending on the file size).

The files in this case are cached eternally, until the origin explitly does not update 
them or delete them. The origin can also ask the CDN to refresh the file in this case, 
which is nothing else but pushing the file again to all edge servers of the world.

### 2. Pull CDN

In a pull CDN, the content is lazily fetched. In this case the origin only pushes the 
configuration to the CDN edge servers. Now when a request lands up on an edge server 
the very first time, it goes ahead and fetches the response from the origin server, 
caches it locally and serves to the browser. There is a TTL (via caching headers, 
`max-age` header, `expires` header, CDN config or otherwise) associated with each file 
(or CDN account depending on plan purchased). After this TTL has expired, the edge 
server is mandated to fetch the file again from the origin server.

However, an edge server may revoke the file before the TTL if it deems that no new 
request every came for the while to this edge server and it needs to free up space for 
other requests. In such a case the edge server will make another request to origin 
before the TTL actually expired.

This is cheaper due to the same edge server being reused for other requests and keeping 
disk space low.

## Design Notes

Now that we understand what a CDN is, let's talk about the different components the
system will have, and the nuances of the same.

### Scalability

### Cache expiration

### Forced cache-expiry

### TTL and custom TTL

### Authentication

## TL;DR

Per design CDN is nothing but a simple reverse-proxy engine like `Apache http`, `nginx`, 
`caddy` etc. The job is to check with the origin server and serve data to clients
downstream.
