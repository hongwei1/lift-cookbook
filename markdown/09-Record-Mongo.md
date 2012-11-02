MongoDB persistence with Record
===============================

This section gives recipes for making use of MongoDB in your Lift application. You should get acquainted with the [Lift Wiki pages for Mongo](https://www.assembla.com/spaces/liftweb/wiki/MongoDB). For recipes regarding Mongo itself, check out the [MongoDB Cookbook](http://cookbook.mongodb.org/).



Connecting to a Mongo database
=========================

Problem
-------

You want to connect to a MongoDB.

Solution
--------

Configure your connection using `net.liftweb.mongodb` and `com.mongodb` in `Boot.scala`:

```scala
import com.mongodb.{ServerAddress, Mongo}
import net.liftweb.mongodb.{MongoDB,DefaultMongoIdentifier}

val srvr = new ServerAddress("127.0.0.1", "20717")
MongoDB.defineDb(DefaultMongoIdentifier, new Mongo(srvr), "mydb")
```

This will give you a connection to a local MongoDB database called "mydb".


Discussion
----------

If your database needs authentication, use `MongoDB.defineDbAuth`:

```scala
MongoDB.defineDbAuth(DefaultMongoIdentifier, new Mongo(srvr), 
  "mydb", "username", "password")
```

Some cloud services will give you a URL to connect to, such as "alex.mongohq.com:10050/fglvBskrsdsdsDaGNs1".  In this case, the host and the port make up the first part, and the database name is the part after the `/`.


See Also
--------

* [Mongo Configuration](https://www.assembla.com/wiki/show/liftweb/Mongo_Configuration) on the wiki contains more details and options, including configuration for replica sets.






Storing a hash in a Mongo record
================================

Problem
-------

You want to store a hash map in Mongo.

Solution
--------

Create a Mongo Record which contains a `MongoMapField`:

```scala
import net.liftweb.mongodb._
import net.liftweb.mongodb.record._
import net.liftweb.mongodb.record.field._

class Country private () extends MongoRecord[Country] with StringPk[Country] { 
  override def meta = Country   
  val population = new MongoMapField[Country,Int](this)
}
  
object Country extends Country with MongoMetaRecord[Country] {
  override def collectionName = "example.earth"
}
```

In this example we are creating a record for infomation about a country, and the `population` is a map from a `String` key to an `Integer` value.

We can use it in a snippet like this:

```scala
class HelloWorld {
  
  val uk = Country.find("uk") openOr {
    val info = Map("Brighton" -> 134293,  "Birmingham" -> 970892, 
       "Liverpool" -> 469017)
    Country.createRecord.id("uk").population(info).save(true)
  }
  
  def facts = "#facts" #> (
      for { (name,pop) <- uk.population.is} yield 
        ".name *" #> name & ".pop *" #> pop
      )  
}
```

When this snippet is called, it looks up a record by `_id` of "uk" or creates it using some canned information.  The template to go with the snippet could include:

```html
<div class="lift:HelloWorld.facts">
 <table>
  <thead>
   <tr><th>City</th><th>Population</th></tr>
  </thead>
  <tbody>
   <tr id="facts">
    <td class="name">Name here</td><td class="pop">Population</td>
   </tr>
  </tbody>
 </table>
</div>
```

In Mongo the resulting data structure would be:

```
\$ mongo scratch
MongoDB shell version: 2.0.7
connecting to: scratch
> show collections
example.earth
system.indexes
> db.example.earth.find()
{ "_id" : "uk", "population" : { "Brighton" : 134293, 
    "Birmingham" : 970892, "Liverpool" : 469017 } }
```

Discussion
----------

The default value for the map will be an empty map, represented in Mongo as `({ "_id" : "uk", "population" : { } })`

To insert data from your snippet, you can modify the record to supply a new `Map`:

```scala
uk.population(uk.population.is + ("Westminster"->81766)).save(true)
```

To access an individual element of the map, you can use `get` (or `value`):

```scala
uk.population.get("San Francisco")
// will throw java.util.NoSuchElementException
```

…or you can access via the standard Scala map interface:

```scala
val sf : Option[Int] = uk.population.is.get("San Francisco")
```

### What the map can contain

You should be aware that `MongoMapField` supports only primitive types at the moment.

The mapped filed is typed, `String => Int` in this case, but of course Mongo will let you mix types. If you do modify the Mongo record in the database and mix types, you'll get a `java.lang.ClassCastException` at runtime.


See Also
--------

* [MongoMapField Query](https://groups.google.com/forum/?fromgroups=#!topic/liftweb/XoseG-8mIPc) mailing list discussion.



Embedding a document inside a Mongo record
================================

Problem
-------

You have a Mongo record, and you want to embed another set of values inside it as a single entity.


Solution
--------

Use `BsonRecord` to define the document to embed, and embed it using `BsonRecordField`.  Here's an example of storing information about an image within a record:

```scala
class Image private () extends BsonRecord[Image] {
  def meta = Image
  object url extends StringField(this, 1024)
  object width extends IntField(this)
  object height extends IntField(this)
}

object Image extends Image with BsonMetaRecord[Image]
```

We can reference instances of the `Image` class via `BsonRecordField`:

```scala

class Country private () extends MongoRecord[Country] with StringPk[Country] {
  override def meta = Country
  object flag extends BsonRecordField(this, Image)
}

object Country extends Country with MongoMetaRecord[Country] {
  override def collectionName = "example.earth"
}
```

To associate a value:

```scala
val unionJack = 
  Image.createRecord.url("http://bit.ly/unionflag200").width(200).height(100)

uk.createRecord.id("uk").flag(unionJack).save(true)
```

In Mongo, the resulting data structure would be:

```
> db.example.earth.findOne()
{
  "_id" : "uk",
  "flag" : {
    "url" : "http://bit.ly/unionflag200",
    "width" : 200,
    "height" : 100
  }
}
```


Discussion
----------

If you don't set a value on the embedded document, the default will be saved as:

```
"flag" : { "width" : 0, "height" : 0, "url" : "" } 
```

You can prevent this by making the image optional:

```scala
object image extends BsonRecordField(this, Image) {
  override def optional_? = true
}
```

With `optional_?` set in this way the image part of the Mongo document won't be saved if the value is not set.  Within Scala you will then want to access the value with `valueBox` call:

```scala
val img : Box[Image] = uk.flag.valueBox
```

In fact, regardless of the setting of `optional_?` you can access the value using `valueBox`.


An alternative is to always provide a default value for the embedded document:

```scala
object image extends BsonRecordField(this, Image) {
 override def defaultValue = 
  Image.createRecord.url("http://bit.ly/unionflag200").width(200).height(100)
}
```


See Also
--------

* [Mongo Record Embedded Objects
](https://www.assembla.com/spaces/liftweb/wiki/Mongo_Record_Embedded_Objects) on the Lift Wiki.



Linking between Mongo records
================================

Problem
-------

You have a Mongo record and want to include a link to another record.

Solution
--------

Create a reference using a `MongoRefField` such as `ObjectIdRefField` or `StringRefField`, and dereference the record using the `obj` call.

As an example we can create records representing countries, where a country references the planet where you can find it:

```scala
class Planet private() extends MongoRecord[Planet] with StringPk[Planet] {
  override def meta = Planet
  object review extends StringField(this,1024)
}

object Planet extends Planet with MongoMetaRecord[Planet] {
  override def collectionName = "example.planet"
}

class Country private () extends MongoRecord[Country] with StringPk[Country] {
  override def meta = Country
  object planet extends StringRefField(this, Planet, 128)
}

object Country extends Country with MongoMetaRecord[Country] {
  override def collectionName = "example.country"
}
```

In a snippet we can make us of the link:

```scala
class HelloWorld {

  val uk = Country.find("uk") openOr {
    val earth = Planet.createRecord.id("earth").review("Harmless").save(true)
    Country.createRecord.id("uk").planet(earth.id.is).save(true)
  }

  def facts = 
    ".country *" #> uk.id &
    ".planet" #> uk.planet.obj.map{ p =>
      ".name *" #> p.id &
      ".review *" #> p.review }
  }
```

For the value `uk` we lookup an existing record, or create one if none is found.  Note that `earth` is created as a separate Mongo record, and then referenced in the `planet` field with the id of the planet.

Retrieving the reference is via the `obj` method, which returns a `Box[Planet]` in this example.


Discussion
----------

Referenced records are fetched from Mongo when you call the `obj` method on a `MongoRefField`.  You can see this by turning on logging in the Mongo driver. Do this by adding the following to the start of your `Boot.scala`:

```scala
System.setProperty("DEBUG.MONGO", "true")
System.setProperty("DB.TRACE", "true")
```

Having done this, the first time you run the snippet above your console will include:

	INFO: find: scratch.example.country { "_id" : "uk"}
	INFO: update: scratch.example.planet { "_id" : "earth"} { "_id" : "earth" , 
	    "review" : "Harmless"}
	INFO: update: scratch.example.country { "_id" : "uk"} { "_id" : "uk" ,
	    "planet" : "earth"}
	INFO: find: scratch.example.planet { "_id" : "earth"}

What you're seeing here is the initial look up for "uk", followed by the creation of the "earth" record and an update which is saving the "uk" record.  Finally, there is a lookup of "earth" when `uk.obj` is called in the `fact` method.

The `obj` call will cache the `planet` reference.  That means you could say...

```scala
".country *" #> uk.id &
".planet *" #> uk.planet.obj.map(_.id) &
".review *" #> uk.planet.obj.map(_.review)
```

...and you'd still only see one query for the "earth" record despite calling `obj` multiple times.  The flip side of that is if the "earth" record was updated elsewhere in Mongo after you called `obj`, you would not see the change from a call to `uk.obj` unless you reloaded the `uk` record first.

### Querying by reference

Searching for records by a reference is straight-forward:

```scala
val earth : Planet = ...
val onEarth : List[Country]= Country.findAll(Country.planet.name, earth.id.is)
```

Or in this case, because we have String references, we could just say:

```scala
val onEarth : List[Country]= Country.findAll(Country.planet.name, "earth")
```

### Updating and deleting

Updating a reference is as you'd expect:

```scala
uk.planet.obj.foreach(_.review("Mostly harmless.").update)
```

This would result in:

	INFO: update: scratch.example.planet { "_id" : "earth"} { "\$set" : {
	   "review" : "Mostly harmless."}}

A `uk.planet.obj` call will now return a planet with the new review.

Or you could replace the reference with another:

```scala
uk.planet( Planet.createRecord.id("mars").save(true).id.is ).save(true)
```

To remove the reference:

```scala
uk.planet(Empty).save(true)
```
This removes the link, but the Mongo record pointed to by the link will remain in the database. If you remove the object being referenced, a later call to `obj` will return an `Empty` box.

### Types of link

The example uses a `StringRefField` as the Mongo records themselves use plain strings as `_id`s, and as such we had to set the size of the string we are storing (128).  Other reference types are:

* `ObjectIdRefField` - possibly the most frequently used kind of reference, when you want to reference via the usual default `ObjectId` reference in Mongo.
* `UUIDRefField` - for records with an id based on `java.util.UUID`.
* `StringRefField` - as used in this example.
* `IntRefField` and `LongRefField` - for when you're using a numeric value as an id.


See Also
--------

* [Mongo Record Referenced Objects](https://www.assembla.com/wiki/show/liftweb/Mongo_Record_Referenced_Objects) wiki page.
* [Configure logging for the MongoDB Java driver](http://stackoverflow.com/questions/9545341/configure-logging-for-the-mongodb-java-driver) on Stackoverflow.

Storing geospatial values
=========================

Problem
-------

You want to store (lat,lon) information in Mongo.

Solution
--------

Create a `Geo` container and use when you need it in your model. For example:

```scala
class Geo extends BsonRecord[Geo] {
  def meta = Geo

  object lat extends DoubleField(this)
  object lon extends DoubleField(this)
}

object Geo extends Geo with BsonMetaRecord[Geo]
```

You can reference instances in your schema:

```scala
class Thing private () extends MongoRecord[Thing] {
  override def meta = Thing

  val loc = new BsonRecordField(this,Geo) {
    override def optional_? = true
  }
}

object Thing extends Thing with MongoMetaRecord[Thing] {
  import mongodb.BsonDSL._
  ensureIndex(loc.name -> "2d", unique=false)
}
```

To store a value you could do something like this:

```scala
val place = Geo.createRecord.lat(50.816673d).lon(-0.13441d)
val thing = Thing.createRecord.loc(place).save(true)
```

This will produce data in Mongo that looks like this:

```
{ "loc" : { "lon" : -0.13441, "lat" : 50.816673 } }
```


Discussion
----------

The `unique=false` in the `ensureIndex` highlights that you can control whether locations needs to be unique (no duplications) or not.


See Also
--------

* Mailing list discussion of [Geospatial indexing on lift-mongodb-record](https://groups.google.com/d/topic/liftweb/qTCry26wfOc/discussion).
* Mongo's [Geospacial Indexing](http://www.mongodb.org/display/DOCS/Geospatial+Indexing) page.


