# Problem

Design a rate-limited web scraper.

# Problem Discussion

Though the problem statement is limited in its definition - there are many non
functional requirements that may be deduced:

* We are asked to build a web-scraper and not a web-crawler - thus there is no
question of crawling pages being referred to by the URL requested - that is
crawling depth is zero

* The URLs to be scraped are being provided to the system by an external
mechanism say over a message queue, or RPC calls

* The system needs to be scalable - millions and billions of URLs

* Scalability also points to the presence of multiple clients that will be using
this system and thus duplicates will be present

* To detect duplicates, URL canonicalization is required for incoming URLs

* A URL that has just been scraped need not be scraped again in a given time
frame say 1 minute, or 1 hour, or 1 day. We will assume the time interval to be
1 hour for our discussion here.

* Rate-limiting is required - we can safely assume that to be at the domain
level

* The data obtained by scraping can be assumed to be safely stored in a large
database - and challenges to scale the storage can be ignored

* As the entire process is asynchronous, we need a way to let the clients know
of the progress of their batch

# Solution

With all the above detailed requirements we just need to tie up the pieces. The
following modules are required in this system:

* Web server: This module is the external facing entity for all the clients. It
receives all incoming RPC calls and collects the packets of URLs that need to be
scraped.

* Scraping Worker: This module is the actual worker - given a URL, it normalizes
the URL, scrapes it from the web and sends it to the data store for storage.

* Notifier: This module is responsible for keeping a track of the progress of
incoming batch of URLs and update their progress back to clients.

We will need multiple multiple scraping workers aka `job workers` - to scrape
URLs to provide high throughput to the system. Each job worker will worker over
a message queue.

## Flow

For all practical purposes no client would like to make one RPC call per one URL
that needs to be scraped. All clients would mostly send the URLs in a batch so
as to optimize the RPC calls. Thus, our front-end acceptor will support RPC
calls to receive incoming batches of URLs. Each **batch** will consist of say a
max of `10000` URLs per request. This is only to reduce the payload and prevent
timeouts in responding to incoming requests.

Once the `web server` receives a `batch` of URLs, it will assign it a `batchID` -
a unique identifier that can be used to track status/progress of the scraping
activity. The `server` then goes over each URL, normalizes the URLs (see [URL
canonicalization](https://en.wikipedia.org/wiki/Canonicalization) for details),
and eliminates duplicates within the batch. We cannot be sure that duplicates will
not be sent by clients - as you cannot enforce during RPC calls and ignoring an
incoming request for this reason would not make sense.

## Chunking

Once we have the set of unique URLs, we group them by the domain name. All URLs
belonging to the same domain name are bunched together as a **JOB**. The group
is also made sure to be not more than `100` URLs - for more, multiple `jobs` are
created for the same domain name. This job is then assigned a `jobID` and linked
back to the `batchID`. Thus a single batch now consists of multiple jobs.

Limiting the number of URLs per job allows us to recover from failure faster and
easily. Also, it makes sure that resources are not wasted sitting idle (read
scalability later).

This information is then relayed by the `web server` to the `notifier` - this
ensures that the `notifier` is aware of the incoming progress notifications and
also the web-hook endpoints for clients where progress signals are to be relayed
back.

The `web server` then hashes the domain name for the job, and based on the hash
modulus **constant** (say 16 or 32 or 64) is assigned a unique bucket. Each
bucket corresponds to a message queue in the system. Either the entire `job` is
posted on the message queue, or it can be persisted to the data store and only
the `jobID` posted on the message queue.

The `constant` above helps to make sure that a given domain always goes to a
specific queue. This will help us implement rate-limiting easily and in a much
consistent way. Note that rate-limiting can also be achieved in a distributed
system with the use of Redis, but it will not guarantee sub-milli-second
consistencies due to latency of the network. One another benefit of dividing
jobs to specific queues is divide-and-conquer. A `batch` divided into `jobs` and
then to `queues` allows us to make our `scrapers` stateless.

## Scraping

`Scraping workers` or `scrapers` monitor a given queue (called their `master
queue`) for incoming jobs and start working on a given job as soon as one is
received. The `scraper` first needs to ensure that the URL has not been crawled
in `cache time` before proceeding to scrape.

### Ensuring caching

`Scraper` may need to connect to a secondary data-store to know if the URL has
been crawled before within the given time. This data-store can be modeled using
either [Lucene](http://lucene.apache.org/core/) (as it allows very fast
lookups over non-analyzed string keys) or a large-scale in-memory cache like
[Evictor (for Java)](https://github.com/stoyanr/Evictor), or by using a
[counting bloom-filter](https://en.wikipedia.org/wiki/Bloom_filter#Counting_filters)
where at cache-eviction we reduce the counts in the bloom filter.

### Rate limiting

The `scraper` then checks for rate-limiting using either an in-memory gate or via
a [Redis based rate-limiting](https://redis.io/commands/INCR) approach. As at a
given time we may have hundreds of workers, using a `Redis` based approach makes
more sense to be sure of entire system-wide rate limiting. This has a drawback
though that resources will be wasted during the idle time of the worker - when
it waits for a green-signal from rate-limiting gate.

The above drawback can however be mitigated allowing the `scraper` to process
more than one job at a time. During normal scenarios there will be thousands of
different domains that a given `message queue` will be serving. The probability
of two jobs along side in the same message queue will be low. Another technique
is discussed below in **Improvements** section.

### Scraping

The `scraper` then goes ahead and scrapes the URL off the internet. It stores
the results back into the data-store. It also updates the secondary data-store
to let it know that the URL has just arrived and also update the relevant cache
time (using response headers, see below for details).

If the URL that we hit from a scraper returns a error like `HTTP 500 - Internal
Sever Error`, `Gateway timeout`, or others from which the scraper can probably
recover, then it may push this URL into a retry queue. Once the URL is retried
a couple of times, say a max of 3 times, the URL can be stored back into the
data store with relevant error details. The cache time of such **errored** URLs
can be kept lower so as to allow their retry if a client requests it again.

## Callbacks

When a `scraper` completes a job successfully (with some URLs scraped and a few
non-scraped with errors), it marks the `job` complete in the database. It also
notifies the `notification` module that a given `job` is complete. The `notifier`
upon receiving the `jobID` checks the status of all `jobs` in a given `batch` -
and updates the client on completion when all are complete. An intermediate
status/progress update may also be relayed back to the clients depending on how
frequent the clients need to know of the progress.

# Improvements

Some of the improvements that can be made to this system are described below.

## Bucketization and rate-limiting

Increasing the number of `message queues` by increasing the `Bucketization
constant` is not an optimal solution as this increases the use of system
resources in monitoring message queues and others.

Another way to reduce wastage of resources by a `scraper` is to first find the
bucket for all the URLs in a `batch` which will be assigned the same `message
queue`. Now before generating `jobs` out of this group, the URLs should be
randomized sorted to ensure that URLs from multiple domains become part of one
job, and URLs from the same-domain are spaced apart.

## Cache time

Earlier we noted that we will choose a time-interval in which a given URL will
not be crawled. To make the system more real-time and aware of the dynamics of
the web today, the cache time can also be set in accordance with the caching
headers as served by the server. This also has a healing effect on the system
that URLs that can be cached longer no longer use network resources for being
poked again and again.

## Robots.txt

The scraping system discussed above misses one another quality gate: to obey
the [robots.txt file](http://www.robotstxt.org/). This file that is present at
the root of the server allows the server to dictate on what should be scanned
and what should be avoided. It would be prudent and in accordance with best
practices to use the `robots.txt` file and to avoid scraping URL paths that are
restricted by the server.

## Resource distribution amongst clients

The message queue used can be converted to a priority queue to enqueue incoming
jobs based on resource allocation. We assign each client (identified by a
`clientID`) a unique auto-incrementing counter starting at `zero`. Whenever a
`job` is created for a client we assign it a priority using this incrementing
counter. The message is then pushed to the message queue with the given
priority. This ensures fair play and usage of system resources by all clients,
and also preventing of abuse of system by a client that turns rouge.

For example, if we have 2 clients. The client C1 sends in 300 URLs of the same
domain `example.com`. The client C2 send in 200 URLs also of the same domain
`example.com`. Jobs created for client C1 are created as:

| JobID | ClientID | Priority | Queue |
|-------|----------|----------|-------|
| j1-c1 | c1       | 1        | 1     |
| j2-c1 | c1       | 2        | 1     |
| j3-c1 | c1       | 3        | 1     |
| j1-c2 | c2       | 1        | 1     |
| j2-c2 | c2       | 2        | 1     |

Thus, the order in which the messages are enqueued is:

```
j1-c1 -> j2-c1 -> j1-c2 -> j2-c2 -> j3-c1
```

This ensures that the scraping worker helps us in fair allocation of resources.

## Secondary queues

As each `scraping` worker works on a single message queue it is easier to
auto-scale based on load on each queue. Say, a worker is assigned `MQ4` to work
upon. Till it can read `jobs` from its own queue - it can keep working. One this
particular worker finds that there is no more job on `MQ4` to work upon - it can
pick up **one (and only one) job** from any other queue. This may be called the
secondary queue for this worker. Once the secondary job is processed, the worker
may return back to processing the jobs from its primary queue, in this case `MQ4`.

There are multiple ways to choose the secondary queue for an idle worker:

* Round robin starting from the next higher queue number

* Pre-configured sequence of secondary queue ID

* Runtime based decision by finding the queue that has the maximum number of jobs
queued up. This approach allows the system to achieve no work-load at the earliest.

## Auto-scaling

Auto-scaling can be implemented to launch more `scraping` workers based on the
number of jobs enqueued over a given message queue. The higher the load on a given
message queue, the more the number of `scraping` workers assigned to it.

This also has the advantage of reducing the number of `scrapers` when the queue
load reduces. Eventually, we may achieve zero scrapers when there is no load on
the system.

The `scrapers` can thus be launched as soon as there is incoming load on the
system. When using a service like [Amazon EC2](https://aws.amazon.com/ec2/) one
may utilize [spot-instances](https://aws.amazon.com/ec2/spot/) to lower down the
cost of the workers.

# Notes on data-store

One aspect of this problem that we have ignored in this discussion is the kind
of data-store that we should be using. As URLs are incoming and we are just
storing the response, the read to write ration would hover around `1:1`. The
load on the server is more of write-intensive in terms of request distribution.
Each URL record is separate and isolated, and thus need not be adjacent to
another record for retrieval.

## SQL data stores

A simple MySQL cluster would work without any problem if we can manage the
lookup in an efficient way. One way is to store the MySQL node ID in a separate
lookup database against the URL and to scale number of MySQL nodes as the
storage needs grow.

## NoSQL data stores

NoSQL data stores like `MongoDB` can be employed but it will suffer from the
same problem as any SQL store, around partitioning of data based on the URL
that forms the primary key of each row.

Stores like `Cassandra` which are meant for large write-loads should be
preferable, as they are append-only stores, with background compaction and can
be linearly scaled by adding more nodes to the cluster.

## BLOB stores

A blob store like `Amazon S3` in my understanding would serve the best in this
use-case. As the store is itself distributed and works on BLOBs it can be easily
employed by hundreds of workers to write data-to without making the data-store
a bottleneck. Also, the `notifier` in such a case can send back the URL of the
data back to the client for read access, thus reducing the dependency of client
on this system for retrieval of data. Security on `Amazon S3` can be maintained
using pre-signed URLs for each client.
