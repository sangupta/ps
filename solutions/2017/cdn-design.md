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

![Push Based CDN](https://github.com/sangupta/ps/blob/master/solutions/2017/cdn-edge-diagram.svg)

```mermaidjs
graph TD;
OS[Actual Server, also called, Origin Server]
EU[Edge US]
EE[Edge Europe]
EA[Edge Asia]
CA[California]
NY[New York]
EN[England]
TR[Turkey]
IN[India]
JP[Japan]
OS-->EE
OS-->EA
OS-->EU
EU-->CA
EU-->NY
EE-->EN
EE-->TR
EA-->IN
EA-->JP
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

![Push Based CDN](https://github.com/sangupta/ps/blob/master/solutions/2017/push-based-cdn.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant CDN
participant Origin
participant Release System
Release System->>Origin: Push new artifacts
Origin->>CDN: Push new artifacts

Note left of Browser: Request 1
Browser->>CDN: Request artifact
CDN-->>Browser: Serve artifact

Note left of Browser: Request 2
Browser->>CDN: Request artifact
CDN-->>Browser: Serve artifact
```

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

![Pull Based CDN](https://github.com/sangupta/ps/blob/master/solutions/2017/pull-based-cdn.svg)

```mermaidjs
sequenceDiagram
participant Browser
participant CDN
participant Origin
participant Release System
Release System->>Origin: Push new artifacts
Note left of Browser: Request 1
Browser->>CDN: Request artifact
CDN->>Origin: Get artifact
Origin-->>CDN: Serve artifact
CDN-->>Browser: Serve artifact

Note left of Browser: Request 2
Browser->>CDN: Request same artifact again
CDN-->>Browser: Serve artifact from cache
```

## Design Notes

Now that we understand what a CDN is, let's talk about the different components the
system will have, and the nuances of the same.

In most scenarios, a CDN system servers only `GET` requests and `POST`, `PUT` and other 
VERB calls are handled by origin directly.

For the sake of this design we will assume that our Edge servers are deployed at `edge.
cdn.com` and that the origin server is available at `origin.my.com`. We will first
discuss the scenrio for a single origin server being served by a given edge server and
then extend it to multiple servers.

Let's assume the following value-object that will help us respond to requests on the
edge server. For simplicity sake, we will assume that we are using a `Servlet` based
system to code this CDN system.

We will first start with a `pull-CDN` and then extend it to the `push-CDN`.

```java
public class MyEdgeFacingServlet extends HttpServlet {

	/**
	 * This is the cache service that we use to check data in local cache at edge
	 * server. As CDN serves blobs the cache service is expected to be a scalable
	 * caching service, something on the lines, of a distributed cache.
	 */
	private CacheService cacheService;

	/**
	 * This is the service that we use to make the HTTP requests to origin server.
	 */
	private HttpService httpService;

	/**
	 * This service basically gets all the configuration related to the incoming
	 * query by identifying the origin host and other parameters.
	 */
	private CdnConfigService cdnConfigService;

	/**
	 *
	 * This method takes an incoming call and then responds to it by looking
	 * for something in the local cache. If present, it will return the same. If not
	 * present it will then query the origin server, get the response, and store in
	 * local cache, and serve to client.
	 *
	 */
	@Override
	public void doGET(HttpServletRequest request, HttpServletResponse response) {
		// find host for which we are proxying
		String incomingHost = RequestUtils.findIncomingHost(request);

		// there was nothing in cache - let's hit the origin server now
		CdnConfig config = this.cdnConfigService.getConfigForOrigin(incomingHost);

		// build a cache key that is unique to this request, including the URI path
		// and other request parameters
		final String cacheKey = RequestUtils.getCacheKey(request);

		// get the cached response
		CachedResponse cached = this.cacheService.getCachedResponseForKey(cacheKey);

		// was something in cache
		if(cached != null) {

			// copy this cached response to client
			ResponseUtils.copyResponse(response, cached);

			// we are done
			return;
		}

		// based on this config build the query for origin server
		HttpRequest originRequest = RequestUtils.proxyRequest(config, request);

		// fire this request
		HttpResponse originResponse = this.httpService.makeRequest(originRequest);

		// because CDN is meant for speed let's not delay the browser
		ResponseUtils.copyResponse(response, originResponse);

		// check if we have a valid response or not
		// depending on the configuration of origin settings or otherwise
		// say a 404 may be a valid response or not
		boolean canCacheResponse = CacheUtils.canCacheResponse(config, originResponse);

		if(!canCacheResponse) {
			// there is nothing to cache
			return;
		}

		// we need to cache this response
		this.cacheService.putInCache(cacheKey, originResponse);
	}

}
```

In the code above, we are trying to handle all `GET` calls that land up on the edge server
by checking in cache. If nothing is found, a similar `GET` call is made to origin server to
fetch the data from the origin server. It is first written to the client browser to have it
proceed ahead (as CDN is all about speed), and then the response is written to the distributed
cache.

### Scaling for multiple origins and clients

As in the code above, we use the following code line to find the origin server that we need
to hit to fetch the source of truth:

```java
String incomingHost = RequestUtils.findIncomingHost(request);
```

This is based on the host-name of the incoming request. CDN servers are transparent to the
end user and thus, always respond on the origin server's host name, say `cdn.my.com`.

We thus can code this method to use a mapping between different incoming host-names and the
associated customer/origin-host for them. In psuedo-code it could be as simple as:

```java
public class RequestUtils {

	private static final Map<String, String> map = new HashMap<>();

	/**
	 * Called when this instance is initialized
	 */
	public void init() {
		map.put("cdn.my.com", "origin.my.com");
		map.put("cdn.client1.com", "origin.client1.com");
		map.put("cdn.client2.com", "client2origin.com");
	}

	public String findIncomingHost(HttpServletRequest request) {
		return map.get(request.getServerName());
	}

}

```

This will ensure that we can fetch the configuration based on many different clients/customers.
The same code can be deployed on multiple edge-servers in a single data-center that is serving
as CDN. If the distributed cache works great, and we build guards towards concurrent requests
properly, then only one request will go to the origin server from the edge-server farm till the
response does not expire.

To scale we only need to scale the caching service (which is nothing but a local BLOB store).

### TTL and custom TTL

TTL, or time-to-live, is the time after which the entry in the edge-server is considered expired
and the edge-server is supposed to be making the request again to the origin server. In our code 
above, we use `CacheUtils.canCacheResponse` to decide if the response should be cached or not.

Let's consider a typical simple implementation:

```java
public class CacheUtils {

	public boolean canCacheResponse(HttpResponse originResponse) {
		// let's check for expires header
		long expires = ResponseUtils.getExpiresValueInMillis(originResponse);
		if(expires > 0) {
			return true;
		}

		// let's check for max-age header
		long maxAge = ResponseUtils.getMaxAgeInMillis(originResponse);
		if(maxAge > 0) {
			return true;
		}

		return false;
	}

}
```

This method just checks for incoming response headers. But in a real-world scenario,  TTL for a 
given request cache needs to be based on the following parameters:

* TTL as per the configuration of the CDN system globally
* TTL as per the configured client
* TTL as per the response headers of the request

Different priorities can be decided to the three to figure out whether a request should be cached
and if yes, for what time. This is the onus of the method `canCacheResponse` in the code above. 

Let's update the method to return the `TTL` value as `milliseconds` than returning a `boolean`:

```java
public class CacheUtils {

	public long canCacheResponse(CdnConfig config, HttpResponse originResponse) {
		// let's check for expires header
		long expires = ResponseUtils.getExpiresValueInMillis(originResponse);
		if(expires > 0) {
			return expires;
		}

		// let's check for max-age header
		long maxAge = ResponseUtils.getMaxAgeInMillis(originResponse);
		if(maxAge > 0) {
			return maxAge;
		}

		// check for the value provided by client (origin-owner) configuration
		long ownerTTL = config.getOwnerTTL();
		if(ownerTTL > 0) {
			return ownerTTL;
		}

		// nothing is available - return the global TTL

		return this.cdnSystem.getGlobalTTL();
	}

}
```

Now we can modify our code to make use of this TTL value:

```java
long responseTTL = CacheUtils.canCacheResponse(config, originResponse);

if(responseTTL > 0) {
	// there is nothing to cache
	return;
}

// we need to cache this response
this.cacheService.putInCache(cacheKey, originResponse, responseTTL);
```

### Request Filters

Typical CDN systems also provide a way to set request filters to indicate what kind of queries
need to be cached or not. Say, all requests ending in `*.png` should always be cached irrespective
of the response headers, where as all requests to `*.meme` should never be cached. If you look at
the code in `CacheUtils.canCacheResponse` it becomes pretty simple to implement using the `CdnConfig`
object.

Consider the following `CdnConfig` object:

```java
public class CdnConfig {

	public String originHost;

	public final List<String> alwaysCache = new ArrayList();

	public final List<String> neverCache = new ArrayList();

	public boolean notCacheResponse(HttpResponse response) {
		String path = response.getRequestPath();

		for(String rule : neverCache) {
			if(StringUtils.wildcardMatch(rule, path)) {
				return true;
			}
		}

		return false;
	}

	public boolean alwaysCacheResponse(HttpResponse response) {
		String path = response.getRequestPath();

		for(String rule : alwaysCache) {
			if(StringUtils.wildcardMatch(rule, path)) {
				return true;
			}
		}

		return false;
	}

}
```

Now, code for `CacheUtils.canCacheResponse` will change to:

```java
public class CacheUtils {

	public long canCacheResponse(CdnConfig config, HttpResponse originResponse) {
		// check if this should not be cached per policy
		if(config.notCacheResponse(originResponse)) {
			return;
		}

		// check if this request is to be always cached per policy
		if(!config.alwaysCacheResponse(originResponse)) {
			// let's check for expires header
			long expires = ResponseUtils.getExpiresValueInMillis(originResponse);
			if(expires > 0) {
				return expires;
			}

			// let's check for max-age header
			long maxAge = ResponseUtils.getMaxAgeInMillis(originResponse);
			if(maxAge > 0) {
				return maxAge;
			}
		}

		// check for the value provided by client (origin-owner) configuration
		long ownerTTL = config.getOwnerTTL();
		if(ownerTTL > 0) {
			return ownerTTL;
		}

		// nothing is available - return the global TTL

		return this.cdnSystem.getGlobalTTL();
	}

}
```

### Cache expiration

Cache-expiration is the facility that the CDN system may allow to its clients to delete the
cache before the actual TTL flushes the data. This is useful when either the cache is corrupt,
or a new version of the asset is being pushed, or we just need to clean for another reason.

Consider the following `servlet` code at the edge location:

```java
public class MyEdgeFlushServlet extends HttpServlet {

	private CacheService cacheService;

	private CdnConfigService cdnConfigService;

	/**
	 *
	 * This method takes an incoming API call, authenticates it, and if genuine, clears
	 * the resources as requested by the client.
	 */
	@Override
	public void doGET(HttpServletRequest request, HttpServletResponse response) {
		String clientID = request.getParameter("clientID");

		// find client information
		CdnConfig config = this.cdnConfig.getConfigForClient(clientID);
		if(config == null) {
			// no such client is present
			response.sendError(HttpStatusCode.BAD_REQUEST, "No such client");
			return;
		}

		// make sure this is an authorized call
		boolean authenticatedRequest = SecurityUtils.isGenuineRequest(config, request);
		if(!authenticatedRequest) {
			response.sendError(HttpStatusCode.UNAUTHORIZED, "Not authenticated");
			return;
		}

		// find request or pattern to flush
		String resource = request.getParameter("requestPathToFlush");

		// clear it off the distributed cache
		this.cacheService.clearResource(config.originHost, resource);

		response.setStatus(HttpStatusCode.OK, "Resource cleared");
	}
```

Note that this `servlet` is deployed at the edge locations. A client is not supposed to call
all edge locations. Thus, a single admin end-point can be deployed that `fans-out` this call via
either direct-call to all edge-servers, or via a work `queue`. A `queue` based system is mostly
deployed to make sure that edge servers are not bombarded with requests from admin end-points
itself, and also to invert the `push` to `pull` which is simpler to implement.

### Authentication

Authentication in CDN usually applies to protected resources, say the paid-ones. For example,
`Disney` wants its paid users view the given movie `Moana` but not to every one. To improve
speed for all its subscribers they deploy a CDN, but the request has to be still authenticated.

Authentication for such request on a CDN is usually implemented in 2 ways:

#### Online Validation

In this methodology, the origin servers embed a validation code as a query-parameter to the request.
For example:

```
GET https://cdn.my.com/resource/at/path?validation=ABCDEFGHIJKLMNOPQRSTUVWXYZ
```

The validation code is generated by the origin server, `ABCDEFGHIJKLMNOPQRSTUVWXYZ` in this case,
and is per-request basis. It needs to change for all users, at every time. To make it more secure
the same token once generated should not be used in future.

The CDN system knows which parameter to read this validation code from, and is also aware of the
origin authentication end-point, where it can validate the token response from. Say the endpoint
is `auth.my.com`. 

Thus, when CDN receives a request such as above, it will pass the parameter to my authentication
service, and see if it returns a `HTTP OK` response. If yes, it will serve the resource, else, shall
send the `HTTP FORBIDDEN` status to the client.

Thus, the code for our `MyEdgeFacingServle.doGET` changes to:

```java
	
    public void doGET(HttpServletRequest request, HttpServletResponse response) {
		// find host for which we are proxying
		String incomingHost = RequestUtils.findIncomingHost(request);


		// there was nothing in cache - let's hit the origin server now
		CdnConfig config = this.cdnConfigService.getConfigForOrigin(incomingHost);

		// check for authentication
		String valiationRequired = config.isValidationRequired(request);
		if(validationRequired) {
			String auth = request.getParameter("validation");

			if(AssertUtils.isEmpty(auth)) {
				response.sendError(HttpStatusCode.BAD_REQUEST, "Auth required");
				return;
			}

			AuthResponse authResponse = this.httpService.makeValidationRequest(config.getValidationEndPoint(), auth);

			// TODO: the below code is just a simple implementation
			// in essence, you need to check for all valid HTTP 2xx codes
			if(authResponse.getStatusCode() != 200) {
				response.sendError(HttpStatusCode.BAD_REQUEST, "Auth required");
				return;
			}
		}

		// continue with checking in cache and all
	}
```

#### Offline Validation

The above approach has a disadvantage that for every incoming request, the CDN system has to call
an authentication end-point. This makes the authentication end-point a bottleneck. If it fails, no
resource will be served by the CDN system. Also, this adds additional latency before the CDN system
can even serve the request.

To overcome this limitation, an offline-validation mechanism can be introduced in the above
authentication workflow. The origin host will generate a valid signature for the request using
known `client-id` and `client-secret` which is known between the origin server and the CDN server
only. 

This is exactly the same workflow as maintaining stateless user sessions, [explained here]
(solutions/2016/stateless-user-session.md). The generated signature can thus be validated by the CDN
system just knowing the `client-secret` used to generate the key and can proceed ahead. For
additional security the concept of `nonce` can be applied here as well.

Another methodology being adopted in the wild is to encrypt the tokens using the public key of the
CDN system, which can only be decrypted by the private key that the CDN server has.

## Summary, TL;DR

Per design CDN is nothing but a simple reverse-proxy engine like `Apache http`, `nginx`, 
`caddy` etc. The job is to check with the origin server and serve data to clients
downstream.

It is a simple web application that is serving its own clients, just that its disguised
in the name of another web application, with two main focus areas: speed and security.
