# Problem

How do you prevent multiple form submission in web interface. This is also known
as preventing replay attacks.

# Solution

**TL;DR**: Use a `nonce` along with a `timestamp` token to prevent replays.

Multiple form submission is a common problem in web development. And the solution
to the problem is much easier than thought.

Let's say we have a form that sends an email to another user. The form parameters
would be:

* email address
* subject
* email contents

If we combine all these parameters into a single string and then check for uniqueness
then it provides us with a way to prevent duplicate form submissions. But, is this
correct? NO.

Consider that a user is sending an email to `johndoe@example.com` with subject `you
there` and contents `call me urgently`. Now suppose this email is sent at 10AM.
There may be a scenario where the user might want to send this email again at
11AM in case the response is not received. Is that a valid form submission? We cannot
ascertain in such a case if the form submission is indeed intended by the user or not.

## NONCE

Thus we introduce another parameter called `no-once, or nonce`. This parameter is
unique (randomly generated with sufficient guarantees of randomness) for every form
rendering and is added by the server during form generation. For ease of understanding
and for very practical purposes the `nonce` token can be taken to be a `UUID` token
itself. Thus, we now have four request parameters in our form:

* email address
* subject
* email contents
* nonce

When the server first receives the form submission it checks if it has seen this
`nonce` token before or not. If the server has not seen this token before, the form
submission is considered intended and the server proceeds normally. If the token
has been seen before, it is assumed that the form submission is a re-submit or a
replay of the previous action.

## Timestamp

The usage of `nonce` above introduces two new problems:

* Storing all previously seen `nonce` tokens which will occupy a huge space
* The chances of conflict in random byte generation however low, there is a chance
that any random algorithm may reproduce the same token again at a later time

Let's think about the second problem first. For any given random algorithm, the
timespan between generation of two similar tokens is usually very large given all
practical purposes. This should may one think that tokens can be expired at a later
stage after which it can be safely assumed that the token is safe to be reused.

This should help us solve the first problem too.

Let's add a `timestamp` to the request in say, [unix epoch](https://en.wikipedia.org/wiki/Unix_time)
format. Now we have five parameters in the request:

* email address
* subject
* email contents
* nonce
* timestamp

The server when receives the form submission checks whether the timestamp presented
is in a given delta of the current server time or not. This delta could be 5 minutes,
or 1 hour or 24 hours. The delta is usually decided based on the number of `nonce`
tokens expected to be presented during this time. The more the number of tokens
the more is the storage space.

Next, the server checks and stores the `nonce` token in its memory (may be RAM, or
a very fast ephemeral storage like [Redis](https://redis.io) or [Memcached](https://memcached.org/)).
The server devises this storage to make sure that the `nonce` tokens being stored
are individually expiring after the given delta time. Note here that the tokens
do not expire in blocks of delta time, but each individual token should expire in
time equal to `current server timestamp + delta time`.

This makes sure that the storage space for `nonce` tokens is kept low as well as
we can stop replay attacks for all practical purposes.

# Resources

* [Replay Attacks on Wikipedia](https://en.wikipedia.org/wiki/Replay_attack)
* [Nonce](https://en.wikipedia.org/wiki/Cryptographic_nonce)
* [Redis](https://redis.io)
* [Memcached](https://memcached.org/)
