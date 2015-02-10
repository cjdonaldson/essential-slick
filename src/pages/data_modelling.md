# Data Modelling

Now that we can do the basics of connecting to a database, running queries, and changing data, we turn to richer models of data.

In this chapter we will:

- understand how to structure an application;
- look at alternatives to modelling rows as case classes;
- expand on our knowledge of modelling tables to introduce optional values and foreign keys; and
- use more custom types to avoid working with just low-level database values.

We'll expand chat application schema to support more than just messages through this chapter.

## Application Structure

Our examples so far have been a single monolithic application. That's now how you'd work with Slick for a more substantial project.  We'll explain how to split up an application in this section.

We've also been importing the H2 driver.  We need a driver of course, but it's useful to delay picking the driver until the code needs to be run. This will allow us to switch driver, which can be useful for testing. For example, you might use H2 for unit testing, but PostgresSQL for production purposes.

### Pattern Outline

The basic pattern we'll use is as follows:

* Isolate our schema into a trait (or a few traits) in which the Slick _profile_ is abstract.  We will often refer to this trait as "the tables".

* Create an instance of our tables using a specific profile.

* Finally, configure a `Database` to obtain a `Session` to work with the table instance.

_Profile_ is a new term for us. When we have previously written...

~~~ scala
import scala.slick.driver.H2Driver.simple._
~~~

...that is giving us H2-specific JDBC driver, which is a `JdbcProfile`, which in turn is a `RelationalProfile` provided by Slick. It means that Slick could, in principle, be used with non-JDBC-based, or indeed non-relational, databases. In other words, _profile_ is an abstraction above a specific driver.


### Example

Re-working the example from Chapter 1, we have the schema in a trait:

~~~ scala
trait Profile {
  // Place holder for a specific profile
  val profile: scala.slick.driver.JdbcProfile
}

trait Tables {
  // Self-type indicating that our tables must be mixed in with a Profile
  this: Profile =>

  // Whatever that Profile is, we import it as normal:
  import profile.simple._

  // Row and table definitions here as normal
}
~~~

We currently have a small schema and can get away with putting all the table defintions into a single trait.  However, there's nothing to stop us from splitting the schema into, say `UserTables` and `MessageTables`, and so on.  They can all be brought together with `extends` and `with`:

~~~ scala
// Bring all the components together:
class Schema(val profile: JdbcProfile) extends Tables with Profile

object Main extends App {

  // A specific schema with a particular driver:
  val schema = new Schema(scala.slick.driver.H2Driver)

  // Use the schema:
  import schema._, profile.simple._

  def db = Database.forURL("jdbc:h2:mem:chapter03", driver="org.h2.Driver")

  db.withSession {
    // Work with the database as normal here
  }
}
~~~

To work with a different database, create a different `Schema` instance and supply a different driver. The rest of the code does not need to change.

### Additional Considerations

There is a potential down-side of packaging everything into a single `Schema` and performing `import schema._`.  All your case classes, and table queries, custom methods, implicits, and other values are imported into your current namespace.

If you recognise this as a problem, it's time to split your code more finely and take care over importing just what you need.


## Representations for Rows

In Chapter 1 we introduced rows as being represented by case classes.
There are in fact three common representations used: tuples, case classes, and an experimental `HList` implementation.

### Case Classes and `<>`

To explore these different representations we'll start with comparing tuples and case classes.
For a little bit of variety, let's define a `user` table so we no longer have to store names in the `message` table.

A user will have an ID and a name. The row representation will be:

~~~ scala
final case class User(name: String, id: Long = 0L)
~~~

The schema is:

~~~ scala
final class UserTable(tag: Tag) extends Table[User](tag, "user") {
 def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def name = column[String]("name")
 def * = (name, id) <> (User.tupled, User.unapply)
}
~~~

None of this should be a surprise, as it is essentially what we have seen in the first chapter. What we'll do now is look a little bit deeper into how rows are mapped into case classes.

`Table[T]` class requires the `*` method to be defined. This _projection_ is of type `ProvenShape[T]`. A "shape" is a description of how the data in the row is to be structured. Is it to be a case class? A tuple? A combination of these? Something else? Slick provides implicit conversions from various data types into a "shape", allowing it to be sure at compile time that what you have asked for in the projection matches the schema defined.

To explain this, let's work through an example.

If we had simply tried to define the projection as a tuple...

~~~ scala
def * = (name, id)
~~~

...the compiler would tell us:

~~~
Type mismatch
 found: (scala.slick.lifted.Column[String], scala.slick.lifted.Column[Long])
 required: scala.slick.lifted.ProvenShape[chapter03.User]
~~~

This is good. We've defined the table as a `Table[User]` so we want `User` values, and the compiler has spotted that we've not defined a default projection to supply `User`s.

How do we resolve this? The answer here is to give Slick the rules it needs to prove it can convert from the `Column` values into the shape we want, which is a case class. This is the role of the mapping function, `<>`.

The two arguments to `<>` are:

* a function from `U => R`, which converts from our unpacked row-level encoding into our preferred representation; and
* a function from `R => Option[U]`, which is going the other way.

We can supply these functions by hand if we want:

~~~ scala
def intoUser(pair: (String, Long)): User = User(pair._1, pair._2)

def fromUser(user: User): Option[(String, Long)] = Some((user.name, user.id))
~~~

...and write:

~~~ scala
def * = (name, id) <> (intoUser, fromUser)
~~~

Case classes already supply these functions via `User.tupled` and `User.unapply`, so there's no point doing this.
However it is useful to know, and comes in handy for more elaborate packaging and unpackaging of rows.
We will see this in one of the exercises in this section.


### Tuples

You've seen how Slick is able to map between a tuple of columns into case classes.
However you can use tuples directly if you want, because Slick already knows how to convert from a `Column[T]` into a `T` for a variety of `T`s.

Let's return to the compile error we had above:

~~~
Type mismatch
 found: (scala.slick.lifted.Column[String], scala.slick.lifted.Column[Long])
 required: scala.slick.lifted.ProvenShape[chapter03.User]
~~~

We fixed this by supplying a mapping to our case class. We could have fixed this error by redefining the table in terms of a tuple:

~~~ scala
type TupleUser = (String, Long)

final class UserTable(tag: Tag) extends Table[TupleUser](tag, "user") {
 def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def name = column[String]("name")
 def * = (name, id)
}
~~~

As you can see there is little difference between the case class and the tuple implementations.

Unless you have a special reason for using tuples, or perhaps just a table with a single value, you'll probably use case classes for the advantages they give:

* we have a simple type to pass around (`User` vs. `(String, Long)`); and
* the fields have names, which improves readability.


<div class="callout callout-info">
**Expose Only What You Need**

We can hide information by excluding it from our row definition. The default projection controls what is returned, in what order, and it is driven by our row definition. You don't need to project all the rows, for example, when working with a table with legacy columns that aren't being used.
</div>



### Heterogeneous Lists

Slick's **experimental** [`HList`][link-slick-hlist] implementation is useful if you need to support tables with more than 22 columns,
such as a legacy database.

To motivate this, let's suppose our user table contains lots of columns for all sorts of information about a user:

~~~ scala
import scala.slick.collection.heterogenous.{ HList, HCons, HNil }
import scala.slick.collection.heterogenous.syntax._

final class UserTable(tag: Tag) extends Table[User](tag, "user") {
  def id           = column[Long]("id", O.PrimaryKey, O.AutoInc)
  def name         = column[String]("name")
  def age          = column[Int]("age")
  def gender       = column[Char]("gender")
  def height       = column[Float]("height_m")
  def weight       = column[Float]("weight_kg")
  def shoeSize     = column[Int]("shoe_size")
  def email        = column[String]("email_address")
  def phone        = column[String]("phone_number")
  def accepted     = column[Boolean]("terms")
  def sendNews     = column[Boolean]("newsletter")
  def street       = column[String]("street")
  def city         = column[String]("city")
  def country      = column[String]("country")
  def faveColor    = column[String]("fave_color")
  def faveFood     = column[String]("fave_food")
  def faveDrink    = column[String]("fave_drink")
  def faveTvShow   = column[String]("fave_show")
  def faveMovie    = column[String]("fave_movie")
  def faveSong     = column[String]("fave_song")
  def lastPurchase = column[String]("sku")
  def lastRating   = column[Int]("service_rating")
  def tellFriends  = column[Boolean]("recommend")
  def petName      = column[String]("pet")
  def partnerName  = column[String]("partner")

  def * = name :: age :: gender :: height :: weight :: shoeSize ::
          email :: phone :: accepted :: sendNews ::
          street :: city :: country ::
          faveColor :: faveFood :: faveDrink :: faveTvShow :: faveMovie :: faveSong ::
          lastPurchase :: lastRating :: tellFriends ::
          petName :: partnerName :: id :: HNil
}
~~~

I hope you don't have a table that looks like this, but it does happen.

You could try to model this with a case class. Scala 2.11 supports case classes with more than 22 arguments,
but it does not implement the `unapply` method we'd want to use for mapping.  Instead, in this situation,
we're using a _hetrogenous list_.

An HList has a mix of the properties of a list and a tuple.  It has an arbitrary length, just as a list,
but unlike a list, each element can be a different type, like a tuple.
As you can see from the `*` definition, an `Hlist` is a kind of shape that Slick knows about.

This `HList` projection needs to match with the definition of `User` in `Table[User]`. For that, we list the types in a type alias:

~~~ scala
type User =
  String :: Int :: Char :: Float :: Float :: Int ::
  String :: String :: Boolean :: Boolean ::
  String :: String :: String :: String :: String :: String :: String :: String :: String ::
  String :: Int :: Boolean ::
  String :: String  :: Long :: HNil
~~~

Typing this in by hand is error prone and likely to drive you crazy. There are two ways to improve on this:

- The first is to know that Slick can generate this code for you from an existing database. We'll look at
that in chatper **TODO**.  We expect you'd be needing `HList`s for legacy database structures, and in that case
code generate is the way to go.

- Second, you can improve the readability of `User` by _value clases_ to replace `String` with a more meaningful type.
We'll see this in section **TODO A BIT LATER IN THIS CHAPTER**.


Once you have an `HList`-based schema, you work with it in much the same way as you would other data representations.
To create an instance of an `HList` we use the cons operator and `HNil`:

~~~ scala
users +=
  "Dr. Dave Bowman" :: 43 :: 'M' :: 1.7f :: 74.2f :: 11 ::
  "dave@example.org" :: "+1555740122" :: true :: true ::
  "123 Some Street" :: "Any Town" :: "USA" ::
  "Black" :: "Ice Cream" :: "Coffee" :: "Sky at Night" :: "Silent Running" :: "Bicycle made for Two" ::
  "Acme Space Helmet" :: 10 :: true ::
  "HAL" :: "Betty" :: 0L :: HNil
~~~

A query will produce an `HList` based `User` instance.  To pull out fields you can use `head`, `apply`, `drop`, `fold`, and the
appropriate types from the `Hlist` will be preserved:

~~~ scala
val dave = users.first

val name: String = dave.head
val age: Int = dave.apply(1)
~~~

Accessing the `HList` by index is dangerous. If you run off the end of the list with `dave(99)`, you'll get a run-time exception.

We not recommending you use a `HList` representation, but you need to know it's there for you when dealing with nasty schemas.

<div class="callout callout-danger">
**Extra Dependencies**

Some parts of `HList`, notably `Nat`, had a dependency on Scala reflection. If you want to use `Nat`,
modify _build.sbt_ to include:

~~~ scala
"org.scala-lang" % "scala-reflect" % scalaVersion.value
~~~

This took one of the authors **far** to long to establish.
</div>


### Exercises

#### Turning Wide Rows into Case Classes

Our `HList` example mapped a table with many columns.
It's not the only way to deal with lots of columns.

Use custom functions with `<>` and map that table into a tree of case classes.
To do this you will need to define the schema, define a `User`, insert data, and query the data.

To make this easier, we're just going to map six of the columns.
Here are the case classes to use:

~~~ scala
case class EmailContact(name: String, email: String)
case class Address(street: String, city: String, country: String)
case class User(contact: EmailContact, address: Address, id: Long = 0L)
~~~

You'll find a definition of `UserTable` that you can copy and paste in the example code in the folder _chapter-03_.

<div class="solution">
A suitable projection is:

~~~ scala
def pack(row: (String, String, String, String, String, Long)): User =
  User(
    EmailContact(row._1, row._2),
    Address(row._3, row._4, row._5),
    row._6
  )

def unpack(user: User): Option[(String, String, String, String, String, Long)] =
  Some((user.contact.name, user.contact.email,
        user.address.street, user.address.city, user.address.country,
        user.id))

def * = (name, email, street, city, country, id) <> (pack, unpack)
~~~

We can insert and query as normal:

~~~ scala
users += User(
  EmailContact("Dr. Dave Bowman", "dave@example.org"),
  Address("123 Some Street", "Any Town", "USA")
)
~~~

Executing `users.run` will produce:

~~~ scala
Vector(
  User(
    EmailContact(Dr. Dave Bowman,dave@example.org),
    Address(123 Some Street,Any Town,USA),
    1
  )
)
~~~

You can continue to select just some fields. For example `users.map(_.email).run` will produce:

~~~ scala
Vector(dave@example.org)
~~~

However, notice that if you used `users.ddl.create`, only the columns defined in the default projection were created in the H2 database.

</div>



## Table Representation


First let's look at how this class relates to the database table.
The name of the table is given as a parameter,
in this case the `String` `user` --- `Table[User](tag, "user")`.
An optional schema name can also be provided, if required by your database.


For the rest of the chapter we'll look at some more in depth areas of data modelling.

### Nullable Columns

Thus far we have only looked at non null columns,
however sometimes we will wish to modal optional data.
Slick handles this in an idiomatic scala fashion using `Option[T]`.

Let's expand our data model to allow direct messaging,
by adding the ability to define a recipient on `Message`.
We'll label the column `to`:

~~~ scala

 final case class Message(sender: Long,
 content: String,
 ts: DateTime,
 to: Option[Long] = None,
 id: Long = 0L)

 final class MessageTable(tag: Tag) extends Table[Message](tag, "message") {

 def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def senderId = column[Long]("sender")
 def toId = column[Option[Long]]("to")
 def content = column[String]("content")
 def ts = column[Timestamp]("ts")

 def * = (senderId, content, ts, toId, id) <> (Message.tupled, Message.unapply)

 }

~~~

<div class="callout callout-danger">
**Equality**
We can not compare these columns as `Option[T]` in queries.

Consider the snippet below,
what do you expect the two results to be?

~~~ scala

val:Option[Long] = None

val a = messages.filter(_.to === to).iterator.foreach(println)
val b = messages.filter(_.to.isEmpty).iterator.foreach(println)

~~~

If you said they would both produce the list of messages,
you'd be wrong.
`a` returns an empty list as `None === None` returns `None`.
We need to use `isEmpty` if we want to filter on null columns.
</div>


**TODO: Understand what ? on a column does and how to use it**

### Primary Keys

There are two methods to declare a column is a primary key.
In the first we declare a column is a primary key using class `O` which provides column options.
We have seen examples of this in `Message` and `User`.

~~~ scala
def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
~~~

The second method uses a method `primaryKey` which takes two parameters ---
a name and a tuple of columns.
This is useful when defining compound primary keys.

By way of a not at all contrived example,
let us add the ability for people to chat in rooms.
I've excluded the room definition,
it is the same as user.

~~~ scala
 final case class Room(name: String, id: Long = 0L)

 final class RoomTable(tag: Tag) extends Table[User](tag, "room") {
 def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def name = column[String]("name")
 def * = (name, id) <> (User.tupled, User.unapply)
 }

 lazy val rooms = TableQuery[RoomTable]

 final case class Occupant(roomId:Long,userId:Long)

 final class OccupantTable(tag: Tag) extends Table[Occupant](tag, "occupant") {
 def roomId = column[Long]("room")
 def userId = column[Long]("user")
 def pk = primaryKey("room_user_pk", (roomId,userId))
 def * = (roomId,userId) <> (Occupant.tupled, Occupant.unapply)
 }

 lazy val occupants = TableQuery[UserTable]
~~~
<div class="callout callout-info">
####TODO Give this a Label
Now we have `room` and `user` the benefit of case classes over tuples becomes apparent.
They both have the same tuple signature `(String,Long)`.
It would get error prone passing around tuples like this.
</div>


The SQL generated for the `occupant` table is:

~~~ sql
create table "occupant" ("room" BIGINT NOT NULL,"user" BIGINT NOT NULL)
alter table "occupant" add constraint "room_user_pk" primary key("room","user")
~~~

### Foreign Keys

Foreign keys are declared in a similar manner to compound primary keys,
with the method --- `foreignKey`.
`foreignKey` takes four required parameters:
 * a name;
 * the column(s) that make the foreignKey;
 * the `TableQuery`that the foreign key belongs to, and
 * a function on the supplied `TableQuery[T]` taking the supplied column(s) as parameters and returning an instance of `T`.

Let's improve our model by using foreign keys for `message`, `sender` and `to` fields:

~~~ scala
 lazy val messages = TableQuery[MessageTable]

 final case class User(name: String, id: Long = 0L)

 final class UserTable(tag: Tag) extends Table[User](tag, "user") {
 def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def name = column[String]("sender")
 def * = (name, id) <> (User.tupled, User.unapply)
 }

 lazy val users = TableQuery[UserTable]

 final case class Message(sender: Long,
 content: String,
 ts: DateTime,
 to: Option[Long] = None,
 id: Long = 0L)

 final class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
 def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
 def senderId = column[Long]("sender")
 def sender = foreignKey("sender_fk", senderId, users)(_.id)
 def toId = column[Option[Long]]("to")
 def to = foreignKey("to_fk", toId, users)(_.id)
 def content = column[String]("content")
 def ts = column[DateTime]("ts")
 def * = (senderId, content, ts, toId, id) <> (Message.tupled, Message.unapply)
 }

 lazy val messages = TableQuery[MessageTable]
~~~


We can see the SQL this produces by running: `dl.createStatements.foreach(println)`.
Which we have included here:
<!-- I've formatted this for readability -->

~~~ sql
CREATE TABLE "message" ("sender" BIGINT NOT NULL,
 "content" VARCHAR NOT NULL,
 "ts" TIMESTAMP NOT NULL,
 "to" BIGINT,
 "id" BIGINT GENERATED BY DEFAULT
 AS IDENTITY(START WITH 1) NOT NULL PRIMARY KEY)

ALTER TABLE "message"
 ADD CONSTRAINT "sender_fk"
 FOREIGN KEY("sender")
 REFERENCES "user"("id") ON UPDATE NO ACTION ON DELETE NO ACTION
alter TABLE "message"
 ADD constraint "to_fk"
 FOREIGN KEY("to")
 REFERENCES "user"("id") ON UPDATE NO ACTION ON DELETE NO ACTION
~~~

<div class="callout callout-info">
####Slick isn't an ORM

Adding foreign keys to our data model does not mean we can traverse from `Message` to `User`, as Slick is not an ORM.

We can however compose our queries and join to return the `User` we are interested in.
The following defines a query which will return the users who sent messages containing `do`:

~~~ scala
 val senders = for {
 message <- messages
 if message.content.toLowerCase like "%do%"
 sender <- message.sender
 } yield sender
~~~

</div>


### Value Classes

Something about case classes are better than tuples.
However,
we are still assigning `Long`s as primary keys,
there is nothing to stop us asking for all messages based on a users id:

~~~ scala
val rubbish = oHAL.map{hal => messages.filter(msg => msg.id === hal.id) }
~~~

This makes no sense, but the compiler can not help us.
Let's see how [value classes][link-scala-value-classes] can help us.
We'll define value classes for `message`, `user` and `room` primary keys.

~~~ scala
 final case class MessagePK(value: Long) extends AnyVal
 final case class UserPK(value: Long) extends AnyVal
 final case class RoomPK(value: Long) extends AnyVal
~~~

For us to be able to use these, we need to define implicits so Slick can convert between the value class and expected type.

~~~ scala
 implicit val messagePKMapper = MappedColumnType.base[MessagePK, Long](_.value, MessagePK(_))
 implicit val userPKMapper = MappedColumnType.base[UserPK, Long](_.value, UserPK(_))
 implicit val roomPKMapper = MappedColumnType.base[RoomPK, Long](_.value, RoomPK(_))
~~~

With our value classes and implicits in place,
we can now use them to give us type checking on our primary and therefore foriegn keys!

~~~ scala
 final case class Message(sender: UserPK,
 content: String,
 ts: DateTime,
 to: Option[UserPK] = None,
 id: MessagePK = MessagePK(0))

 final class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
 def id = column[MessagePK]("id", O.PrimaryKey, O.AutoInc)
 def senderId = column[UserPK]("sender")
 def sender = foreignKey("sender_fk", senderId, users)(_.id)
 def toId = column[Option[UserPK]]("to")
 def to = foreignKey("to_fk", toId, users)(_.id)
 def content = column[String]("content")
 def ts = column[DateTime]("ts")
 def * = (senderId, content, ts, toId, id) <> (Message.tupled, Message.unapply)
 }
~~~

Now, if we try our query again:

~~~ scala
[error] /Users/jonoabroad/developer/books/essential-slick-example/chapter-03/src/main/scala/chapter03/main.scala:129: Cannot perform option-mapped operation
[error] with type: (chapter03.Example.MessagePK, chapter03.Example.UserPK) => R
[error] for base type: (chapter03.Example.MessagePK, chapter03.Example.MessagePK) => Boolean
[error] val rubbish = oHAL.map{hal => messages.filter(msg => msg.id === hal.id) }
[error]   ^
[error] /Users/jonoabroad/developer/books/essential-slick-example/chapter-03/src/main/scala/chapter03/main.scala:129: ambiguous implicit values:
[error] both value BooleanOptionColumnCanBeQueryCondition in object CanBeQueryCondition of type => scala.slick.lifted.CanBeQueryCondition[scala.slick.lifted.Column[Option[Boolean]]]
[error] and value BooleanCanBeQueryCondition in object CanBeQueryCondition of type => scala.slick.lifted.CanBeQueryCondition[Boolean]
[error] match expected type scala.slick.lifted.CanBeQueryCondition[Nothing]
[error] val rubbish = oHAL.map{hal => messages.filter(msg => msg.id === hal.id) }
[error]  ^
[error] two errors found
[error] (compile:compile) Compilation failed
[error] Total time: 2 s, completed 06/02/2015 12:12:53 PM
~~~

The compiler helps,
by telling us we are attempting to compare a `MessagePK` with a `UserPK`.

###Row and column control

We have already seen several examples of these,
including `O.PrimaryKey` and `O.AutoInc`.
Which unsurprislingly declare a column to be a primary key and auto incrementing.
Column options are defined in [ColumnOption][link-slick-column-options],
and as you have seen are accessed via `O`.
We get access to `O` when we import the slick driver,
in our case `import scala.slick.driver.H2Driver.simple._`.

As well as `PrimaryKey` and `AutoInc`,
there is also `Length`, `DBTYPE` and `Default`.

**TODO: add words around examples:**

`Length` takes two parameters:

 * integer - number of unicode characters,
 * boolean - true `VarChar`, false `Char`.

~~~ scala
def name = column[String]("name",O.Length(128,true))
~~~

~~~ sql
-- true
create table "user" ("name" VARCHAR(128) NOT NULL,"id" BIGINT GENERATED BY DEFAULT AS IDENTITY(START WITH 1) NOT NULL PRIMARY KEY)
-- false
create table "user" ("name" CHAR(128) NOT NULL,"id" BIGINT GENERATED BY DEFAULT AS IDENTITY(START WITH 1) NOT NULL PRIMARY KEY)
~~~


`DBType` ...


~~~ scala
def avatar = column[Option[Array[Byte]]]("avatar",O.DBType("Binary(2048)"))
~~~

~~~ sql
create table "user" ("name" VARCHAR DEFAULT '☃' NOT NULL,
 "avatar" Binary(2048),
 "id" BIGINT GENERATED BY DEFAULT AS IDENTITY(START WITH 1)
 NOT NULL PRIMARY KEY)
~~~


`Default` ...

~~~ scala
def name = column[String]("name",O.Default("☃"))
~~~

~~~ sql

create table "user" ("name" VARCHAR DEFAULT '☃' NOT NULL,"id" BIGINT GENERATED BY DEFAULT AS IDENTITY(START WITH 1) NOT NULL PRIMARY KEY)

~~~

<div class="callout callout-info">
#### Notes

##### Nullability

We can also explicitly declare the nullability of a column using `NotNull` and`Nullable`.
Just use `Option[T]` - `NotNull` and `Nullable` are redundant.

##### `Strings`

It is worth noting when defining a `String` column,
if you do not provide either a `Length` or `DBType` column option Slick will default to either `VARCHAR` or `VARCHAR(254)` in the DDL.
</div>

##Custom Column Mapping

We have already seen two examples `DateTime` and value classes as primary keys.
Custom mappings require two things,
definition of the type and an implicit mapping between the type and valid JDBC type.

Let's look at how to use a scala enumeration.
First we define our type:

~~~ scala
object RoomType extends Enumeration {
 type RoomType = Value
 val Private,Public = Value
}

~~~

Next we define an implict mapping between our new type and a valid JDBC type:


~~~ scala
implicit val roomTypeMapper = MappedColumnType.base[RoomType.Value, Int](_.id, RoomType(_))
~~~

Finally, we use our type to define a column:

~~~ scala
def roomType = column[RoomType.Value]("roomType", O.Default(RoomType.Public))
~~~

Here is the SQL the DDL will output:

~~~ sql
create table "room" ("name" VARCHAR NOT NULL,
 "roomType" INTEGER DEFAULT 1 NOT NULL,
 "id" BIGINT GENERATED BY DEFAULT AS IDENTITY(START WITH 1)
 NOT NULL PRIMARY KEY)
~~~

##Virtual columns and server-side casts here?

## Exercises

### Add a message

What happens if you try adding a message with a user id of `3`?
For example:

~~~ scala
messages += Message(3L, "Hello HAl!", new DateTime(2001, 2, 17, 10, 22, 50))
~~~

<div class="solution">

We get a runtime exception as we have violated referential integrity.
There is no row in the `user` table with a primary id of `3`.

~~~ bash

[error] (run-main-12) org.h2.jdbc.JdbcSQLException: Referential integrity constraint violation: "sender_fk: PUBLIC.""message"" FOREIGN KEY(""sender"") REFERENCES PUBLIC.""user""(""id"") (3)"; SQL statement:
[error] insert into "message" ("sender","content","ts","to") values (?,?,?,?) [23506-185]
org.h2.jdbc.JdbcSQLException: Referential integrity constraint violation: "sender_fk: PUBLIC.""message"" FOREIGN KEY(""sender"") REFERENCES PUBLIC.""user""(""id"") (3)"; SQL statement:
insert into "message" ("sender","content","ts","to") values (?,?,?,?) [23506-185]
 at org.h2.message.DbException.getJdbcSQLException(DbException.java:345)
 at org.h2.message.DbException.get(DbException.java:179)
 at org.h2.message.DbException.get(DbException.java:155)
 at org.h2.constraint.ConstraintReferential.checkRowOwnTable(ConstraintReferential.java:372)
 at org.h2.constraint.ConstraintReferential.checkRow(ConstraintReferential.java:314)
 at org.h2.table.Table.fireConstraints(Table.java:920)
 at org.h2.table.Table.fireAfterRow(Table.java:938)
 at org.h2.command.dml.Insert.insertRows(Insert.java:161)
 at org.h2.command.dml.Insert.update(Insert.java:114)
 at org.h2.command.CommandContainer.update(CommandContainer.java:78)
 at org.h2.command.Command.executeUpdate(Command.java:254)
 at org.h2.jdbc.JdbcPreparedStatement.executeUpdateInternal(JdbcPreparedStatement.java:157)
 at org.h2.jdbc.JdbcPreparedStatement.executeUpdate(JdbcPreparedStatement.java:143)
 at scala.slick.driver.JdbcInsertInvokerComponent$BaseInsertInvoker$$anonfun$internalInsert$1.apply(JdbcInsertInvokerComponent.scala:183)
 at scala.slick.driver.JdbcInsertInvokerComponent$BaseInsertInvoker$$anonfun$internalInsert$1.apply(JdbcInsertInvokerComponent.scala:180)
 at scala.slick.jdbc.JdbcBackend$SessionDef$class.withPreparedStatement(JdbcBackend.scala:191)
 at scala.slick.jdbc.JdbcBackend$BaseSession.withPreparedStatement(JdbcBackend.scala:389)
 at scala.slick.driver.JdbcInsertInvokerComponent$BaseInsertInvoker.preparedInsert(JdbcInsertInvokerComponent.scala:170)
 at scala.slick.driver.JdbcInsertInvokerComponent$BaseInsertInvoker.internalInsert(JdbcInsertInvokerComponent.scala:180)
 at scala.slick.driver.JdbcInsertInvokerComponent$BaseInsertInvoker.insert(JdbcInsertInvokerComponent.scala:175)
 at scala.slick.driver.JdbcInsertInvokerComponent$InsertInvokerDef$class.$plus$eq(JdbcInsertInvokerComponent.scala:70)
 at scala.slick.driver.JdbcInsertInvokerComponent$BaseInsertInvoker.$plus$eq(JdbcInsertInvokerComponent.scala:145)
 at chapter03.Example$$anonfun$5.apply(main.scala:84)
 at chapter03.Example$$anonfun$5.apply(main.scala:57)
 at scala.slick.backend.DatabaseComponent$DatabaseDef$class.withSession(DatabaseComponent.scala:34)
 at scala.slick.jdbc.JdbcBackend$DatabaseFactoryDef$$anon$4.withSession(JdbcBackend.scala:61)
 at chapter03.Example$.delayedEndpoint$chapter03$Example$1(main.scala:56)
 at chapter03.Example$delayedInit$body.apply(main.scala:12)
 at scala.Function0$class.apply$mcV$sp(Function0.scala:40)
 at scala.runtime.AbstractFunction0.apply$mcV$sp(AbstractFunction0.scala:12)
 at scala.App$$anonfun$main$1.apply(App.scala:76)
 at scala.App$$anonfun$main$1.apply(App.scala:76)
 at scala.collection.immutable.List.foreach(List.scala:381)
 at scala.collection.generic.TraversableForwarder$class.foreach(TraversableForwarder.scala:35)
 at scala.App$class.main(App.scala:76)
 at chapter03.Example$.main(main.scala:12)
 at chapter03.Example.main(main.scala)
 at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
 at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
 at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
 at java.lang.reflect.Method.invoke(Method.java:606)
~~~
</div>

### ADT

Rewrite our enumeration example of a custom type using an [Algebraic Data Type][link-adt-wikipedia].

~~~ scala
implicit val roomTypeMapper = MappedColumnType.base[RoomType.Value, Int](_.id, RoomType(_))

object RoomType extends Enumeration {
 type RoomType = Value
 val Private,Public = Value
}

def roomType = column[RoomType.Value]("roomType", O.Default(RoomType.Public))
~~~

<div class="solution">

~~~ scala
 sealed trait RoomType { val id:Int }
 case object Private extends RoomType { val id = 0 }
 case object Public extends RoomType { val id = 1 }

 object RoomType {
 def apply(id: Int) = id match {
 case 0 ⇒ Private
 case 1 ⇒ Public
 case _ ⇒ Public
 }
 }
~~~
</div>

### Use foreign keys to find everyone who sent HAL a message

This might be too hard for the moment, as we haven't talked about queries.

<div class="solution">

There isn't one yet.

</div>


