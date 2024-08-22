# High Performance SQL Lite Course
[Course Link](https://highperformancesqlite.com/)

## Introduction
* Traditional DBs have a client-server model
    * Files are all over, not supposed to mess with those typically
* SQLite does NOT - it's a C library to talks to the file on the disk
    * Far less configuration
    * Database is a single file
        * Sometimes a couple in certain scenarios
* Was written in 2000 - stability matters
* Can use SQLite can be used in place of Postgres/MySQL, but not in ALL cases
    * Maintainers say they don't compete with them, but that's arguable

* SQLite is safe, and has strong guarantees that databases provide (ex: ACID Transactions)
* Open Source, but not Open Contribution - kind of weird
    * [libsql](https://github.com/tursodatabase/libsql) - is open contribution fork of SQL

* Strictly defined file format. Great backwards/forwards compatibility
    * Makes it easy to distribute
    * No overhead to creating additional databases.
    * Great for testing

### Good Use Cases
1. Embedded apps
    * Arduino, Raspberry Pi, Phone
    * Out on the "edge"
2. Web apps
    * Constrained to 1 machine, and they can get big.
    * Thousands of requests per second
    * Many use cases fit this, no network roundtrip overhead
3. Portable
    * Shipping data around
4. File Formats
    * Can be named anything, and desktop applications can use it as the file format
    * Better than having many little files

* Reading from SQLite can be 35x faster than going straight to the disk
    * Open it once, make many reads/etc. before closing it
* SQLite has low operational overhead

### Limitations
1. Concurrent writers
    * Massive number of writers, better to use a client-server database
    * Many readers are *NOT* a problem
    * Thousands of writes per second... that's when you want to move off
    * WAL mode makes writer contetion *less* of an issue.
2. Shared database
    * Multiple machines that need to talk to the same database
    * Need to trust file system locks
    * Turso could have a solution that replicates read copies on machines.
3. Large Sizes
    * 1 TB -> start to think about putting this somewhere else.
4. Fine Grained Privileges
    * Not great for access control

## SQLite's Internals

### SQLite's Structure
* SQL Command Processer
    * Takes your SQL query, and turns it into byte code (tokenizer, parser, code generator / query planner)
    * Virtual machine (just a c file) takes that bytecode and executes the instructions
    * Query Planner - necessary because SQL is declarative. It's the database's job to figure out *how* to get what you're looking for and plan the most efficient/best way to do it.

* All SQLite data stored as a B-Tree
* B-Tree module figures out which data it needs, and communicates with the Pager
* Pager module then pulls data out in fixed size chunk called "pages" (off disk back to b-tree)
* Pager has to handle the messy details:
    * Atomic commits
    * Rollbacks
    * Locking
* OS Interface responible for talking with the actual OS

* B-trees and Pages are *most* important to understand, and are general concepts that apply to *all* databases

### SQLite's File Format
* The file is a lot of equal size chunks of data.
* When you delete it. It deletes, the page where they are
* Chunk size is configurable- and configuration itself is a page
* First line is ASCII of "SQLite format 3"
* Settings like page size/journal mode, etc. are all at the front of the file to inform sqlite how to operate

### Dot commands
* .help
* .exit
* .shell clear
    * runs shell commands
* .headers on
    * Gives headers to first row (col names)
* .mode
    * many different formats (ex: list, box, csv, column, ...)
    * only remains for this session
    * insert -> creates an INSERT INTO statement for the results
* .output <somefile.sql>
    * outputs the command to <somefile.sql> or whatever you called that file
* .once <somefile.sql>
    * will output that just once, then back to STD out
* .tables
    * lists tables
* .schema
    * Produces SQL commands that create the structure of the db (DDL)
* .expert
    * Index suggester / expert mode

### Pragma statements
* Pragma statements modify the behavior of the SQLite db
* page_count
    * Number of pages in database
* page_size
    * 4096 bytes
* busy_timeout
    * 0 -> 5000 is 5 seconds
    * Reset on connection only, but some are persistent
* journal_mode
    * persistent configuration at the database level
    * delete is the default
    * WAL mode = persistent across all connections 
* compile_options
* .dbconfig
* foreign_keys
    * session level setting - enable foreign_keys every time you open the connection

### Virtual Tables
* Allows you to query a table that is not actually written to your database
* Example:  Reading a CSV file but operating on it in SQLite
* Useful to generate data

## Schema

### Flexible Types

* Can create a table without any types whatsoever
* Even when you declare a type, when you insert something of another type, it will just allow it by default
* `select typeof(a) from ex` shows that data type is applied at the cell level, and not at the column level
* **type affinity** - When a value is inserted in to a typed column, if the value being inserted does not match the type but can be converted, it will do the converstion (ex: string '3' is inserted to an integer column, and becomes an int when written)

### Types
* SQLite does have types:
    * types declared at column level are suggestions
    * actual type is at the data level
    * **strict tables** are different, column declarations are strict unless you use the "any" type.

* 5 types
    1. null
    2. integer
    3. real
    4. text
    5. blob

* **Type Affinity** - how it determines if it should take your input and turn it into a different type. Converts if it doesn't lose any data.

* If it will lose data, it will store a different type.
    * Ex: inserting 1.1 into an int column will store that value as a real

* "INT" -> Integer
* "CHAR", "CLOB", "TEXT" -> Text
* "BLOB", "not specified" -> Blob
* "REAL", "FLOA", "DOUB" -> Real
* Otherwise -> Numeric

* Varchar limits aren't respected

### Strict Types
* **Strict Tables** are created by just addding a `strict` table modifier
    * ex: `create tables strict_types (int integer, text text) strict;`

* Can mix and match strictness and permissive types
    * Good use case of this would be a key-value store
    * Declare table  as strict, and use an "any" type

### Dates

* Three ways to store dates:
    1. Text (ISO 8601) 
        * date = YYYY-MM-DD
        * timestamp = YYY-MM-DD H:i:s
    2. Real (Julian Day - JDN)
    3. Integer (Unix Timestamp)
        * unixepoch('subsec") - subsec gives you fraction

* Time functions:
    * strftime('%d', 'now') = day of the month of 'now'
    * timediff(date(), date())
    * datetime('now', 'start of month', '+1 month', '-1 day')
    * datetime(12309234092, 'unixepoch')

### Booleans

* Store as an int - 1 = true, - = false

### Floating Point

* No difference between float vs. decimal. Only Floating point with the `Real` type
* Working around floating points:
    * decimal_sum(colname) workaround
    * This is an extension you would need to load
* `Real` guarantees 15 decimal places of precision
* If you need precision like price for instance:
    * Stripe does this: Converting value into into an integer with the lowest denominoation.
    * ex: $99.99 is stored as 99.99 * 100 -> 9999.0
    * Then you just need to access the value by dividing by 100
 
### Rowid tables

* Rowid = "secret" primary key for your row on your table
* `rowid`, `_rowid_`, `oid`
* Mainly this is a vestigial/historical artifact for backwards compatibilty
* Tradiational databases have you declare a primary key, and that's how it keep track of you data on the disk under the hood
    * ex: auto-incrementing bigint will all be written next to eachother on disk (physically close)
    * That is called the "clustered index" -> defines how data is stored on disk
* In SQLite, the rowid is the clustered index
* Even if you create the primary key yourself, it still uses rowid under the hood
    * Can have a null primary key beacuse rowid is the real primary key and not primary key
* If you create a primary key that is an int, and start at 1, and it matches the rowid, it will alias instead of being a secondary ID
    * This will prevent a 2 index hop that happens when you don't have the rowid matching the primary key
* It's a good idea to leverage that and alias the id
    * If you don't create a column that is an alias of the rowid, then rowid is subject to change
    * When you create that alias of rowid, it becomes stable
* rowid selection algorithm will reuse rowids, so it is by default not guaranteed to be ever incrementing/increasing by the next number
    * Auto increments changes the rowid selection algorithm

### Auto Increments

* `create table auto (id integer primary key autoincrement, text text);`
* If you add the biggest int id possible, it will say the database or disk is full (it won't try and fill any space, etc.)
* Even after deletion, it won't fix itself because the sequence is still there
* `select * from sqlite_sequence` will pull the table and the last sequence number
* Algorithm with auto increment is slightly more CPU intensive, thus less performant, but feels safer

### Without rowid

* "without rowid" table modifier
* Must declare its own primary key, and cannot use auto-increment with it
* This is a performance optimization, and is arranged on disk by your primary key -> single index lookup vs. multiple
* Use it if you have a string or compound primary key

### Generated Columns

* Traditionally, you create a column and you put data in it.
* With SQLite, you create a column and have SQLite put data in it.
* Ex: `create table gen (id integer primary key, first_name text, last_name text, full_name generated always as (concat(first_name, ' ', last_name)))`
* Cannot control or change that column, because SQLite puts the data in it
* Virtual Generated column -> calculated at runtime (This is the default)
    * Little bit more expensive in CPU in overhead.
* Stored Generated column -> calculated once then written to the disk.
    * Takes more space storage-wise, but saves the CPU overhead
* Can put an index on both
* Also, don't need "generated always": `create table gen (id integer primary key, first_name text, last_name text, full_name generated always as (concat(first_name, ' ', last_name)) stored)`
* Function needs to be deterministic (can't be random, etc.)
* Great use cases:
    * Extracting JSON automatically
    * Breaking up email address parts
    * Math
