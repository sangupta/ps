# Problem

Design a stateless user-session mechanism.

# Solution

Stateless user-sessions is a methodology where the cookie or [session-id](https://en.wikipedia.org/wiki/Session_ID)
generated to represent an authenticated user is not persisted on the server. This
allows the request to present the user details to any of the web server within the
cluster serving the request. This helps in removing the **stickyness** from the
load balancer, which otherwise leads to wastage of resources.

A session identifier is responsible for the following functionalities:

* Identifying to the server the user who is making the request
* Expiry of the token after certain period of in-access
* Preventing [session-fixation](https://www.owasp.org/index.php/Session_fixation)
or [session-hijacking](https://www.owasp.org/index.php/Session_hijacking_attack) attacks

## Stateful Sessions

Consider a sticky user-session, where the session ID is generated as `98e9fecc89d2e0fc9c7f4a0cb790a0bf`
and bound to a user `User123`. Now the server will need to store the following
metadata in a database or a memcached/redis instance:

| Primary Key                      |  User   | Created On | Last Accessed |
|----------------------------------|---------|------------|---------------|
| 98e9fecc89d2e0fc9c7f4a0cb790a0bf | User123 | 1480359766 | 1480361564    |

The server stores the generated session key in the cookie and sends back the response.
When the client makes the request again, the server reads the cookie, and checks
the database for its presence. Once the cookie value is found, the user details
are available, and so are the timestamps which can be used to ascertain if the
session is still valid or not.

Now as every request requires a DB hit to ascertain the validity of the session
identifier this approach leads to the following disadvantages:

* Causes extra load on the database server
* Slows down the response time of every incoming request
* The session identifier can be brute-force generated using randomness to hijack
an existing session

## Stateless Sessions

If the server is not to store the details of the user session, then the only way
is to store all these details in the session value (mostly, the cookie value) in
a safe way. Observe that the information needed to identify the user is usually
limited. Let's say we create a `JSON` packet of the same as:

```json
{
  "userID" : "User123",
  "name" : "Sandeep Gupta",
  "created" : 1480359766,
  "lastAccess" : 1480361564
}
```

Now storing the data-packet as raw-text is not a good idea. So let's convert the
above string to [Base64](https://en.wikipedia.org/wiki/Base64) encoding:

```text
ew0KICAidXNlcklEIiA6ICJVc2VyMTIzIiwNCiAgIm5hbWUiIDogIlNhbmRlZXAgR3VwdGEiLA0KICA
iY3JlYXRlZCIgOiAxNDgwMzU5NzY2LA0KICAibGFzdEFjY2VzcyIgOiAxNDgwMzYxNTY0DQp9
```

Now when the server receives the request it can reading the cookie or session-id
know which user is making the request and can serve the request without hitting
a database. This improves the overall performance of the system, and reduces the
load on the database.

## Challenges in stateless sessions

### Preventing brute-force attacks

But this token value is still susceptible to being brute-forced. To prevent such
misuse, we can keep a digital signing key on the server. For example, consider
that the server uses the following key to sign the token above:

```
Signing Key: 528da031caa54cb32f86a2e3944e958c
```

Now signing the above `base64` JSON token with the signing key using `SHA256-HMAC`
algorithm, provides us the signature as: `23b983bb2dadfe757d0514597d820e0f86426ca74bf5fa852463f05b2904045f`
Append this signature to the end of the token with a separator (say dot, `.`)
resulting in the following token:

```
ew0KICAidXNlcklEIiA6ICJVc2VyMTIzIiwNCiAgIm5hbWUiIDogIlNhbmRlZXAgR3VwdGEiLA0KICA
iY3JlYXRlZCIgOiAxNDgwMzU5NzY2LA0KICAibGFzdEFjY2VzcyIgOiAxNDgwMzYxNTY0DQp9.23b98
3bb2dadfe757d0514597d820e0f86426ca74bf5fa852463f05b2904045f
```

Thus, when the server receives the above token, it can verify the validity of the
token before using the same information. As checking HMAC signature is inexpensive
it does not slow down the request processing. A malicious user trying to fudge the
request will not be able to generate the same signature (due to missing signing
key).

Before you think that the above token can be brute forced, consider that someone
was able to generate a concatenated token that passes the signing key check. But
the same brute forced string also representing a valid JSON in the first part, and
containing valid user details is too high a coincidence.

### Prevent session hijacking

Apart from usual security measures like usage of SSL, generating a very-long session
key, the following are the few key things to prevent session hijacking:

* Store the IP of the incoming request when the JSON token is created first. For
every subsequent request, check that the IP of the next request is the same IP as
mentioned in the JSON token. However, do note that this may break users accessing
your service over the [TOR](https://www.torproject.org/) network. Also, this may
not prevent malicious users using the same egress IP or on the same network.

* An alternate mechanism to bind the request to the IP, is to bind the request to
a [browser fingerprint](https://en.wikipedia.org/wiki/Device_fingerprint). This
makes it difficult for a malicious user to have the exact same browser, machine and plugin
configuration as the host, to have the same browser fingerprint being generated.

* Using a [nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce) token with
every request and changing that on server side with every subsequent request. This
ensures that the malicious user has a very small window to align himself with the
server. This will invalidate the session of the actual user immediately, which would
force him/her to sign-in again. Thus, effectively invalidating the malicious user
session.
