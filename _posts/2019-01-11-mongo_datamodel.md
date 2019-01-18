---
layout: post
title: Mongo Data Modeling
category: MongoDB 
tags: [database, mongo]
---

The key challenge in data modeling is the needs of applications, the database engine performance, and data retrieval patterns.
Mongo supports felxible schema, the documents in a single collection do not need to have the same set of fields and the data type for a field can differ across documents.   

#### two types of data models: embedded data vs. references
* embedded data   
MongoDB allows related data to be embedded within a single document.
These denormalized data models allow applications to retrieve and manipulate related data in a single database operation.  
* reference   
references store the relationships between data by including links or references from one document to another.
application can resolve these references to access the related data.   

#### atomicity of write operations: single vs. multiple atomicity
* single document atomicity   
A write operation is atomic on the level of a single document, even if the operation modifies multiple embedded documents within a single document.   
* multi-docoument transactions   
Starting in version 4.0, for situations that require atomicity for updates to multiple documents or consistency between read to multiple documents, mongoDB povide multi-document transactions for replica sets.   

#### two types of references, manual references vs. dbref 
 manual reference is used when referencing documents in the same collection   
 dbref is used when referencing documents in the different collections.  
##### example of a manual reference 
Document structure for a Class: 
```
{
 name: '1A',
 start_date: '2019-09-01',
 description: 'grade 1 A - EFI',
 students: [?] 
}
```

Document structure for a Student: 
```
{
 firstName: "Yellow",
 lastName:  "Little",
 birthday: "2012-01-01",
 hobbies: "hockey, swimming, reading"
}
```

We put these two separate document structures in the same collection "class".
```
db.class.save ({
 firstName: "Yellow",
 lastName:  "L",
 birthday: "2012-01-01",
 hobbies: "hockey, swimming, reading"
});
db.class.save ({
 firstName: "Blue",
 lastName:  "Z",
 birthday: "2012-02-01",
 hobbies: "hiking, dance"
});
rs1:PRIMARY> db.class.find().pretty()
{
        "_id" : ObjectId("5c38ca2c3ec9b7b9e4144815"),
        "firstName" : "Yellow",
        "lastName" : "L",
        "birthday" : "2012-01-01",
        "hobbies" : "hockey, swimming, reading"
}
{
        "_id" : ObjectId("5c38ca343ec9b7b9e4144816"),
        "firstName" : "Blue",
        "lastName" : "Z",
        "birthday" : "2012-02-01",
        "hobbies" : "hiking, dance"
}
```

Reference these 2 students in class 1A 
```
db.class.save ({
 name: '1A',
 start_date: '2019-09-01',
 description: 'grade 1 A - EFI',
 students: [
      ObjectId("5c38ca2c3ec9b7b9e4144815"),
      ObjectId("5c38ca343ec9b7b9e4144816")
   ] 
})
```

#### example of dbref 
There are three important fields which should be used in order to implement DBRefs relationship as follows.  
$ref − collection of the referenced document.  
$id −  _id field of the referenced document.  
$db – optional, contains the name of the database in which the referenced document is present.  
add a school record : 
```
{
 name: "peas in a pod",
 description: "public primary school",
 location: "01 march road"
}

db.schools.save ({
 name: "peas in a pod",
 description: "public primary school",
 location: "01 march road"
})
rs1:PRIMARY> db.schools.find().pretty()
{
        "_id" : ObjectId("5c38ca643ec9b7b9e4144818"),
        "name" : "peas in a pod",
        "description" : "public primary school",
        "location" : "01 march road"
}

db.class.save ({
 "_id" : ObjectId("5c38cae43ec9b7b9e4144819"),
 name: '1A',
 start_date: '2019-09-01',
 description: 'grade 1 A - EFI',
 students: [
      ObjectId("5c38ca2c3ec9b7b9e4144815"),
      ObjectId("5c38ca343ec9b7b9e4144816")
   ], 
    school: {
        "$ref": "schools",
        "$id": ObjectId("5c38ca643ec9b7b9e4144818")}
 });

rs1:PRIMARY> db.class.find().pretty()
{
        "_id" : ObjectId("5c38ca2c3ec9b7b9e4144815"),
        "firstName" : "Yellow",
        "lastName" : "L",
        "birthday" : "2012-01-01",
        "hobbies" : "hockey, swimming, reading"
}
{
        "_id" : ObjectId("5c38ca343ec9b7b9e4144816"),
        "firstName" : "Blue",
        "lastName" : "Z",
        "birthday" : "2012-02-01",
        "hobbies" : "hiking, dance"
}
{
        "_id" : ObjectId("5c38ca573ec9b7b9e4144817"),
        "name" : "1A",
        "start_date" : "2019-09-01",
        "description" : "grade 1 A - EFI",
        "students" : [
                ObjectId("5c38ca2c3ec9b7b9e4144815"),
                ObjectId("5c38ca343ec9b7b9e4144816")
        ]
}
{
        "_id" : ObjectId("5c38cae43ec9b7b9e4144819"),
        "name" : "1A",
        "start_date" : "2019-09-01",
        "description" : "grade 1 A - EFI",
        "students" : [
                ObjectId("5c38ca2c3ec9b7b9e4144815"),
                ObjectId("5c38ca343ec9b7b9e4144816")
        ],
        "school" : DBRef("schools", ObjectId("5c38ca643ec9b7b9e4144818"))
}
```
  
### Q1 - save vs. insert 
save is a wrapper for update and insert. Functionally, save and insert are very similar, especially if no _id value is passed. However, if an _id key is passed, save() will update the document, while insert() will throw a duplicate key error.  
  
## References
official document: https://docs.mongodb.com/manual/core/data-modeling-introduction/   
safari books - 50 tips for MongoDB developers: https://www.oreilly.com/library/view/50-tips-and/9781449306779/


