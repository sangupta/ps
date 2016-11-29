# Problem

Assume two users on a Social network start with zero `0` connections. And add
connections at a rate of 100 per minute. Explain how you would design a solution
to find out the degrees of separation between two network profiles, given a known
function `getConnections(profile.id)` that returns data from over the network.
What differences would you make for realtime less consistent/optimal result vs
a slower more accurate result?

# Solution

I can think of many different ways to compute this. I will put them out below.

`Approach 2` seems the best considering the storage cost and traversal costs.
`Approach 3` can be near real-time if we can take care of the storage, and add
optimal number of workers for fan-out.


## Assumptions

* If a degree of separation cannot be found in a level of `DEPTH_TO_BREAK_AT`
(an integer constant) we will break and say the users are not connected

## Various Approaches

### Approach 1: Crude approach - Brute Force

As said, we already have a function `getConnections(profileID)` that returns a
list of all users that are directly connected to this user. Now, let's say
we want to compute the degrees of separation between two users `u1` and `u2`.
Assuming that the profile ID is a `String` so that we can accommodate `UUID`s
as well:

```java
/**
 * Get degrees of separation between two users
 */
public int degressOfSeparation(User u1, User u2) {
	// skipping null checks and all

	String p1 = u1.getProfileID();
	String p2 = u2.getProfileID();

	return doLevelSearch(1, p1, p2);
}

/**
 * Return the level we have nested deep in.
 */
private int doLevelSearch(int level, String p1, String p2) {
	do {
		List<String> connections = getConnections(p1);
		if(connections == null || connections.isEmpty()) {
			return -1; // no connection for user
		}

		if(connections.contains(p2)) {
			// found the user
			return level;
		}

		if(level >= DEPTH_TO_BREAK_AT) {
			// this can be a breaking condition to break in case of
			// no connection between two users
			// we need to break at some place
			return -1;
		}

		for(String connection : connections) {
			return doLevelSearch(level + 1, connection.getProfileID(), p2);
		}
	} while(true);
}
```

The following advantages are in this approach:

* Storage cost on disk, and in-memory are less

The following disadvantages are in this approach:

* Too slow - traverse the entire graph at each time
* When data is less, we can get into an entire graph traversal - thus need to
code `level >= 20` style condition to break at some point
* Need too many queries O(n ^ d) in the worst case, where d is the degree's of
separation
* The degrees of separation may not be accurate - this is because we are doing
a depth-first traversal

NOTE: As this is depth-first traversal, this will result in hell-slow output.
We MUST NOT use this approach.

### Approach 2: Brute Force - Two sided

To improve upon the approach 1, we can start with traversal from both the sides
of the tree, keeping in memory the list of all connections from both the sides.
This way we can save a lot of queries that we fire to the network.

The appraoch is a simple breadth-first traversal, but starting from both sides.

```java
/**
 * Get degrees of separation between two users
 */
public int degressOfSeparation(User u1, User u2) {
	// skipping null checks and all

	String p1 = u1.getProfileID();
	String p2 = u2.getProfileID();

	return doSearch(p1, p2);
}

/**
 * Do an iterative search for the degree of separation
 */
private int doSearch(String p1, String p2) {
	// now for the set
	List<Connection> c1 = new ArrayList<>();
	List<Connection> c2 = new ArrayList<>();

	// add users to their own owning lists
	c1.add(new Connection(p1, 0));
	c2.add(new Connection(p2, 0));

	do {
		// fetch first available user from list
		boolean hadNonScanned1 = fetchNextAvailable(c1);
		if(hadNonScanned1) {
			int hit = chechHit(c1, c2);
			if(hit > 0) {
				return hit;
			}
		}

		// check if there is atleast one connection that we can scan
		boolean hadNonScanned2 = fetchNextAvailable(c2);
		if(hadNonScanned2) {
			int hit = checkHit(c1, c2);
			if(hit > 0) {
				return hit;
			}
		}

		// check if there was at least one user in any of the two lists
		// that was scanned - if not, we have reached our crawling
		// limits
		if(!hadNonScanned1 && !hadNonScanned2) {
			// we have reached breaking limit
			return -1;
		}		
	} while(true);
}

/**
 * Function that scans an arraylist and finds out the first
 * connection that has not yet been scanned. Fetches the list
 * of connections that this profile has, adds them to the end
 * of the list and returns a `true` - this makes sure that we
 * continue looping.
 *
 * If we find no connection that is scannable, we return `false`
 */
private boolean fetchNextAvailable(List<Connection> list) {
	Iterator<Connection> iterator = list.iterator();
	while(iterator.hasNext()) {
		Connection c = iterator.next();

		if(c.isScanned()) {
			continue;
		}

		// we have a non-scanned version
		if(c.level >= DEPTH_TO_BREAK_AT) {
			continue;
		}

		// we have a candidate within list
		// mark this connection scanned
		c.markScanned();

		// go ahead and scan this connection
		List<String> ids = getConnections(c.profileID);
		if(ids != null && !ids.isEmpty()) {
			for(String id : ids) {
				list.add(new Connection(id, c.level + 1));
			}
		}

		return true;
	}

	return false;
}

private int checkHit(List<Connection> c1, List<Connection> c2) {
	for(Connection x : c1) {
		for(Connection y : c2) {
			if(c1.equals(c2)) {
				return c1.level + c2.level;
			}
		}
	}

	// not found
	return -1;
}

/**
 * A simple structure to store the level and the ID of the person
 * Also, stores a boolean to say if we have scanned the list or not.
 */
public static class Connection {

	// the level at which we found the user
	int level;

	// the profileID of the user
	String profileID;

	// keeps track if we have scanned this profile for its depth
	boolean scanned;

	Connection(String profileID, int level) {
		// do basic sanity of values
		this.profileID = profileID;
		this.level = level;
		this.scanned = false;
	}

	public int hashCode() {
		return this.profileID.hashCode();
	}

	public boolean equals(Object obj) {
		if(obj == null) {
			return false;
		}

		if(this == obj) {
			return true;
		}

		if(!(obj instanceof Connection)) {
			return false;
		}

		Connection other = (Connection) obj;

		return this.profileID.equals(other.profileID);
	}

	public void markScanned() {
		this.scanned = true;
	}

	public boolean isScanned() {
		return this.scanned;
	}
}
```

#### Advantages of this method over `Approach 1`

* Faster as we are running breadth-first traversal from both sides, which is
more likely to give us a match than running

* We break as soon as we have a match between any neighbour - thus the chain
can be discovered early.

#### Optimizations

* Needs a little extra memory on the machine calculating the degrees of separation
- this can be reduced by the use of sparse-bit-arrays if the `profileID`s can be
converted to `integer` values of a sequence.

* Use a bloom filter in method `checkHit` to be able to faster check if we have
a connection (or using bit-arrays)

* As we know more about users, than just their profile name. Thus when scanning
a connections list, for next level, we can given preference to connections that
are either from same city, same school/college, or same organization - they have
a higher chance of match than user's who are not from same city, school and org.

### Approach 3 - Fan out on Write

Another approach is where we can fanout on write - thus taking a near-real-time
approach. We will maintain a list of connections in depth of degree of a user in
multiple lists (distributed columnar storage like Cassandra, or a Redis style
stores). Whenever a connection is added, we go ahead and fanout adding him to
the list of all his connections at level 1, then for all users connected at
level 2, and so on... to a certain level.

Thus, say `A` has two connections, `X` and `Y`. `Y` in turn has one connection
called `Z` - now `A` adds a connection `M`. We add `M` to the list of `A` as
level 1, to the list of `X` and `Y` as level 2, and `Z` as level 3.

To be non-blocking and not hit performance, this is done in background using a
message broker for fan out.

Initially when the network is building, the connection addition rate will be high
and we may need more workers to flush out the queue. But with time the rate of
addition of new connections will dwindle, as people would have stabilized with
all their friend in the network. At this point the fan-out queue will be smaller.

This has advantage over `Approach 1` and `Approach 2` that finding a hit, just
involves O(1) operation. If we don't find a match we can safely return. If the
number of fan-out workers make sure that the fan-out queue never gets crowded,
we can hit near realtime performance.

`Columnar` storage like Cassandra or HBase will help as a single row needs to be
scanned. Using Redis may make it super-fast but the amount of memory we would
need would be too huge.

#### Optimizations

* Use of integerIDs as profile identifiers can help in improving the speed
further reducing the disk space cost. We can use a sparse bit-array where the bit
corresponding to the userID of connection is turned on. But storage cost when
degree of separation exceeds, say 4 or 5, will increase as the connections tend
to reach in millions then (from practical examples from LinkedIn and Facebook).
