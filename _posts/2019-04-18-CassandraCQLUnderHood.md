---
layout: post
title: Cassandra - CQL under the hood 
category: Cassandra
tags: [Cassandra]
---

+ Thrift vs. CQL   
+ CQL under the hood   
+ partition key vs. clustering column   
https://academy.datastax.com/planet-cassandra/making-the-change-from-thrift-to-cql  

Terms:  
+ Thrift: a legacy RPC protocol combined with a code generation tool, performed directly against the storage layer.  
+ Native Protocol: replacement for Thrift that only supports CQL  
+ Storage rows: keys and columns are stored on disk.  
+ CQL rows: an abstraction layer on top of the storage rows, not always match the underlying storage structure.  
So, how the CQL statements translated to storage layer? 

# Single Primary Key  
```
CREATE TABLE books (title text,author text,year int,PRIMARY KEY (title));
INSERT INTO books (title, author, year) VALUES ('Patriot Games', 'Tom Clancy', 1987);
INSERT INTO books (title, author, year) VALUES ('Without Remorse', 'Tom Clancy', 1993);
SELECT * FROM books;
title           | author     | year
-----------------+------------+------
Without Remorse | Tom Clancy | 1993
  Patriot Games | Tom Clancy | 1987
```
So far it looks like ANSI SQL, but something very different under the hood. 
cassandra-cli, which allow us to interact directly with storage rows, disappears in 3.0+ though. 
```
RowKey: Without Remorse
=> (name=, value=, timestamp=1393102991499000)
=> (name=author, value=Tom Clancy, timestamp=1393102991499000)
=> (name=year, value=1993, timestamp=1393102991499000)
RowKey: Patriot Games
=> (name=, value=, timestamp=1393102991499100)
=> (name=author, value=Tom Clancy, timestamp=1393102991499100)
=> (name=year, value=1987, timestamp=1393102991499100)
```
This is nearly a direct mapping to the CQL rows, except that we have an empty column at the beginning of each row (which is not a mistake; it is used internally by Cassandra).  
First, remember that the row key is distributed randomly using a hash algorithm, so the results are returned in no particular order. By contrast, columns are stored in sorted order by name, using the natural ordering of the type. In this case, “author” comes before “year” lexicographically, so it appears first in the list. These are critical points, as they are central to effective data modeling.  

# Compound keys 
```
CREATE TABLE authors (name text,year int,title text,isbn text,publisher text,PRIMARY KEY (name, year, title));
SELECT * from authors; 
name       | year | title           | isbn          | publisher
------------+------+-----------------+---------------+-----------
Tom Clancy | 1987 |   Patriot Games | 0-399-13241-4 |    Putnam
Tom Clancy | 1993 | Without Remorse | 0-399-13825-0 |    Putnam
```

**Partition Keys**  
When declaring a primary key, the first field in the list is always the partition key. This translates directly to the storage row key, which is randomly distributed in the cluster via the hash algorithm. In general, you must provide the partition key when issuing queries, so that Cassandra will know which nodes contain the requested data.  

**clustering columns**   
The remaining fields in the primary key declaration are called clustering columns, and these determine the ordering of the data on disk. They do not help determine the distribution of data in the cluster. But they play a key role in determining the kinds of queries you can run against your data  
```
RowKey: Tom Clancy
=> (name=1987:Patriot Games:ISBN, value=0-399-13241-4)
=> (name=1987:Patriot Games:publisher, value=Putnam)
=> (name=1993:Without Remorse:ISBN, value=0-399-13825-0)
=> (name=1993:Without Remorse:publisher, value=Putnam)
```
Our two CQL rows translated to a single storage row, because both of our inserts used the same partition key. But perhaps more interesting is the location of our year and title column values. They are stored as parts of the column name, rather than column values!  
Those who are experienced with Thrift-based data models will recognize this structure, which is referred to as composite columns. You can also observe that the rows are sorted first by year, then by title, which is the way we specified them in our primary key declaration. It is also possible to reverse the stored sort order by adding the WITH CLUSTERING ORDER BY clause, as follows:  
CREATE TABLE authors (name text,year int,title text,isbn text,publisher text,PRIMARY KEY (name, year, title)) WITH CLUSTERING ORDER BY (year DESC);

**Composite Partition Keys**  
In the previous examples we demonstrated the use of a single partition key with multiple clustering columns. But it’s also possible to create a multi-part (or “composite”) partition key. The most common reason for doing this is to improve data distribution characteristics. A prime example of this is the use of time buckets as keys when modeling time-series data.   
```
CREATE TABLE authors (name text,year int,title text,isbn text,publisher text,PRIMARY KEY ((name, year), title));
RowKey: Tom Clancy:1993
=> (name=Without Remorse:isbn, value=0-399-13241-4)
=> (name=Without Remorse:publisher, value=5075746e616d)
-------------------
RowKey: Tom Clancy:1987
=> (name=Patriot Games:isbn, value=0-399-13825-0)
=> (name=Patriot Games:publisher, value=5075746e616d
```

**Why This Matters**
You may be wondering why it matters how the data is stored internally. In fact it matters a great deal, for several important reasons:  
+ Your queries must respect the underlying storage. Cassandra doesn’t allow ad hoc queries of the sort that you can perform using SQL on a relational system. If you don’t understand how the data is stored, at best you will be constantly frustrated by the error messages you receive when you try to query your data, and at worst you will suffer poor performance.
+ You must choose your partition key carefully, because it must be known at query time, and must also distribute well across the cluster.
+ Because of its log-structured storage, Cassandra handles range queries very well. A range query simply means that you select a range of columns for a given key, in the order they are stored.
+ You have to carefully order your clustering columns, because the order affects the sort order of your data on disk and therefore determines the kinds of queries you can perform.
+ Proper data modeling in Cassandra requires you to structure your data in terms of your queries. This is backward compared to the approach taken in most relational models, where normalization is typically the objective. With Cassandra you must consider your queries first.
With these principles in mind, let’s examine what happens when you run different kinds of queries, so you can better understand how to structure your data.  
 

# Understanding Queries 
use this table:  CREATE TABLE authors (name text,year int,title text,isbn text,publisher text,PRIMARY KEY ((name, year), title));  
**Query by Key**
SELECT * FROM authors WHERE name='Tom Clancy';  
the query makes the request to the coordinator node, which owns a replica for our key. The coordinator then retrieves the row from another replica node to satisfy the quorum. Thus, we need a total of two nodes to satisfy the query:  
At the storage layer, this query first locates the partition key, then scans all the columns in order, as follows:  
```
RowKey: Tom Clancy
=> (name=1996:Executive Orders:publisher, value=Putnam)
=> (name=1996:Executive Orders:ISBN, value=0-399-13825-0)
=> (name=1994:Debt of Honor:publisher, value=Putnam)
=> (name=1994:Debt of Honor:ISBN, value=0-399-13826-1)
=> (name=1993:Without Remorse:publisher, value=Putnam)
=> (name=1993:Without Remorse:ISBN, value=0-399-13825-0
=> (name=1991:The Sum of All Fears:publisher, value=Putnam)
=> (name=1991:The Sum of All Fears:ISBN, value=0-399-13241-6)
...
=> (name=1987:Patriot Games:publisher, value=Putnam)
=> (name=1987:Patriot Games:ISBN, value=0-399-13241-4)
```
So even though this appears to be a simple query by key, at the storage layer it actually translates to a range query!

**Range Queries**
```
SELECT * FROM authors WHERE name = 'Tom Clancy' AND year >= 1993;  
RowKey: Tom Clancy
=> (name=1996:Executive Orders:publisher, value=Putnam)
=> (name=1996:Executive Orders:ISBN, value=0-399-13825-0)
=> (name=1994:Debt of Honor:publisher, value=Putnam)
=> (name=1994:Debt of Honor:ISBN, value=0-399-13826-1)
=> (name=1993:Without Remorse:publisher, value=Putnam)
=> (name=1993:Without Remorse:ISBN, value=0-399-13825-0)
=> (name=1991:The Sum of All Fears:publisher, value=Putnam)
=> (name=1991:The Sum of All Fears:ISBN, value=0-399-13241-6)
...
=> (name=1987:Patriot Games:publisher, value=Putnam)
=> (name=1987:Patriot Games:ISBN, value=0-399-13241-4)
```
Once it finds the year 1991, Cassandra knows there are no more records to scan. Therefore, this query is efficient because it must only read the required number of columns, plus one.   
To recap, there are three key points you should take from this discussion:  
+ Sequential queries are fast, because they take advantage of Cassandra’s natural sort order at the storage layer.
+ Queries by key and combination of key plus clustering column are sequential at the storage layer, which of course means they are optimal.
+ Write your data the way you intend to read it. Or, put another way, model your data in terms of your queries, not the other way around. Following this rule will help you avoid the most common data modeling pitfalls that plague those who are transitioning from a relational database.

# Collections 
+ Each item in a collection must not be more than 64 KB
+ A maximum of 64,000 items may be stored in a single collection
+ Querying a collection always returns the entire collection
+ Collections are best used for relatively small, bounded data sets

**sets**
It is a unique collection of items, meaning it does not allow for duplicates. In most languages sets have no specific ordering; Cassandra, however, stores them in their natural sort order, as you might expect.  
```
CREATE TABLE authors (name text,books set,  PRIMARY KEY (name));
INSERT INTO authors (name, books) VALUES ('Tom Clancy', {'Without Remorse', 'Patriot Games'});
UPDATE authors SET books = books + {'Red Storm Rising'} WHERE name = 'Tom Clancy';
UPDATE authors SET books = books - {'Red Storm Rising'} WHERE name = 'Tom Clancy';
```
At the storage layer, set values are stored as column names, with the values left blank. This guarantees uniqueness, as any attempt to rewrite the same item would simply result in overwriting the old column name. The storage representation of the books set would look like this:  
```
RowKey: Tom Clancy
=> (name=books:50617472696f742047616d6573, value=)
=> (name=books:576974686f75742052656d6f727365, value=)
```
Unfortunately Cassandra does not support a contains operation, so you must retrieve the entire set and perform this on the client. But sets can be quite useful as a container for unique items in a variety of data models.  
**lists**
lists look very similar to sets.And since lists are ordered, CQL supports prepend and append operations, which involve simply placing the item as either the first (prepend) or second (append) operands,   
```
CREATE TABLE authors (name text,books list, PRIMARY KEY (name));
INSERT INTO authors (name, books) VALUES ('Tom Clancy', ['Without Remorse', 'Patriot Games']);
UPDATE authors SET books = books – ['Red Storm Rising'] WHERE name = 'Tom Clancy';
UPDATE authors SET books = ['Red Storm Rising'] + books WHERE name = 'Tom Clancy';
UPDATE authors SET books = books – ['Red Storm Rising'] WHERE name = 'Tom Clancy';
RowKey: Tom Clancy
=> (name=books:d36de8b0305011e4a0dddbbeade718be, value=576974686f)
=> (name=books:d36de8b1305011e4a0dddbbeade718be, value=506174726)
```
Unlike the set, the list structure at the storage layer places the list item in the column value, and the column name instead contains a UUID for ordering purposes.   

**maps**
maps are a highly useful structure, as they can offer similar flexibility to the old dynamic column names many grew accustomed to in the Thrift days, as long as the total number of columns is kept to a reasonable number.  
```
CREATE TABLE authors (name text,books map(text, int), PRIMARY KEY (name));
INSERT INTO authors (name, books) VALUES ('Tom Clancy', {'Without Remorse':1993, 'Patriot Games':1987});
UPDATE authors SET books['Red Storm Rising'] = 1986 WHERE name = 'Tom Clancy';
DELETE books['Red Storm Rising'] FROM authors WHERE name = 'Tom Clancy';
RowKey: Tom Clancy
=> (name=books:50617472696f742047616d6573, value=000007c3)
=> (name=books:576974686f75742052656d6f727365, value=000007c9)
```
At the storage layer, maps look very similar to lists, except the ordering ID is replaced by the map key  

# Multi-Key Queries 


# Secondary Indices
# Deleting Immutable Data 
## Unexpected deletes
## The Problem with Tombstones
## Expiring columns 
## How NOT to use TTLs
## When Null Does Not Mean Empty






