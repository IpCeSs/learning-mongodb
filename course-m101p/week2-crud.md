# M101P · Week 2 · CRUD

Reading notes and homework related to course [Week 2: CRUD](https://education.mongodb.com/courses/10gen/M101P/2014_February/courseware/Week_2_CRUD/) .

## Recap

### CRUD, Mongo Shell and BSON types
* CRUD is IFUR: `insert()`, `find()`, `update()`, `remove()`
* 1. **sort** 2. **skip** 3. **limit**; sort order is _lexicographic_, according to the binary UTF-8 encoding of strings
* Recap on JS properties and dictionary lookup
* Data is stored in [BSON](http://bsonspec.org) format in MongoDB, which is a superset of JSON, having namely _ObjectId_, _date_, _integer 32/64_ and _binary_ types
* MongoDB Shell command-line editing tips, namely history and autocomplete; the _shell_ has support for the various MongoDB types: `Date()` (gets converted to `ISODate()` automatically), `NumberInt()`, `NumberLong()` for instance
* ObjectId is a UUID; each doc must have a PK, which is immutable in the DB

### Inserting
* Inserting: `db.fruits.insert({ name: "apple", color: "red", shape: "round"})`

### Querying
* `.find(…).pretty()` displays results in a more human readable form
* Finding: `find()`, `findOne()`, excluding fields from output, specifying query by example; for instance: `db.scores.find({ type: "essay", score: 50}, { student: true, _id: false})`
* All search ops are strongly typed and dynamically typed; when using polymorphic field contents, beware! such searches can be refined with `$type` (for instance, `.find( { name: { $type: 2 } })` with _type_ according to [BSON spec](http://bsonspec.org/#/specification) of element types; type 2 being a string) and `$exists`(for instance, `.find( { profession: { $exists: true | false }})`)
* Querying for string patterns with regular expressions: `.find( { name: { $regex: "e$" }} )`; regular expressions tend not to be efficiently optimized, except a few cases, such as expressions starting with a caret (for instance, `$regex: "^A"`)
* Combining operators: `db.users.find({"name": {$regex: "q"}, "email": {$exists: true}})`
* Combining expressions on the same field with `$or: [ …, …]` and `$and: [ …, …]`: `db.scores.find({ $or: [ { score: { $lt: 50}}, { score: { $gt: 90}} ] })`; if the expression are bound to different fields, `$and` or `$or` are not needed
* Beware! Javascript will silently parse the following, although it is probably an incomplete query expression: `db.scores.find( { score: { $lt: 50}}, { score: { $gt: 90}})` will return all documents with score greater than 90 – the second property definition replaces the first
* Querying arrays: matching is polymorphic over arrays (top level only); `db.products.find( { tags: "shiny"})` would match `{ _id: 42, name: "wiz-o-matic", tags: [ "awesome", "shiny", "green"]}` as well as `{ _id: 42, name: "snap-o-lux", tags: "shiny"}`
* Querying for all given values with `$all: [ …, …]`: `db.accounts.find( { favorites: { $all: [ "pretzels", "beer"] }})` the array values have to be a subset of the values of the field that is queried for, in any order
* Querying for any of the given values with `$in: […, …]`: `db.accounts.find( { name: { $in: [ "Howard", "John" ]}})`
* Combining both `$all` and `$in`: `db.users.find( { friends: { $all: [ "Joe" , "Bob" ] }, favorites: { $in: [ "running" , "pickles" ]}})` will match `{ name : "Cliff" , friends : [ "Pete" , "Joe" , "Tom" , "Bob" ] , favorites : [ "pickles", "cycling" ] }`
* Querying on fields in embedded documents, beware: in `db.users.find( { email: { work: "richard@10gen.com", personal: "kreuter@home.org"}})` the order of the fields in the embedded document has to match the order of the fields in the database! subdocuments in queries by example are compared byte to byte to the database's contents. So the following query won't return results if all subdocs have a `"personal"` field: ``db.users.find( { email: { work: "richard@10gen.com"}})``
* Queries with Dot notation: use **dot notation** to recurse into subdocuments and look for **any** matching field: `db.users.find( { "email.work": "richard@10gen.com" })`; dot notation allows to reach fields inside a subdocument, without any knowledge of the surrounding fields; to query for all products that cost more than 10,000 and that have a rating of 5 or better: `db.catalog.find( { price: { $gt: 10000}, "reviews.rating": { $gte: 5}})`
* Query cursors, to iterate programmatically: `cur = db.people.find(); null; while( cur.hasNext()) printjson( cur.next());`; cursors have many methods: `cur.limit(5)`, `cur.sort({ name: -1})`, `cur.skip(2)`; these modifiers return the cursor, so they can be chained: `cur.sort({ name: -1}).limit(3)`; must be called before retrieving any result
* Query modifiers are executed in the server (not the client) in following order: 1. sort 2. skip 3. limit; `db.scores.find( { type: "exam"}).sort( { score: -1}).skip( 50).limit( 20)` retrieves exam documents, sorted by score in descending order, skipping the first 50 and showing only the next 20.

### Counting
* Counting results of a query: use `db.coll.count( …)` with a query I would give to `db.coll.find( …)`

### Updating

In the Mongo Shell, the API for `update()` does 4 different things:

1. **Wholesale updating** – a bit dangerous, completely replaces the documents content: `db.people.update( { name: "Smith"}, { name: "Thompson", salary: 50000})`will replace all documents matching the primary keys returned by the first query, discard their content and replace it with the second document – keeping only the primary key value

2. **Manipulating individual fields** with `$set`: `db.people.update( { name: "Alice"}, { $set: { age: 30}})` will define the field `age` if it doesn't exist, or modify its value if it exists.

   Similary, `$inc` allows to modify a value, or define it if it doesn't exist, with the value of the increment step: `db.people.update( { name: "Alice"}, { $inc: { age: 1 }})`; these operations are efficient in MongoDB.

   Manipulating arrays in documents with `$push`, `$pop`, `$pull`, `$pushAll`, `$pullAll` and `addToSet`; note that `pop` removes one or more elements from the end of the array (or beginning if the given element count is negative), while `pull` removes the actual given values; `addToSet` considers the array as a set, rather than an ordered list: it will push a value if it doesn't exist, otherwise do nothing.

   Removing a field and its value with `$unset`: `db.people.find({ name: "Jones"}, { $unset: { profession: 1}})` (some value must be specified, 1 in this example, but it is ignored); `$unset` may be used to change the schema: `db.users.update( {}, { $unset: { interests: 1}}, { multi: true})` (see multi-update below for the `multi` extra argument)

3. **Upserting** with the `.update( …, …, { upsert: true})` third optional argument: update a record that does exist, otherwise insert a new document; frequent use case when merging data from a data vendor: `db.users.update( { name: "George"}, { $set: { interests: [ "cat", "dog"]}}, { upsert: true})` will create a document { name: "George", interests: [ "cat", "dog"]} if it doesn't already exist, otherwise update the existing document.

4. Updating **multiple documents** in a collection: default behavior of MongoDB is to update just one document in the collection; add the `{multi:true}` extra argument to update all matching documents: `db.scores.update( { score: {$lt: 70}}, {$inc: { score: 20}}, { multi:true}))` will give every document with a score less than 70 an extra 20 points; `{multi:true}` is for Javascript – some drivers have a separate method for multi-updates.

   These multi-updates are atomically executed for each document; however, from the concurrency point of view, the server does not offer isolation: these updates are a sequential collection of updates, executed in a single thread, that might however be _yielded_ (paused) by the server allow for other write and read operations.

## Homework

* [Homework 2.1](hw2-1-answer.md) Find all exam scores greater than or equal to 65, and sort those scores from lowest to highest  `db.grades.find( { "score": { $gte: 65}}).sort( { "score": 1})`
* [Homework 2.2](hw2-2-answer.md) Identity of the student with the highest average in the class
* [Homework 2.3](hw2-3-answer.md) Adapt `userDAO.py` to add a new user upon sign-up and validate a login by retrieving the right user document.