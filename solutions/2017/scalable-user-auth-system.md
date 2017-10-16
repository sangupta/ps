# Problem

Design a scalable user authentication system that can support a billion
users and is fast enough.

# Solution

A user authentication system is one which stores username and password
combinations on users. When a user signs-up, the credentials of the new
user are stored in the system. When a user signs-in again, the provided
credentials are matched against the existing ones, to indicate if the user
returning back is a genuine one, or not.

For the purpose mentioned above, consider the following snippet of code:

```java
public class UserAuthenticationService {

	// Service that persists user related data to database and provides
	// convenience methods. The details of this service are out-of-scope
	// of this article
	private UserDatabaseService userService;

	public boolean addUser(String username, byte[] password) {
		if(AssertUtils.isEmpty(username)) {
			return false;
		}

		if(AssertUtils.isEmpty(password)) {
			return false;
		}

		// this is a pseudo-code to convey the intent
		String query = "INSERT INTO USER (name, password) VALUES (?, ?)";

		// the fireQuery method is assumed to return correct type
		// as parsing results of SQL is not the intent of this article
		return this.userService.fireQuery(query, new Object[] { username, password });
	}

	public boolean existsUser(String username) {
		if(AssertUtils.isEmpty(username)) {
			return false;
		}

		String query = "SELECT COUNT(*) FROM USER WHERE username = ?";
		int count = this.userService.fireQuery(query, new Object[] { username });
		if(count == 0) {
			return false;
		}

		return true;
	}

	public User login(String username, byte[] password) {
		if(AssertUtils.isEmpty(username)) {
			return null;
		}

		String query = "SELECT * FROM USER WHERE username = ? AND password = ?";
		Object user = this.userService.fireQuery(query, new Object[] { username, password });
		if(user == null) {
			return null;
		}

		return parseToUserObject(user);
	}
}
```

What makes this problem interesting is that it is one of the most critical
functionalities and involves a lot many non-functional requirements. Let's
take a look at all these NFR's and then look to solve them elegantly:

* It MUST NOT be super-easy to collect passwords from a running system by
taking heap dumps or looking at logs

* Passwords must meet basic security criteria - say 8 characters in length,
consist of 1 alphabet, 1 number and 1 symbol at the least

* Passwords must NOT be encrypted as the system does not need to know the
real credentials of the user - this also prevents system developers to turn
nasty

* Passwords must thus be hashed strongly. Salts should be added to the passwords
to make sure that they cannot be deciphered using dictionary attacks

* When changing the password, the users should not be able to re-use their
previous N passwords, say last 5 passwords

* The system should be damn-fast in responding to queries to check if a user
already exists in the system or not

* The username must be normalized to make sure that there are no user-conflicts
when the usernames seem identical, say prevent case-sensitive usernames

* The system should expire the security cookie/authentication token in 30 minutes
from last use time

* The system should allow users to have multiple logins from across devices and
make sure that the tokens do not conflict. Also, it should be possible to
expire the tokens on other devices for added security

We will see below as to what are the best available techniques to tackle each
of the requirements/constraints above.

## Passwords as byte-arrays

In the world of managed runtimes, for example, JVM, .NET etc the passwords should
always be stored using byte arrays, `byte[]`. This is essential as these
runtimes manage a `String pool` which are garbage collected at a lot slower rate
than passwords need to be removed from memory of running system. `byte[]` are
eligible for GC as soon as they run out of scope. Also, as password strings are
not same across users, the idle strings will just occupy memory-space in the
`String` pool than being re-used. This is additional CPU cycles for GC.

## Password standard

It is pretty easy to enforce a password standard on the user input. We just need
a function that matches the incoming password for the constraints. The function
will only be executed to check guideline enforcement when creating a new account,
or when a user wishes to change the password. The check need NOT be made when
authenticating as we need to compare against the given password, which would already
have been validated.

The following pseudo-code shows a typical password validation function:

```java
private boolean passwordMatchesCriteria(byte[] password, int len, int chars,int num, int symbols) {
	if(password == null) {
		return false;
	}

	if(password.length < len) {
		return false;
	}

	boolean hasChar = false;
	boolean hasNum = false;
	boolean hasSymbol = false;

	// the following loop is pseudo-code
	// can be made a lot performant
	for(int index = 0; index < password.length; index++) {
		byte b = password[index];

		hasChar = CharUtils.isAlphabet(b);
		hasNum = CharUtils.isNumber(b);
		hasSymbol = CharUtils.isSymbol(b);
	}

	return hasChar & hasNum & hasSymbol;
}
```

## Password Hashing

As mentioned before, there is no need for a system to be aware of the actual
password of a user. Even when the user needs to go through the **forgot password
workflow** they can do so using the on-file email address or by other means. Thus,
passwords need to be hashed and not encrypted.

## Password Salts

## Password change

## Multiple sessions
