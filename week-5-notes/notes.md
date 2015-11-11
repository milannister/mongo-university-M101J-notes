# Simple Aggregation Example (1-simple-agregation-example.js)
in SQL number of products from each manufacturer:
select manufacturer, count(*) from products group by manufacturer
same thing in MongoDB given object in items collection:
`{ "_id" : ObjectId("5642fa533e24ec0ee744e9cf"), "name" : "ZenFone 2", "category" : "cell phone", "manufacturer" : "Asus", "price" : 249 }`
`db.items.aggregate( [ {$group: { _id : "$manufacturer", num_products : {$sum:1}}}])`

# The Aggregation Pipeline
* aggregation uses pipline (similar as in Unix)
* stages:
collection -> $project -> $match -> $group -> $sort -> result
* each stage can appear multiple times in a pipeline
* each stage is presented as an array element in syntax

1. `$project` - reshape the document - 1 : 1 (for each document that enters the stage one documents leaves the stage)
2. `$match` - filtering stage - n : 1
3. `$group` - aggregate (allowes summing and counting) - n : 1
4. `$sort` - sort - 1:1
5. `$skip` - skips - n:1
6. `$limit` - limits - n:1
7. `$unwind` - normalize data (creates the explosion of data) - 1 : n
8. `$out` - output (redirect the output to a collection) - 1:1
9. `$redact` - security related feature (authorizations)
10. `$geonear`

# Simple Example Expended (1-simple-agregation-example.js)
`db.items.aggregate( [ {$group: { _id : "$manufacturer", num_products : {$sum:1}}}])`
it will
1. process each of the documents from the input
2. upsert new document to the output collection with _id that is equal to the value of the 'manufacturer' property in the doc from the input
3. since upsert is done if the id does not exist it will create the doc and for num_products property it will add 1 to the previous value (at start it is 0)

example:
1. first document comes in `{ "_id" : ObjectId("5642f8ca3e24ec0ee744e9c6"), "name" : "Nexus 7", "category" : "Tablets", "manufacturer" : "Google", "price" : 199 }`
2. upsert document with id 'Google' (in this case it will insert it) and num_products property with value 0
...
3. next document comes in `{ "_id" : ObjectId("5642f9f63e24ec0ee744e9cb"), "name" : "Nexus 5", "category" : "cell phone", "manufacturer" : "Google", "price" : 349 }`
4. upsert document with id 'Google' (in this case it will update it) and num_products property with value incremented by 1

# Compound Grouping (1-simple-agregation-example.js)
* number of products that each manufacturer has in each category
  * in SQL: 
```SQL
select manufacturer, category, count(*) from products group by manufacturer, category
```
  * in aggregation framework using compaund id key `_id : {maker : "$manufacturer", category : "$category"}`
```
db.items.aggregate( [ { $group : {_id : {"maker" : "$manufacturer", "category" : "$category"}, num_products: {$sum : 1}}}])
```
will result with:
```json
{ "_id" : { "maker" : "Sony", "category" : "cell phone" }, "num_products" : 1 }
{ "_id" : { "maker" : "Samsung", "category" : "cell phone" }, "num_products" : 2 }
{ "_id" : { "maker" : "Google", "category" : "cell phone" }, "num_products" : 1 }
{ "_id" : { "maker" : "Apple", "category" : "cell phone" }, "num_products" : 1 }
```
...

* compound id key can be used to improve readability.
Instead  `db.items.aggregate( [ {$group: { _id : "$manufacturer", num_products : {$sum:1}}}])`
it would be more readable to have 
`db.items.aggregate( [ {$group: { _id : {"manufacturer" :"$manufacturer"}, num_products : {$sum:1}}}])`
which results with:
```
{ "_id" : { "manufacturer" : "Asus" }, "num_products" : 1 }
{ "_id" : { "manufacturer" : "Samsung" }, "num_products" : 2 }
...
```
# Using a document for _id
* it's possible to have document as a value for _id
` db.foo.insert( { _id : { name : "milan", birthyear: 1986}, hometown : "Zrenjanin"}) `

# Aggregation Expressions (in $group stage)
* $sum - count if we adds 1, if we add value of the key it will sum up values
* $avg - averages value of the key across documents
* $min - finds the minimum value of the key in the documents
* $max - finds the maximum value of the key in the documents
* $push - builds an array
* $addToSet - build an array
* $first - (require to sort the documents first) find the first value for the key that it see across the documents
* $last - (require to sort the documents first) find the last value for the key that it see across the documents


# Using $sum (1-simple-agregation-example.js, zipcodes.js)
* sum up the prices that each manufacturer charges
`db.items.aggregate([ {$group : { _id : {"maker" : "$manufacturer"}, sum_prices : {$sum : "$price"}}}])`
results with:
```json
{ "_id" : { "maker" : "Asus" }, "sum_prices" : 249 }
{ "_id" : { "maker" : "Samsung" }, "sum_prices" : 948 }
{ "_id" : { "maker" : "Sony" }, "sum_prices" : 299 }
{ "_id" : { "maker" : "Amazon" }, "sum_prices" : 328 }
{ "_id" : { "maker" : "Apple" }, "sum_prices" : 1198 }
{ "_id" : { "maker" : "Google" }, "sum_prices" : 548 }
```

* quiz (zipcode.js) - sum up the population (pop) by state and put the result in a field called population.
`db.zipcode.aggregate([ {$group : { _id : "$state", "population" : {$sum : "$pop"}}}  ])`

# Using $avg (1-simple-agregation-example.js, zipcodes.js)
* average price per category
```json
db.items.aggregate( [{$group : {_id : { category : "$category" }, avg_price : { $avg : "$price"}}}])
```
* quiz: Given population data by zip code (postal code), write an aggregation expression to calculate the average population of a zip code (postal code) by state.
```json
db.zipcode.aggregate([ {$group : { _id : "$state", avg_pop : {$avg : "$pop"}}}])
```

# Using $addToSet (1-simple-agregation-example.js, zipcodes.js)
* no SQL analogy
* $addToSet creates an array and adds element to the array if the element does not already exist
* for each manufacturer list the product categories they sell
```json
db.items.aggregate([ {$group : { _id : {maker : "$manufacturer"}, categories : {$addToSet : "$category"}}}])
```
* quiz: query that will return the postal codes that cover each city. The results should look like this:
```json
{
	"_id" : "CENTREVILLE",
	"postal_codes" : [
		"22020",
		"49032",
		"39631",
		"21617",
		"35042"
	]
}
```
answer:
```json
db.zipcode.aggregate([ {$group : { _id : { city : "$city"}, zip_codes : { $addToSet : "$_id"}}}])
```

# Using $push 
* similar to $addToSet except it does not check if element already exists

# Using $max and $min  (1-simple-agregation-example.js, zipcodes.js)
* max price by manufacturer
```javascript
db.items.aggregate([ {$group : {_id : {maker : "$manufacturer" }, max_price : {$max : "$price"}}}])
```
* quiz: query that will return the population of the postal code in each state with the highest population.
```javascript
db.zipcode.aggregate([ {$group : { _id : "$state", pop : {$max : "$pop"}}}])
```

# Double $group stage (student_scores.js)
* we can group more than once
* what's the average class grade for each class
  1. aggregate average score for each student (but include class_id as a part of id in order to use it in the next step)
  2. aggregate average for each class using the average for each student

```javascript
db.student_scores.aggregate([ 
    {$group : 
        {
            _id : {class_id : "$class_id", student_id : "$student_id"},
            avg_score : {"$avg" : "$score"}
        }
    }, 
    {$group : 
        {
            _id : {class_id : "$_id.class_id"},
            avg_score : {$avg : "$avg_score"}
        }
    }
])
```
* try out the first step only and see the result to make it more clear what is going on
* side note: We cannot just sum all the scores in all the classes and divide by number of total scores; see [average of average](http://math.stackexchange.com/questions/95909/why-is-an-average-of-an-average-usually-incorrect)

# Using $project (1-simple-agregation-example.js, zipcodes.js)
* let us reshape the documents (1:1)
  * remove keys
  * add keys
  * reshape keys
  * use some simple functions on keys
    * $toUpper
    * $toLower
    * $add
    * $multiply

* example: let's reshape documents from items collection
  * exclude id
  * lowercase manufacturer and change the key to 'maker'
  * add 'details' key with 'category' and price multiplied by 10
  * change the key 'name' to 'item'
```javascript
db.items.aggregate([{
    $project: { 
        _id : 0, 
        maker : {
            $toLower: "$manufacturer"
        }, 
        'details' : {
            category : "$category",
            price : {"$multiply":["$price", 10]}
        }, 
        item: "$name"
    }
}])
```

* quiz: Write an aggregation query with a single projection stage that will transform the documents in the zips collection from this:
```json
{
	"city" : "ACMAR",
	"loc" : [
		-86.51557,
		33.584132
	],
	"pop" : 6055,
	"state" : "AL",
	"_id" : "35004"
}
```
to this:
```json
{
	"city" : "acmar",
	"pop" : 6055,
	"state" : "AL",
	"zip" : "35004"
}
```
answer is:
```json
db.zipcode.aggregate([ {
    $project : {
        _id : 0,
        city : {$toLower: "$city"},
        pop : 1, 
        state : 1,
        zip : "$_id"
    }
}])
```
* note: If you don't mention a key, it is not included, except for _id, which must be explicitly suppressed. If you want to include a key exactly as it is named in the source document, you just write key:1, where key is the name of the key. You will probably get more out of this quiz is you download the zips.json file and practice in the shell. zips.json link is in the using $sum quiz

# Using $match (zipcodes.js)
* performs a filtering (n:1) - if document matches the criteria it will push it to the next stage of the pipeline
* find population for each city in California (CA) and its zip codes
  1. filter on 'state': "CA"
  2. group on city (sum on population and add zip codes)
```javascript
db.zipcode.aggregate([{
    $match : { state : "CA"}},
    {
    $group : {
        _id : "$city",
        population : {$sum : "$pop"},
        zip_codes : {$addToSet: $_id}}
    }
])
```
* quiz: Write an aggregation query with a single match phase that filters for zipcodes with greater than 100,000 people.
```javascript
db.zipcode.aggregate([ {$match : { pop : {$gt : 100000}}}])
```

* note: $match (and $sort) is that they can use indexes, but only if done at the beginning of the aggregation pipeline

# Using $sort (zipcodes.js)
* disk sorting
* memory based sorting (default) - limit 100MB for any given pipeline stage
* before or after the grouping stage
* example: show population per city in state NY sorted from largest to smallest
```javascript
db.zipcode.aggregate([ {$match : {state : "NY"}}, {$group : { _id : "$city", population : {$sum : "$pop"}}}, {$project: {_id : 0, city : "$_id", population: 1}}, {$sort : {population : -1}}])
```
* quiz: Write an aggregation query with just a sort stage to sort by (state, city), both ascending.
```javascript
db.zipcode.aggregate([ {$sort : { state : 1, city : 1}}])
```

# Using $limit and $skip
* make sense only if you first do the sory
* make sense only first to $skip then to $limit
* example:
```javascript
db.zipcode.aggregate([ {$match : {state : "NY"}}, {$group : { _id : "$city", population : {$sum : "$pop"}}}, {$project: {_id : 0, city : "$_id", population: 1}}, {$sort : {population : -1}}, {$skip : 10}, {$limit: 5}])
```

# Revisiting $first and $last (zipcodes.js)

* find largest city in each state, phases:
  1. get the population of every city in every state
  2. sort by state, population
  3. group by state, get the first item in each group
  4. sort by state (which in _id)

```javascript
db.zipcode.aggregate([ {$group : {_id : {state : "$state", city : "$city"},  population : {$sum : "$pop"}}}, {$sort : {"_id.state" : 1, "population" : -1}}, {$group : {_id : "$_id.state", city : {$first : "$_id.city"}, population : {$first : "$population"}}}, {$sort : {"_id" : 1}}])
```

# Using $unwind

* used to flatten the arrays (creates one document for each element in the array)

# $unwind example (posts.json)

* find how many times each tag appears in the posts
  * $unwind by tags
  * group by tag and count each tag
  * sort by popularity
  * limit to 10
  * change the name to be tag
```javascript
db.posts.aggregate([ {$unwind : "$tags"}, {$group : {"_id" : "$tags", count : {$sum : 1}}}, {$sort : {count : -1}}, {$limit : 10}, {$project : {tag : "$_id", count : 1, "_id" : 0}}])
```
* note : $push can be used to reverse the effect of $unwind

# Double $unwind (inventory.js)

* used for more than one array in a document
* find number of products available in any given size of color regardless of name of the product
```javascript
db.inventory.aggregate([ {$unwind: "$sizes"}, {$unwind: "$colors"}, {$group :  { "_id" : {"size": "$sizes", "color" : "$colors"}, "count" : {$sum : 1}}}])
```

* note: two pushes in a row can be used to reverse effect of double $unwind
```javascript
db.inventory.aggregate([
    {$unwind: "$sizes"},
    {$unwind: "$colors"},
    /* create the color array */
    {$group: 
     {
	'_id': {name:"$name",size:"$sizes"},
	 'colors': {$push: "$colors"},
     }
    },
    /* create the size array */
    {$group: 
     {
	'_id': {'name':"$_id.name",
		'colors' : "$colors"},
	 'sizes': {$push: "$_id.size"}
     }
    },
    /* reshape for beauty */
    {$project: 
     {
	 _id:0,
	 "name":"$_id.name",
	 "sizes":1,
	 "colors": "$_id.colors"
     }
    }
])
```
# Mapping between SQL and Aggregation

* WHERE - $match
* GROUP BY - $group
* HAVING - $match
* SELECT - $project
* ORDER BY - $sort
* LIMIT - $limit
* SUM - $sum
* COUNT - $count
* join - N/A

# Some Common SQL Examples

* see [documentation](https://docs.mongodb.org/manual/reference/sql-comparison)

# Limitations of Aggregation Framework

* 100MB for pipeline stages - solve with `allowDiskUse`
* 16MB limit for single document (might be problem if result should be returned in single document, which is default in python - solution for pyhtno init `curs = {}`)
* sharded - group by, sort -> returns data to the primary shard -> performance of agregation framework on sharded system are not as good as they are for, say, hadoop
  * can be used hadoop connector
  * mongo's map/reduce - not recommanded

# Aggregation Framework with Java Driver (zipcodes.js)
```Java
List<Document> pipeline;
pipeline = Arrays.asList(new Document("$group", new Document("_id", "$state").append("totalPop", new Document("$sum", "$pop"))),
                        new Document("$match", new Document("totalPop", new Docuement("$gte", 10000000))));
List<Document> results = collection.aggregate(pipeline).into(new ArrayList<Document>());
```
or better to use Document.parse(String) for this kind of queries:
```Java
pipeline = Arrays.asList(Document.parse("{ $group: { _id : \"$state\", totalPop: { $sum: \"$pop\" }}}"),
                        Document.parse("{ $match: { totalPop : { $gte: 10000000 }}}"));
```






















