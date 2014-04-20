rxjava-jdbc
============

Efficient execution, concise code, and functional composition of database calls
using JDBC and [RxJava](https://github.com/Netflix/RxJava/wiki) [Observable](http://netflix.github.io/RxJava/javadoc/rx/Observable.html). 

Status: Released to Maven Central

[Release Notes](RELEASE_NOTES.md)

Features
--------------------------
* Functionally compose database queries run sequentially or in parallel 
* Queries may be only partially run or indeed never run due to subscription cancellations thus improving efficiency
* Concise code
* Queries can depend on completion of other Observables and can be supplied parameters through Observables.
* Method chaining just leads the way (once you are on top of the RxJava api of course!)
* All the RxJava goodness!
* Automatically maps query result rows into typed tuples or your own classes
* CLOB and BLOB handling is simplified greatly

Continuous integration with Jenkins for this project is [here](https://xuml-tools.ci.cloudbees.com/). <a href="https://xuml-tools.ci.cloudbees.com/"><img  src="http://web-static-cloudfront.s3.amazonaws.com/images/badges/BuiltOnDEV.png"/></a>

Maven site reports are [here](http://davidmoten.github.io/rxjava-jdbc/index.html) including [javadoc](http://davidmoten.github.io/rxjava-jdbc/apidocs/index.html).

Todo
------------
* Callable statements

Build instructions
-------------------

```
git clone https://github.com/davidmoten/rxjava-jdbc.git
cd rxjava-jdbc
mvn clean install
```

Getting started
--------------------
Include this maven dependency in your pom (available in Maven Central):
```xml
<dependency>
    <groupId>com.github.davidmoten</groupId>
    <artifactId>rxjava-jdbc</artifactId>
    <version>0.1.2</version>
</dependency>
```

After using [RxJava](https://github.com/Netflix/RxJava/wiki) on a work project and being very impressed with 
it (even without Java 8 lambdas!), I wondered what it could offer for JDBC usage. The answer is lots!

Here's a simple example:
```java
Database db = Database.from(url);
List<String> names = db.
		.select("select name from person where name > ? order by name")
		.parameter("ALEX")
		.getAs(String.class)
		.toList().toBlockingObservable().single();
System.out.println(names);
```
output:
```
[FRED, JOSEPH, MARMADUKE]
```
Without using rxjava-jdbc the code is ugly mainly because of the pain of closing jdbc resources:
```java
Connection con = null;
PreparedStatement ps = null;
ResultSet rs = null;
try {
    con = DriverManager.getConnection(url);
	ps = con.prepareStatement("select name from person where name > ? order by name");
	ps.setObject(1, "ALEX");
	rs = ps.executeQuery();
	List<String> list = new ArrayList<String>();
	while (rs.next()) {
		list.add(rs.getString(1));
	}
	System.out.println(list);
} catch (SQLException e) {
	throw new RuntimeException(e);
} finally {
	if (rs != null)
		try {
			rs.close();
		} catch (SQLException e) {
		}
	if (ps != null)
		try {
			ps.close();
		} catch (SQLException e) {
		}
	if (con!=null) {
		try {
			con.close();
		} catch (SQLException e) {
		}
	}
}
```

Query types
------------------
The ```Database.select()``` method is used for 
* SQL select queries. 

The ```Database.update()``` method is used for
* update
* insert
* delete
* DDL (like *create table*, etc)

Examples of all of the above methods are found in the sections below. 

Functional composition of JDBC calls
-----------------------------------------
Here's an example, wonderfully brief compared to normal JDBC usage:
 
```java
import com.github.davidmoten.rx.jdbc.Database;
import rx.Observable;

// use composition to find the first person alphabetically with
// a score less than the person with the last name alphabetically
// whose name is not XAVIER. Two threads and connections will be used.

Database db = new Database(connectionProvider);
Observable<Integer> score = db
		.select("select score from person where name <> ? order by name")
		.parameter("XAVIER")
		.getAs(Integer.class)
		.last();
String name = db
		.query("select name from person where score < ? order by name")
		.parameters(score)
		.getAs(String.class)
		.first()
		.toBlockingObservable().single();
assertEquals("FRED", name);
```
or alternatively using the ```Observable.lift()``` method to chain everything in one command:
```java
String name = db
    .select("select score from person where name <> ? order by name")
    .parameter("XAVIER")
    .getAs(Integer.class)
    .last()
    .lift(db.select("select name from person where score < ? order by name")
            .parameterOperator()
            .getAs(String.class))
    .first()
    .toBlockingObservable().single();
```

About BlockingObservable
----------------------------
You'll see ```toBlockingObservable()``` used in the examples in this page and in 
the unit tests but in your application code you should try to avoid using it. The most benefit 
from the reactive style is obtained by *not leaving the monad*. That is, stay in Observable land and make 
the most of it. Chain everything together and leave toBlockingObservable to 
an endpoint or better still just subscribe with an ```Observer```.

Dependencies
--------------
You can setup chains of dependencies that will determine the order of running of queries. 

To indicate that a query cannot be run before one or more other Observables
have been completed use the `dependsOn()` method. Here's an example:
```java
Observable<Integer> insert = db
		.update("insert into person(name,score) values(?,?)")
		.parameters("JOHN", 45)
		.count()
		.map(Util.<Integer> delay(500));
int count = db
		.select("select name from person")
		.dependsOn(insert)
		.get()
		.count()
		.toBlockingObservable().single();
assertEquals(4, count);
```

Mixing explicit and Observable parameters
------------------------------------------
Example:
```java
String name= db
	.query("select name from person where name > ?  and score < ? order by name")
	.parameter("BARRY")
	.parameters(Observable.from(100))
	.getAs(String.class)
	.first()
	.toBlockingObservable().single();
assertEquals("FRED",name);
```

Passing multiple parameter sets to a query
--------------------------------------------
Given a sequence of parameters, each chunk of parameters will be run with the query and the results appended. 
In the example below there is only one parameter in the sql statement yet two parameters are specified.
This causes the statement to be run twice.

```java
List<Integer> list = 
	db.query("select score from person where name=?")
	    .parameter("FRED").parameter("JOSEPH")
		.getAs(Integer.class).toList().toBlockingObservable().single();
assertEquals(Arrays.asList(21,34),list);
```

Processing a ResultSet
-----------------------------
Many operators in rxjava process items pushed to them asynchronously. Given this it is important that ```ResultSet``` query results are processed 
before being emitted to a consuming operator. This means that the select query needs to be passed a function that converts a ```ResultSet``` to
a result that does not depend on an open ```java.sql.Connection```. Use the ```get()```, ```getAs()```, ```getTuple?()```, and ```autoMap()``` methods to process the method
to specify this function as below.

```java
Observable<Integer> scores = db.query("select score from person where name=?")
	    .parameter("FRED")
		.getAs(Integer.class);
```


Automap
------------------------------
Given this class:
```java
static class Person {
		private final String name;
		private final double score;
		private final Long dateOfBirth;
		private final Long registered;

		Person(String name, Double score, Long dateOfBirth,
				Long registered) {
				...
```
We can get *rxjava-jdbc* to use reflection to auto map the fields in a result set to create an instance of ```Person```:
```java
Observable<Person> persons = db
				.select("select name,score,dob,registered from person order by name")
				.autoMap(Person.class);
```
The main requirement is that the number of columns in the select statement must match 
the number of columns in a constructor of ```Person``` and that the column types can be 
automatically mapped to the types in the constructor.

Auto mappings
------------------
The automatic mappings below of objects are used in the ```autoMap()``` method and for typed ```getAs()``` calls.
* ```java.sql.Date```,```java.sql.Time```,```java.sql.Timestamp``` <==> ```java.util.Date```
* ```java.sql.Date```,```java.sql.Time```,```java.sql.Timestamp```  ==> ```java.lang.Long```
* ```java.sql.Blob``` <==> ```java.io.InputStream```, ```byte[]```
* ```java.sql.Clob``` <==> ```java.io.Reader```, ```String```
* ```java.math.BigInteger``` ==> ```Long```, ```Integer```, ```Decimal```, ```Float```, ```Short```, ```java.math.BigDecimal```
* ```java.math.BigDecimal``` ==> ```Long```, ```Integer```, ```Decimal```, ```Float```, ```Short```, ```java.math.BigInteger```

Note that automappings do not occur to primitives so use ```Long``` instead of ```long```.

Tuples
---------------
Typed tuples can be returned in an ```Observable```:
###Tuple2
```java
Tuple2<String, Integer> tuple = db
		.query("select name,score from person where name >? order by name")
		.parameter("ALEX").create()
		.execute(String.class, Integer.class).last()
		.toBlockingObservable().single();
assertEquals("MARMADUKE", tuple.value1());
assertEquals(25, (int) tuple.value2());
```
Similarly for ```Tuple3```, ```Tuple4```, ```Tuple5```, ```Tuple6```, ```Tuple7```, and finally 
###TupleN
```java
TupleN<String> tuple = db
		.query("select name, lower(name) from person order by name")
		.create()
		.executeN(String.class).first()
		.toBlockingObservable().single();
assertEquals("FRED", tuple.values().get(0));
assertEquals("fred", tuple.values().get(1));
```

Large objects support
------------------------------
Blob and Clobs are straightforward to handle.

### Insert a Clob
Here's how to insert a String value into a Clob (*document* column below is of type ```CLOB```):
```java
String document = ...
Observable<Integer> count = db
		.update("insert into person_clob(name,document) values(?,?)")
		.parameter("FRED")
		.parameter(document).count();
```
or using a ```java.io.Reader```:
```java
Reader reader = ...;
Observable<Integer> count = db
		.update("insert into person_clob(name,document) values(?,?)")
		.parameter("FRED")
		.parameter(reader).count();
```
### Read a Clob
```java
Observable<String> document = db.select("select document from person_clob")
				.getAs(String.class);
```
or
```java
Observable<Reader> document = db.select("select document from person_clob")
				.getAs(Reader.class);
```
### Insert a Blob
Similarly for Blobs (*document* column below is of type ```BLOB```):
```java
byte[] bytes = ...
Observable<Integer> count = db
		.update("insert into person_blob(name,document) values(?,?)")
		.parameter("FRED")
		.parameter(bytes).count();
```
### Read a Blob
```java
Observable<byte[]> document = db.select("select document from person_clob")
				.getAs(byte[].class);
```
or
```java
Observable<InputStream> document = db.select("select document from person_clob")
				.getAs(InputStream.class);
```

Lift
-----------------------------------

Using the ```Observable.lift()``` method you can perform multiple queries without breaking method chaining. ```Observable.lift()``` 
requires an ```Operator``` parameter which are available via 

* ```db.select(sql).parameterOperator().getXXX()```
* ```db.select(sql).parameterListOperator().getXXX()```
* ```db.select(sql).dependsOnOperator().getXXX()```
* ```db.update(sql).parameterOperator()```
* ```db.update(sql).parameterListOperator()```
* ```db.update(sql).dependsOnOperator()```

Example:   
```java
Observable<Integer> score = Observable
    // parameters for coming update
    .from(Arrays.<Object> asList(4, "FRED"))
    // update Fred's score to 4
    .lift(db.update("update person set score=? where name=?")
            //parameters are pushed
            .parameterOperator())
    // update everyone with score of 4 to 14
    .lift(db.update("update person set score=? where score=?")
            .parameters(14, 4)
            //wait for completion of previous observable
            .dependsOnOperator())
    // get Fred's score
    .lift(db.select("select score from person where name=?")
            .parameters("FRED")
            //wait for completion of previous observable
            .dependsOnOperator()
			.getAs(Integer.class));
```

Note that conditional evaluation of a query is obtained using 
the ```parameterOperator()``` method (no parameters means no query run) 
whereas using ```dependsOnOperator()``` just waits for the 
dependency to complete and ignores how many items the dependency emits.  

If the query does not require parameters you can push it an empty list 
and use the ```parameterListOperator()``` to force execution.

Example:
```java
Observable<Integer> rowsAffected = Observable
    //generate two integers
    .range(1,2)
    //replace the integers with empty observables
    .map(toEmpty())
    //execute the update twice with an empty list
    .lift(db.update("update person set score = score + 1")
            .parameterListOperator())
    // flatten
    .lift(RxUtil.<Integer> flatten())
    // total the affected records
    .lift(SUM_INTEGER);
```

Transactions
------------------
When you want a statement to participate in a transaction then either it should
* depend on ```db.beginTransaction()``` 
* be passed parameters or dependencies through ```db.beginTransactionOnNext()```

###Transactions as dependency
```java
Observable<Boolean> begin = db.beginTransaction();
Observable<Integer> updateCount = db
    // set everyones score to 99
    .update("update person set score=?")
    // is within transaction
    .dependsOn(begin)
    // new score
    .parameter(99)
    // execute
    .count();
Observable<Boolean> commit = db.commit(updateCount);
long count = db
    .select("select count(*) from person where score=?")
	// set score
	.parameter(99)
	// depends on
	.dependsOn(commit)
	// return as Long
	.getAs(Long.class)
	// log
	.doOnEach(RxUtil.log())
	// get answer
	.toBlockingObservable().single();
assertEquals(3, count);
```

###onNext Transactions
```java
List<Integer> mins = Observable
    // do 3 times
    .from(asList(11, 12, 13))
    // begin transaction for each item
    .lift(db.beginTransactionOnNextOperator())
    // update all scores to the item
    .lift(db.update("update person set score=?").parameterOperator())
    // to empty parameter list
    .map(toEmpty())
    // increase score
    .lift(db.update("update person set score=score + 5").parameterListOperator())
    //only expect one result so can flatten
    .lift(RxUtil.<Integer>flatten())
    // commit transaction
    .lift(db.commitOnNextOperator())
    // to empty lists
    .map(toEmpty())
    // return count
    .lift(db.select("select min(score) from person").parameterListOperator()
            .getAs(Integer.class))
    // list the results
    .toList()
    // block and get
    .toBlockingObservable().single();
assertEquals(Arrays.asList(16, 17, 18), mins);
```

Asynchronous queries
--------------------------
Unless run within a transaction all queries are synchronous by default. However, if you request an asynchronous 
version of the database using ```Database.asynchronous()``` or if you use asynchronous operators then watch out because this means that 
something like the code below could produce unpredictable results:

```java
Database adb = db.asynchronous();
Observable
    .from(asList(1,2,3))
    .lift(adb.update("update person set score = ?")
            .parameterOperator());
```
After running this code you have no guarantee that the *update person set score=1* ran before the *update person set score=2*. 
To run those queries synchronously either use a transaction:

```java
Database adb = db.asynchronous();
Observable
   .from(asList(1, 2, 3))
   .lift(adb.update("update person set score = ?")
           .dependsOn(db.beginTransaction())
           .parameterOperator())
    .lift(adb.commitOnCompleteOperator());
```

or use the default version of the ```Database``` object that schedules queries using ```Schedulers.trampoline()```.

```java
Observable.from(asList(1,2,3))
          .lift(db.update("update person set score = ?")
                  .parameterOperator());
```

Logging
-----------------
Logging is handled by slf4j which bridges to the logging framework of your choice. Add
the dependency for your logging framework as a maven dependency and you are sorted. See the test scoped log4j example in [rxjava-jdbc/pom.xml](https://github.com/davidmoten/rxjava-jdbc/blob/master/pom.xml).

Database Connection Pools
----------------------------
Include the dependency below:
```xml
<dependency>
	<groupId>com.mchange</groupId>
	<artifactId>c3p0</artifactId>
	<version>0.9.5-pre8</version>
</dependency>
```
and you can use a c3p0 database connection pool like so:
```java
Database db = Database.builder().url(url).pool(minPoolSize,maxPoolSize).build();
```
Once finished with a ``Database`` that has used a connection pool you should call 
```java
db.close();
```
This will close the connection pool and  release its resources.

Note: do not use a c3p0 version earlier than the one above as a c3p0 bug may prevent proper closure of connections.

Use a single Connection
---------------------------
A ```Database``` can be instantiated from a single ```java.sql.Connection``` which will 
be used for all queries in companion with the current thread ```Scheduler``` (```Schedulers.trampoline()```).
The connection is wrapped in a ```ConnectionNonClosing``` which suppresses close calls so that the connection will
 still be open for all queries and will remain open after use of the ```Database``` object.
 
 Example:
 ```java
 Database db = Database.from(con);
 ```
