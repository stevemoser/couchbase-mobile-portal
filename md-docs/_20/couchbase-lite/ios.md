---
id: api-guide
title: API Guide
---

{% include landing.html %}

### What's New?

Most of Couchbase Lite has been rewritten, based on what we've learned from implementing a lightweight cross-platform NoSQL database.

* The core functionality is implemented in a C++ library known as Couchbase Lite Core, or LiteCore. This library is used on all platforms, eliminating a lot of code duplication. The result is more consistent behavior across platforms, and faster development of new features.
* Much higher performance, thanks to LiteCore. Efficient code, and better algorithms and data storage schema, mean that Couchbase Lite 2.0 runs many times faster than 1.x. Performance will vary by platform and by operation, but we've seen large database insertion and query tasks run 5x faster on iOS.
* Queries are now based on expressions, with semantics based on Couchbase Server's N1QL query language. This reduces the learning curve, compared to map/reduce, and makes it easier to create flexible queries. It's also a better fit with platform query APIs like LINQ and NSPredicate. Version 2.0 will not support map/reduce views.
* Full-text search is now available on all platforms.
* The document API has changed significantly, reflecting feedback from developers. Document objects are mutable, so you can update properties incrementally and then save changes. We provide efficient typed accessors so you can get and set numeric/boolean values without the overhead of conversion to objects. In later preview releases we'll add a cross-platform object modeling API (comparable to CBLModel in iOS 1.x) that lets you map documents to native objects.
* Conflict handling is much more direct. You provide conflict-resolver callbacks to control what happens when a save conflicts with a new change, or when the replicator pulls a new revision. Conflicts won't pile up invisibly causing scalability problems. (We will be providing canned conflict resolvers for common algorithms.)
* We've made the API's concurrency rules explicit, and consistent, across all platforms. All Couchbase Lite objects are now bound to the thread they're created on; they can't be called re-entrantly. (This has always been the case in Objective-C.) This makes our code more efficient by avoiding the need for expensive synchronization/locking, and also prevents a lot of tricky concurrency bugs.

#### New in developer build 2:

* A .NET version with a C# API is now available.
* For Mac/iOS/tvOS we've added a native Swift API, which is more idiomatic than the automatic translation of the Objective-C API.
* Nested JSON objects in documents are now represented by `CBLSubdocument` objects instead of the platform's regular dictionary / map type (e.g. NSDictionary.)

### What's Missing?

Some of the new features aren't implemented yet, and some existing features are temporarily missing because they have yet to be adapted to the new core engine and APIs. Pardon our dust! We will be releasing new previews often, so if this one is too incomplete for you to evaluate or use, please check back later.

In developer build 2:

* The replicator is unavailable. It needs to be adapted to the LiteCore APIs.
* The REST API (Listener) is unavailable.
* The database file format has changed, and there is not yet any support for upgrading/migrating 1.x databases. (The format is likely to change again, incompatibly, in future preview releases until we implement migration.)
* The query engine doesn't support joins (querying across multiple documents) yet.
* The query engine is missing a lot of N1QL functions.
* Object modeling (mapping documents to native objects) isn't implemented yet.

## Getting Started

<div class="dp">
	<div class="tiles">
		<div class="column size-1of2">
			<div class="box">
				<div class="container">
					<a href="https://www.couchbase.com/nosql-databases/downloads#couchbase-mobile" taget="_blank">
						<p style="text-align: center;">Download Developer Build</p>
					</a>
				</div>
			</div>
		</div>
		<div class="column size-1of2">
			<div class="box">
				<div class="container">
					<a href="http://cb-mobile.s3.amazonaws.com/api-references/couchbase-lite-2.0DB1/index.html" taget="_blank">
						<p style="text-align: center;">API References</p>
					</a>
				</div>
			</div>
		</div>
	</div>
</div>

### iOS

<img width=600 src="img/download-screenshot.png" style="margin: 0 auto;display: block;" />

Couchbase Lite 2 for iOS is built as a dynamic library, not a static library. (It *looks* the same, a directory named “`CouchbaseLite.framework`”, but the library file inside is different.) This isolates it more from your app's build process, since the linker doesn't have to deal with all the internals of the Couchbase Lite code.

The build process is mostly the same as in 1.x:

- Drag the framework into your Xcode project's left-hand navigator pane (probably into the Frameworks group, but it doesn't really matter).
- Make sure it appears in the “Link Binary With Libraries” group in your target's Build Phases.

Unlike 1.x, you do not need to add any extra libraries like libsqlite3 or libc++.

> Note: If you're replacing CBL 1.x in an app target, remove the old framework from the Xcode project first, then add the new one. If you just overwrite the framework in the filesystem, Xcode may be confused because it still thinks it's linking with a static library, but the library is now dynamic.

<!---
TBD: Add information about stripping the simulator binaries out of the framework during a release build
--->

### macOS

Nothing's changed in the macOS build process. Couchbase Lite 1.x was already a dynamic framework. Just build the same way you would with 1.x.

<!---
### Using CocoaPods or Carthage

Couchbase Lite 2's Xcode project has a simpler build system, which makes it more easily compatible with the package managers CocoaPods and Carthage.

**CocoaPods:** TBD
**Carthage:** Just add this line to your project's Cartfile: `github "couchbase/couchbase-lite-ios" "feature/2.0"`
--->

## The New API

Here are the highlights of the new API. We assume you're already familiar with Couchbase Lite 1.x. This isn't an exhaustive description; please refer to the [Doxygen-generated API docs](http://cb-mobile.s3.amazonaws.com/api-references/couchbase-lite-2.0DB1/index.html), or to the framework's header files, for details.

> Note: The rest of this document uses Objective-C to describe the details of the API. The API concepts (except for platform-specific bindings like NSPredicate) are applicable to all platforms.

### No Manager

You'll notice that CBLManager is gone. Instead, databases are the top-level entities in the API. The main function of CBLManager was to act as a collection of databases, both conceptually and in the filesystem, but it turned out not to be worth the added complexity. Instead, in 2.0 you create and manage databases individually, the way you'd manage files.

## Databases

### Creating Databases

As the top-level entity in the API, CBLDatabase now has initializer methods. You instantiate one by giving a name and options. Just as before, the database will be created in a default location (in Application Support, or in Caches on tvOS). You can override this by specifying a parent directory in the CBLDatabaseOptions.

You can instantiate multiple CBLDatabases with the same name and directory; these will all share the same storage. Do this if you will be calling Couchbase Lite from multiple threads or dispatch queues, since (as in 1.x)  Couchbase Lite objects are not thread-safe and can only be called from one thread/queue. Otherwise, for use on a single thread/queue, it's more efficient to use a single instance.

### Transactions / batch operations

As before, if you're making multiple changes to a database at once, it's *much* faster to group them together. (Otherwise each individual change incurs overhead, from flushing writes to the filesystem to ensure durability.) In 2.0 we've renamed the method, to `-inBatch:do:`, to emphasize that Couchbase Lite does not offer transactional guarantees, and that the purpose of the method is to optimize batch operations rather than to enable ACID transactions.

At the *local* level this operation is still transactional: no other CBLDatabase instances, including ones managed by the replicator or HTTP listener, can make changes during the execution of the block, and other instances will not see partial changes. But Couchbase Mobile is a *distributed* system, and due to the way replication works, there's no guarantee that Sync Gateway or other devices will receive your changes all at once.

Again, the behavior of the method hasn't changed, just its name.

## Documents

CBLDocument has changed a lot. The data model is still the same — a JSON object with a fixed ID string — but the API has absorbed ideas from CBLModel to make it easier to work with.

> Note: CBLModel isn't available yet in this preview release. We are still redesigning its API, and it will be available — on all platforms — in a future preview before 2.0 ships.

### Mutability

The biggest change is that CBLDocument's properties are now mutable. Instead of having to make a mutable copy of the properties dictionary, update it, and then save it back to the document, you can now modify individual properties in place and then save.

This does create the possibility of confusion, since the document's in-memory state may not match what's in the database. Unsaved changes are not visible to other CBLDatabase instances (i.e. other threads), or to queries.

### Typed Accessors

CBLDocument now offers a set of property accessors for various scalar types, including boolean, integers, floating-point and strings. These accessors take care of converting to/from JSON encoding, and make sure you get the type you're expecting: for example, `-stringForKey:` returns either an NSString or `nil`, so you can't get an unexpected object class and crash trying to use it as a string. (Even if the property in the document has an incompatible type, the accessor returns nil.)

> Note: If you're looking for these accessors in the headers, they're not in CBLDocument.h; they're defined in its superclass CBLProperties, so look in CBLProperties.h.

In addition, as a convenience we offer NSDate accessors. Dates are a common data type, but JSON doesn't natively support them, so the convention is to store them as strings in ISO-8601 format. `-dateForKey:` and `setDate:forKey:` do this conversion for you. (So does `-setObjectForKey:` if you pass it an NSDate object. However, `-objectForKey:` does *not*, because it has no way of knowing whether a JSON string should be interpreted as a date. So always call `-dateForKey:` if you want an NSDate.)

### Subdocuments

A *subdocument* is a nested document with its own set of named properties. In JSON terms it's a nested object. This isn't a new feature of the document model; it's just that we're exposing it in a more structured form. In Couchbase Lite 1.x you would see a nested object as a nested NSDictionary. In 2.0 we expose it as a CBLSubdocument object instead.

CBLSubdocument, like CBLDocument, inherits from CBLProperties. That means it has the same set of type-specific accessors discussed in the previous section. Like CBLDocument, it's mutable, so you can make changes in-place. The difference is that a subdocument doesn't have its own ID. It's not a first-class entity in the database, it's just a nested object within the document's JSON. It can't be saved individually; changes are persisted when you save its document.

### Attachments, AKA Blobs

We've renamed "attachments" to "blobs", for clarity. The new behavior should be clearer too: a CBLBlob is now a normal object that can appear in a document as a property value, either at the top level or in a subdocument. In other words, there's no special API for creating or accessing attachments; you just instantiate a CBLBlob and set it as the value of a property, and then later you can get the property value, which will be a CBLBlob object.

CBLBlob itself has a simple API that lets you access the contents as in-memory data (an NSData object) or as a stream (NSInputStream.) It also supports an optional `type` property that by convention stores the MIME type of the contents. Unlike CBLAttachment, blobs don't have names; if you need to associate a name you can put it in another document property, or make the filename be the property name (e.g. `[doc setObject: imageBlob forKey: @"thumbnail.jpg"]`.)

>Note: A blob is stored in the document's raw JSON as an object with a property `"_cbltype":"blob"`. It also has properties such as `"digest"` (a SHA-1 digest of the data), `"length"` (the length in bytes), and optionally `"type"` (the MIME type.) As always, the data is not stored in the document, but in a separate content-addressable store, indexed by the digest.

### Conflict Handling

We're approaching conflict handling differently, and more directly. Instead of requiring application code to go out of its way to find conflicts and look up the revisions involved, Couchbase Lite will detect the conflict (while saving a document, or during replication) and invoke an app-defined conflict-resolver handler. The conflict resolver is given "my" document properties, "their" document properties, and (if available) the properties of the common ancestor revision.

* When saving a CBLDocument, "my" properties will be the in-memory properties of the object, and "their" properties will be one ones already saved in the database (by some other application thread, or by the replicator.)
* During replication, "my" properties will be the ones in the local database, and "their" properties will be the ones coming from the server.

The resolver is responsible for returning the resulting properties that should be saved. There are of course a lot of ways to do this. By the time 2.0 is released we want to include some resolver implementations for common algorithms (like the popular "last writer wins" that just returns "my" properties.) The resolver can also give up by returning nil, in which case the save fails with a "conflict" error. This can be appropriate if the merge needs to be done interactively or by user intervention.

A resolver can be specified either at the database or the document level. If a document doesn't have its own, the database's resolver will be used. If the database doesn't have one either (the default situation), a default algorithm is used that picks the revision with the larger number of changes in its history.

## Queries

Database queries have changed significantly. Instead of the map/reduce algorithm used in 1.x, they're now based on expressions, of the form “*return ____ from documents where ____, ordered by ____*”, with semantics based on Couchbase Server's N1QL query language. If you've used Core Data, or other query APIs based on SQL, you'll find this familiar.

### Query API

The Query API provides a simple way to construct a query statement from a set of API methods. There will be two API styles (builder and chainable) implemented based on what makes sense for each platform.

In the Developer Build 3, a builder API has been implemented into the Objective-C SDK. You can call one of the select methods
in the `CBLQuery` class to build up your query statement.

For example, the `SELECT * FROM type='account' AND owner='Wayne' ORDER BY dealSize` statement can be written with the
builder API as follows:

```Objective-C
CBLQuery *query =
    [CBLQuery select: [CBLQuerySelect all]
                from: [CBLQueryDataSource database: database]
               where: [[CBLQueryExpression property: @"type"] equalTo: @"account"] and:
                      [CBLQueryExpression property: @"owner"] equalTo: @"Wayne"]]
             orderBy: [CBLQueryOrderBy expression: [CBLQueryExpression property: @"dealSize"]]
    ];
```

There are several parts to specifying a query:

1. What document criteria to match (corresponding to the “`WHERE …`” clause in N1QL or SQL)
2. What properties (JSON or derived) of the documents to return (“`SELECT …`”)
3. What criteria to group rows together by (“`GROUP BY …`”)
4. Which grouped rows to include (“`HAVING …`”)
3. The sort order (“`ORDER BY …`”)

These all have defaults:

* If you don't specify criteria, all documents are returned
* If you don't specify properties to return, you just get the document ID and sequence number
* If you don't specify grouping, rows are not grouped
* If you don't specify what groups to include, all are included
* If you don't specify a sort order, the order is undefined

The `CBLQuery` object can be run by calling the `-run:` method which will return an Enumerator of `CBLQueryRow` objects. As of Developer Build 3, column projection is not supported yet but will be supported in a future release.

### NSPredicate API

Same as Developer Build 2, in Objective-C and Swift, we also support NSPredicate query. However in the Developer Build 3, the `NSPredicate` query will be named as `CBLPredicateQuery` and can be constructed from `[CBLDatabase createQueryWhere:]` method. In the future release, we plan to have `NSPredicate` query along side with the query builder style.

Similarly to Core Data, we support the same Core Foundation classes:

1. Document criteria are expressed as an NSPredicate
2. The sort order is an array of NSSortDescriptors
3. The properties to return is an array of NSExpressions.

> Note: If you're not familiar with these Foundation classes, you'll need to read Apple's documentation first.

For convenience, you can provide these as NSStrings: document criteria will be interpreted as NSPredicate format strings, properties to return as NSExpression format strings, and sort orders as key-paths (optionally prefixed with “-” to indicate descending order.)

A CBLPredicateQuery object can be created by calling -createQueryWhere: method on CBLDatabase. After creating a query you can set additional attributes like grouping and ordering before running it.

#### Parameters

A query can have placeholder parameters that are filled in when it's run. This makes the query more flexible, and it improves performance since the query only has to be compiled once (see below.)

Parameters are specified in the usual way when constructing the NSPredicate. In the string-based syntax they're written as “`$`”-prefixed identifiers, like “`$MinPrice`”. (The “`$`” is not considered part of the parameter name.) If constructing the predicate as an object tree, you call `+[NSExpression expressionForVariable:]`.

The compiled CBLPredicateQuery has a property `parameters` , an NSDictionary that maps parameter names (minus the “`$`”!) to values. The values need to be JSON-compatible types. All parameters specified in the query need to be given values via the `parameters` property before running the query, otherwise you'll get an error.

#### Return Values

As in 1.x, running a CBLQuery returns an enumeration of CBLQueryRow objects. Each row's `documentID` property gives the ID of the associated document, and its `document` property loads the document object (at the cost of an extra database lookup.) But a query row can also return values directly, which is often faster than having to load the whole document.

To return values directly from query rows, set the query object's `returning:` property to an array of NSExpressions (or strings that parse to NSExpressions.) It's common to use key-paths, to return document properties directly, but you can add logic or computation.

To access the values returned by a CBLQueryRow, call any of the methods `-objectAtIndex:`, `integerAtIndex:`, etc., where the index corresponds to the index in the query's `returning:` array. Use the most appropriate method for the type of value returned; the numeric/boolean accessors are more efficient, as well as more convenient, because they avoid allocating NSNumber objects. `-stringAtIndex:` will return nil if the value is not a string (avoiding the possibility of an exception), and `-dateAtIndex:` additionally converts an ISO-8601 date string into an NSDate for you.

#### Aggregation and Grouping

If the return values of a query include calls to aggregate functions like `count()`, `min()` or `max()`, all of its rows will be combined together into one, with the aggregate functions operating on their parameters from all the rows.

If you set the query's `groupBy` property, all rows that have the same values of the expressions given in that property will be grouped together. In this case, aggregate functions will operate on the rows in a group, not all the rows of the query.

### Query Performance

Queries have to be parsed and compiled into an optimized form for the underlying database to execute. This doesn't take long, but it's best to create a CBLQuery once and then reuse it, instead of recreating it every time. (Of course, only reuse a CBLQuery on the same thread/queue you created it on!)

Expression-based queries have different performance-vs-flexibility tradeoffs than map/reduce queries. Map functions can be unintuitive to design, and an individual map function isn't very flexible (all you can control is the range of keys.) But any map/reduce query will be fast because, by definition, it's just a single traversal of an index.

On the other hand, expression-based queries are easier to design and more flexible, but there's no guarantee of performance. In fact, by default *all* queries will be unoptimized, because they have to make a linear scan of the entire database, testing every document against the criteria! In a small database you might not notice, but as the database grows, query time will increase linearly. So how do you make a query faster? By creating any necessary indexes.

### Indexing

A query can only be fast if there's a pre-existing database index it can search to narrow down the set of documents to examine. On the other hand, every index has to be updated whenever a document is updated, so too many indexes can hurt performance. Thus, good performance depends on designing and creating the *right* indexes to go along with your queries.

To create an index, call -[CBLDatabase createIndexOn:error:]. (This is a no-op if the index already exists, so it's OK to call it every time the app runs.) The parameter is an array of one or more NSExpressions, or NSStrings that compile to NSExpressions. These are most often key-paths, but they don't have to be. If there are multiple expressions, the first one will be the primary key, the second the secondary key, etc.

### Full-Text Search

Queries can perform a full-text search (FTS), powered by SQLite's FTS4 engine, by using the `MATCHES` operator in an NSPredicate. (This operator is defined as a regular-expression match by Apple, but we've hijacked it for FTS.) The left-hand side is usually a document property, but can be any expression producing a string. The right-hand side is the pattern to match: usually a word or a space-separated list of words, but it can be a more powerful [FTS4 search expression](https://www.sqlite.org/fts3.html#full_text_index_queries).

But hold on! *Before* issuing a query that uses `MATCHES`, you *must* have created a full-text index on the expression being matched. Unlike regular queries, the index is not optional. The index's (single) expression should be the expression you'll use on the left-hand side of the `MATCHES` operator, and its type must be `kCBLFullTextIndex`.

When you run a full-text query, the resulting rows are instances of CBLFullTextQueryRow. This subclass has extra API that lets you access the full string that was matched, and the character range(s) in that string where the match(es) occur.

It's very common to sort full-text results in descending order of relevance. This can be a very difficult heuristic to define, but Couchbase Lite comes with a fairly simple ranking function you can use. In the `orderBy:` array, use a string of the form `rank(X)`, where `X` is the property or expression being searched, to represent the ranking of the result. Since higher rankings are better, you'll probably want to reverse the order by prefixing the string with a `-`.

### Under The Hood

For the time being, the Objective-C NSPredicate query API also allows you to compose queries using the underlying [JSON-based query syntax](https://github.com/couchbase/couchbase-lite-core/wiki/JSON-Query-Schema) recognized by LiteCore. This can be useful as a workaround if you run into limitations or bugs in the NSPredicate/NSExpression-based API. (But if so, please report the issue to us so we can fix it.)

>Disclaimer: This low-level query syntax is not part of Couchbase Lite's public API. We will probably remove this
 access to it before the final release of Couchbase Lite 2.0. By then the public API should be robust enough to handle your needs.

Using the JSON query syntax is very simple: just construct a JSON object tree out of Foundation objects, in accordance with the [spec](https://github.com/couchbase/couchbase-lite-core/wiki/JSON-Query-Schema), then pass the top level NSArray or NSDictionary to the CBLDatabase method that creates a query or creates/deletes an index:

* `-createQueryWhere:` — The `query` parameter can be a JSON NSArray (interpreted as a WHERE clause), or NSDictionary (interpreted as an entire SELECT query.)
* `-createIndexOn:error:` — Any item of the `expressions` array can be a JSON NSArray (interpreted as an expression to index.)
* `-createIndexOn:type:options:error:` — Same as above.
* `-deleteIndexOn:type:error:` — Same as above.

**Troubleshooting:** If LiteCore doesn't like your JSON, the call will return with an error. More usefully, LiteCore will log an error message to the console, so check that. (For internal reasons these messages don't propagate all the way up to the NSError yet.) If you're still stuck, it may help to set an Xcode breakpoint on all C++ exceptions; this will get hit when the parser gives up, and the stack backtrace _might_ give a clue. A common mistake is to pass an expression where an _array of_ expressions is expected; this is easy to do since expressions themselves are arrays. For example, `returning: @[@".", @"x"]` won't work; instead use `returning: @[@[@".", @"x"]]`.