# Building your own MongoDB

This article talks about creating your own MongoDB kind of database.

Before we delve into the technical details on how something like MongoDB could be 
implemented, let's first understand the basic capabilities/features that the database
provides to a user.

As defined in Mongo documentation at https://www.mongodb.com/what-is-mongodb:

```
MongoDB is a document database with the scalability and flexibility that you want 
with the querying and indexing that you need

* MongoDB stores data in flexible, JSON-like documents, meaning fields can vary 
from document to document and data structure can be changed over time

* The document model maps to the objects in your application code, making data 
easy to work with

* Ad hoc queries, indexing, and real time aggregation provide powerful ways to
access and analyze your data

* MongoDB is a distributed database at its core, so high availability, horizontal 
scaling, and geographic distribution are built in and easy to use
```

For the simplicity of this article we will not talk about connection handling or the
MongoDB REST interface.

## Document

The primary objective of MongoDB is to store documents in a collections. A group of
collections is known as a database.

As we understand a document is a free-form JSON style object containing multiple
attributes, composed child objects which in turn have their own attributes. Every 
MongoDB document has a unique ID, `_id`, associated with it. Let's define an interface 
to work with this ID attribute:

```java
public interface MongoDocument {

    public String getID();
    
    public void setID(String id);
}
```

The ID above is unique in a given collection and is guranteed to return the document it was
assigned to. Different documents in a different collections, in same or different databases,
can have the same ID. 

For example:

```json
{
  "_id" : "some-unique-id",
  "name" : "Sandeep",
  "title" : "Developer",
  "social" : {
      "github" : "sangupta",
      "twitter" : "sangupta"
  }
}
```

For simplicity, we will assume and assign a UUID to a document that already does not
have a key. In actual scenario, MongoDB generates time-stamped machine-specific sequential 
values. Read more at https://docs.mongodb.com/manual/reference/method/ObjectId/

## Collection

We can now employ a `Map` to store all these documents based on this ID as the primary
key. This gives us a `O(1)` complexity in terms of access over primary key. Now, given 
a document we can store it in the DB as:


```java
public final Map<String, MongoDocument> collection = new HashMap<>();


public void storeDocument(MongoDocument doc) {
  if(doc == null) {
    return;
  }
  String id = doc.getID();
  if(id == null) {
    id = UUID.randomUUID().toString();
    doc.setID(id);
  }

  collection.put(id, doc);
}
```

## Database

A database is nothing but a list of named collections. Each database has multiple collections
that are identified by name. Thus, we can again utilize a `Map` structure to store the same.

```java
Map<String, MongoCollection> database = new HashMap<>();

// replacing the MongoCollection with exact datastructure:
Map<String, Map<String, MongoDocument>> database = new HashMap<>();
```

## Querying

As all documents live as natural Java objects inside the memory, we can run a query over these
objects using reflection to make sure the property exists and we can also check child object
attributes using the `dot` attribute.

```java
// somewhere in our code
// find all objects that have the property `firstName` with value `Sandeep`
myDatabase.myCollection.find("firstName", "Sandeep");

// iterate over each document
public List<MongoDocument> find(String name, String value) {
  List<MongoDocument> documents = collection.values();
  for(MongoDocument doc : documents) {
    // check if doc has a property with given name
    // we can use Java reflection for the same
  }
}
```

I previously developed [Gather](https://github.com/sangupta/gather) a simple Java library to 
run SQL like queries over Java collections. The same can be employed to run these queries.

## Mongo GridFS

Per Mongo documentation at https://docs.mongodb.com/manual/core/gridfs/:

```
GridFS is a specification for storing and retrieving files that exceed the BSON-document size 
limit of 16 MB.
```

Documents that exceed in size can be stored as a `byte[]` (byte-array). They do not have child 
operations over them and thus, the files are either created, read, or deleted. They are not
modified in place.