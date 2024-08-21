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





