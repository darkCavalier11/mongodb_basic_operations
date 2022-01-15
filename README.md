# MongoDB
### Intro
* ObjectID is 12 bytes represented in a string of 24 hexadecimal digit. 1 byte is for 2 hexadecimal digits.
0-4 -> seconds from epoch
5-9 -> Random number
10-12(inclusive) -> A counter from a random number
```
 > db.movies.updateOne
	// source code of updateone method
```
```
> mongo script1.js script2.js
	// running scripts on connected mongo db
```

### CRUD
* `insertMany` is more efficient for inserting many documents to the db and has a limit of 48MB per batch.
* If we specify ordered to false then db try to insert all documents after the error occurs. if true insert operation stops after an error encountered.
```
.insertMany([...], {"ordered":false})
```
* Each document has a limit of 16MB.
* `deleteOne({"k":"v"})` deletes the document that it found first and `deleteMany({"k":"v"})` deletes many.
* `db.coll.drop()` to drop the collection.
* `db.coll.replaceOne({"k":"v"}, {...new document})` to replace a document.
* Update operations are special as they use update operators.
	* `$set` operator sets value of a field and creates the field if not exist.
`.updateOne({"k":"v"}, {"$set":{"field":"value"}})`. for updating embedded documents use `.updateOne({"k":"v"}, {"$set":{"field.subfield":"value"}})` and to remove a field use `$unset` like `.updateOne({"k":"v"}, {"$unset":{"field":1}})` 
	* `$inc` operator used to change numeric key or generate if not present(with initial value 0) and only applicable on *numeric types*. 
`.updateOne({"k":"v"}, {"$inc":{"field":value}}) -> value must be a numeric type`  
	* **Array update operators** are used only on keys with array values
		* `$push` used to append element to the array and array is created if not present already. `$push` comes handy with `$each` and `$slice`.  `$each` operator insert each element of the array to the document array i.e. here `name` and `$slice` limits the size of the array i.e `name`. if `$slice` > 0 it means `name` array should be of size <= `$slice` . if size already == `$slice` update ignored. if `$size` < 0 the the elements were inserted in a queue fashion i.e. for each element appended an element from the start will be popped respecting the size of the `$slice`  provided. `.updateOne( { "k": "v" },{ "$push": { "name": { "$each": [4, 7, 8, 9, 2, 3], "$slice": 15 } } })`
		* `$addToSet` operator can used to make the array behave as a set and can be used with `$each` operator. The email address below get inserted only if it’s not present already.
`.updateOne({"k":"v"}, {"$addToSet":{"email":"jondoe@example.com"}})`
		* `$pop` used to pop element from array. `$pop : {"myArray": 1}` -> from end, and -1 for from start. `$pull` to remove **all instance of** particular element from the array. `$pull: {"myArray" : element}` 
		* Positional operator used to update the array at particular position
`{"$inc":{"myArray.0.value" : 17}}` -> increments the 1st element having a field value to 17. But for a more real scenario we don’t know the exact index, and in such case the numeric value (0 here) is replaced with `$` to update the **first instance** of the array that satisfies the update criteria provided.
`.updateOne( { "colors.color": "gray" }, { "$set": { "colors.$.vibrant": false } })`  -> updates the vibrant property of black as false.
		* Array filters are used to update arrays that pass certain criteria inside the document. `updateOne( { "k": "v" }, { $set: { "colors.$[elem].vibrant": true } }, {arrayFilters: [{"elem.color": {"$eq":"blue"}}]})`
	* setting `upsert:true` insert a document with query and update fields combined which save round trip to db to check if the document exists or not.`updateOne({...}, {...}, {"upsert": true})`
	* `$setOnInset` parameter similar to `$set` used to set the value once only during the insertion and not on subsequent updates.
	* `$updateMany` follows same semantic as `$updateOne` to update multiple documents.

### Querying
* `.find(<queries>, <fields>)` to query multiple data. fields here is the field you wanted where _id will always shows unless explicitly hidden. `.find({"name":"john"}, {"_id":0, "email":1})` 
* **Query operators**
	* `$lt $gt $lte $gte $eq $ne` for normal queries. `.find({"age":{"$gte":18}})` 
	* `$in $nin` used to query variety of values for a single key. `.find({"lucky_number":{"$in":[7, 11, 23]}})`  
	* `$or` used to join queries as OR operator. `.find("$or":[{"name":"John", "email":"john@example.com"}])` . use `$in` rather than `$or` for efficiency. 
	* `$exists` check if the key exists or not. `.find({"z":{"$exists":true}})`. 
	* **Querying arrays**
		* Query for array similar to query for scalar values. `find({"emails":"abc@example.com"})` returns all the results with email array containing the given email. For querying particular index of an array we can use *index* operator, for eg `.find({"emails.2":"abc@example.com"})` . 
		* `$all` used match by more than one element. `find({"emails": {"$all":["abc@example.com", "efg@example.com"]}})` 
		* `$size` used to find array of particular size.
		* `$slice` used as a second argument to refactor the document array we wanted
`.find(query, {"myArray":{"$slice":10}})` -> returns first 10 elements
`.find(query, {"myArray":{"$slice":-10}})` -> returns last 10 elements
`.find(query, {"myArray":{"$slice":[2,10]}})` -> skip 2 and returns next 10 elements
		* `$elemMatch` used for range query only on *array element* types. `$find({"myArray": {"$elemMatch":{"$gt":10, "$lt": 55}}})`. `$elemMatch` is useful because we can’t do this `$find({"myArray":{"$gt":10, "$lt": 55}})` to \[7, 65] which will be returned as a matching document because 7 < 55 and 65 > 10.
	* **Querying** embedded documents work very intuitively with . operator to get the subdocument field and update it. `find({"user.email":"abc@example.com"})`
	* `$where` clause can be used with JS function but should be restricted for usage due to slow operation and potential security issues.
* DB returns the results of a query as a cursor which is a lazy operator and not fired until the user requested the data.  So the below cursors are all the same as they are only built and not executed yet.
```javascript
var cursor = db.user.find().sort({"x": 1}).limit(1).skip(5)
var cursor = db.user.find().limit(1).sort({"x": 1}).skip(5)
var cursor = db.user.find().skip(5).sort({"x": 1}).limit(1)
```
* `skip`, `limit` and `sort` are extended cursor operator and self explanatory. `sort` with more than 1 query fields sorts from left to right order.