# **Chapter 3 - Storage and Retrieval**

**Outline** 
* ## Data Structures That Power Your Database
  * ### Hash Indexes 
  * ### SSTables and LSM-Trees 
    * #### Constructing and maintaining SSTables 
    * #### Making an LSM-tree out of SSTables 
    * #### Performance optimizations 
  * ### B-Trees 
    * #### Making B-trees reliable 
    * #### B-tree optimizations 
  * ### Comparing B-Trees and LSM-Trees 
    * #### Advantages of LSM-trees 
    * #### Downsides of LSM-trees
  * ### Other Indexing Structures 
    * #### Storing values within the index
    * #### Multi-column indexes
    * #### Full-text search and fuzzy indexes 
    * #### Keeping everything in memory 
* ## Transaction Processing or Analytics? 
  * ### Data Warehousing 
    * #### The divergence between OLTP databases and data warehouses 
  * ### Stars and Snowflakes: Schemas for Analytics 
* ## Column-Oriented Storage
  * ### Column Compression 
    * #### Memory bandwidth and vectorized processing 
  * ## Sort Order in Column Storage 
    * **Several different sort orders** 
  * ## Writing to Column-Oriented Storage
  * ## Aggregation: Data Cubes and Materialized Views 
* ## Summary 
- - -
### NOTES
## Data Structures That Power Your Databases 

* An implementation of a db using two Base functions: 

*#!/bin/bash* 
```
    db_set () {
        echo "$1,$2" >> database
} 
    db_get () {
        grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
} 
```

- These two functions implement a key-store value 
- *db_set key value* will store *key* and *value* in the db
- *db_get key looks up the most recent value associated with a key and returns it
- the storage format is a text file where each line contains a key-value pair, separated by a comma 
- each call *db_set* adds on to the end of the file,
- updating a key multiple times  does not mean the old versions of the value are not overwritten
- you need to look at the last occurrence of a key in a file to find the latest value (hence the *tail -n 1* in *db_get*)

Example for Setting and Getting 
```
$ **db_set 42 '{"name":"San Francisco","attractions":["Golden Gate Bridge"]}'** 
```
```
$ **db_get 42**{"name":"San Francisco","attractions":["Golden Gate Bridge"]} 
```

Example db 
```
$ **cat database**123456,{"name":"London","attractions":["Big Ben","London Eye"]} 42,{"name":"San Francisco","attractions":["Golden Gate Bridge"]} 42,{"name":"San Francisco","attractions":["Exploratorium"]} 
```

* *db_set* has good performance because appending to the end of a file is generally very efficient 
* many databases internally use a *log*, which is an append-only data file [^1]

[^1]: This reminds me of how we utilize docker logs at work. I’ve been trying to improve my debugging skills and referring to logs when something isn’t working is something I’m trying to remember to do. 

* real databases deal with concurrency control, reclaiming disk space so that the log doesn’t grow forever, and handling errors and partially written records
* in this text, *log* is used in the more general sense: an append-only sequence of records that can be human-readable or binary and intended only for other programs to read. 
* *db_get* function has terrible performance if you have a large number of records in your database 
* to look up a key, db_get has to scan the entire database file from beginning to end, looking for occurrences of the key
* in algorithmic terms, the cost of a lookup is *O*(*n*): if you double the number of records *n* in your database, a lookup takes twice as long [^2]

[^2]: What would it look like to optimize this? What is a better run time?

* to efficiently find the value for a particular key in the database, we need a different data structure: an *index* 
* an index is an additional data structure that is derived from primary data 
* a lot of db’s allow you to add and remove indexes, and this doesn’t affect the contents of the database
* keeping track of additional structures incurs overhead, especially on writes
* for writes, it’s hard to beat the performance of simply appending to a file, because that’s the simplest possible write operation 
* any kind of index usually slows down writes, because the index also needs to be updated every time data is written 
* **this is important trade-off in storage systems:** well-chosen indexes speed up read queries, but every index slows down writes 

## Hash Indexes
* key-value stores are similar to *dictionary* types that are common and implemented via hash maps or hash tables 
  [^3]
* *Compaction* means throwing away duplicate keys in the log, and keeping only the most recent update for each key 
* Compaction could be used as a solution to avoid running out of disk space when appending to a file 
* merging and compaction of frozen segments can be done in a background thread, and while it is going on, we can still continue to serve read and write requests as normal, using the old segment files 
* after merging and compaction, the old segment files can be deleted 
* some of the issues that are important in a real implementation are file format, deleting records, crash recovery, partially written records, and [^4]concurrency control 
* the hash table index also has limitations: 
  * the hash table must fit in memory, so if you have a very large number of keys, you’re out of luck 
  * Range queries are not efficient

  [^3]: A hash map is a data structure that stores key-value pairs for efficient retrieval. It uses a hash function to convert keys into indexes, allowing quick access to values. Collisions, where multiple keys map to the same index, are resolved using techniques like chaining or open addressing.

[^4]: did not understand what a ‘writer thread’ is 

## SSTables and LSM-Trees 
* *Sorted String Table*, or *SSTable* requires key-value pairs to be *sorted by key* 
* also requires that each key only appear once within each merged segment file (taken care of by compaction) 
* SSTables advantages over log segments
  * Merging segments is simple and efficient, even if the files are bigger than the available memory 
  * In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory.
  * Since read requests need to scan over several key-value pairs in the requested range anyway, it is possible to group those records into a block and compress it before writing it to disk 

#### Constructing and maintaining SSTables 
* maintaining a sorted structure in memory is easier than on a disk 
* we can maintain as follows
  * when a write comes in, add it to an in-memory balanced tree data structure (for example, a red-black tree). This in-memory tree is sometimes called a *memtable*. 
  * when the memtable gets bigger than some threshold. write it out to disk as an SSTable file. 
  * the new SSTable file becomes the most recent segment of the database.
  * while the SSTable is being written out to disk, writes can continue to a new memtable instance. 
  * in order to serve a read request, first try to find the key in the memtable, then in the most recent on-disk segment, then in the next-older segment, etc. 
  * from time to time, run a merging and compaction process in the background to combine segment files and to discard overwritten or deleted values. 
* works well but if the database crashes, the most recent writes (which are in the memtable but not yet written out to disk) are lost
* to avoid that problem, keep a separate log on disk to which every write is immediately appended
* that log is not in sorted order, but that doesn’t matter, because its only purpose is to restore the memtable after a crash 
* every time the memtable is written out to an SSTable, the corresponding log can be discarded. 

**Making an LSM-tree out of SSTables** 
* a full-text index is more complex than a key-value index but is based on a similar idea: given a word in a search query, find all the documents (web pages, product descriptions, etc.) that mention the word
* this is implemented with a key-value structure where the key is a word (a *term*) and the value is the list of IDs of all the documents that contain the word (the *postings list*)

#### Performance optimizations 
* a lot of detail goes into making a storage engine perform well in practice. 
* example: the LSM-tree algorithm can be slow when looking up keys that do not exist in the database: you have to check the memtable, then the segments all the way back to the oldest (possibly having to read from disk for each one) before you can be sure that the key does not exist.
* to optimize this kind of access, storage engines often use additional *Bloom filters* 
  * A Bloom filter is a memory-efficient data structure for approximating the contents of a set
  * It can tell you if a key does not appear in the database, and thus saves many unnecessary disk reads for nonexistent keys.
* The most common options for determining the order and timing of how SSTable are compacted and merged are *size-tiered* and *leveled* compaction. 
  * in size-tiered compaction, newer and smaller SSTables are successively merged into older and larger SSTables. 
  * in leveled compaction, the key range is split up into smaller SSTables and older data is moved into separate “levels,” which allows the compaction to proceed more incrementally and use less disk space. 
* the basic idea of LSM-trees—keeping a cascade of SSTables that are merged in the background—is simple and effective
* Pros
  * even when the dataset is much bigger than the available memory it continues to work well
  * since data is stored in sorted order, you can efficiently perform range queries (scanning all keys above some minimum and up to some maximum), and because the disk writes are sequential the LSM-tree can support remarkably high write throughput

### B-Trees
* btree is the most widely used indexing structure
* they remain the standard index implementation in almost all relational databases, and many nonrelational databases use them too. 
* b-trees keep key-value pairs sorted by key, which allows efficient key- value lookups and range queries. But that’s where the similarity ends: B-trees have a very different design philosophy. 
* log-structured indexes we saw earlier break the database down into variable-size *segments*, typically several megabytes or more in size, and always write a segment sequentially vs  B-trees break the database down into fixed-size *blocks* or *pages*, traditionally 4 KB in size (sometimes bigger), and read or write one page at a time
* this design corresponds more closely to the underlying hardware, as disks are also arranged in fixed-size blocks
* each page can be identified using an address or location, which allows one page to refer to another—similar to a pointer, but on disk instead of in memory
* one page is designated as the root of a b-tree, where you start any time you want to look up a key in the index
* the root page has several keys and reference to child pages 
* each child is responsible for a continuous range of keys and the keys between references indicate where the boundaries between those ranges lie 
* a leaf page is a page containing individual keys, which either contains the value for each key inline or contains references to the pages where the values can be found
* The number of references to child pages in one page of the B-tree is called the *branching factor* 
* branching factor depends on the amount of space required to store the page references and the range boundaries, but typically is several hundred 
* if you want to update the value for an existing key in a B-tree,
  * you search for the leaf page containing that key
  * change the value in that page,
  * write the page back to disk (any references to that page remain valid)
* if you want to add a new key
  * find the page whose range encompasses the new key 
  * add it to that page
  * if there isn’t enough free space in the page to accommodate the new key, it is split into two half-full pages, and the parent page is updated to account for the new subdivision of key ranges
* this algo ensures that the tree remains *balanced*: a B-tree with *n* keys always has a depth of *O*(log *n*)
* most databases can fit into a B-tree that is three or four levels deep, so you don’t need to follow many page references to find the page you are looking for

### Making B-Trees reliable
* the basic underlying write operation of a B-tree is to overwrite a page on disk with new data 
* it is assumed that the overwrite does not change the location of the page; i.e., all references to that page remain intact when the page is overwritten
* vs log-structured indexes such as LSM-trees, which only append to files (and eventually delete obsolete files) but never modify files in place

### B-tree optimizations 
* copy-on-write scheme - a modified page is written to a different location, and a new version of the parent pages in the tree is created, pointing at the new location 
* save space in pages by not storing the entire key, but abbreviating it 
* additional pointers have been added to the tree
* b-tree variants such as *fractal trees* borrow some log-structured ideas to reduce disk seeks
* 

## Comparing B-Trees and LSM Trees
* rule of thumb
  * LSM-trees are typically faster for writes
  * B-trees are thought to be faster for reads
  * 
| Advantages of LSM Trees                                      | Downsides of LSM Trees                                       |
|--------------------------------------------------------------|--------------------------------------------------------------|
| - LSM-trees sometimes have lower write amplication<br>- LSM-trees can be compressed better, and thus often produce smaller files on disk<br>- | - the compaction process can sometimes interfere with the performance of ongoing reads and writes<br>- at high write throughput: the disk’s finite write bandwidth needs to be shared between the initial write (logging and flushing a memtable to disk) and the compaction threads running in the background<br>- if write throughput is high and compaction is not configured carefully, it can happen that compaction cannot keep up with the rate of incoming write<br> |
* B-trees are ingrained in the architecture of databases and provide consistently good performance for many workloads 
* in new datastores, log-structured indexes are becoming increasingly popular. 

## Other Indexing Structures 
* key-value indexes are like a *primary key* index in the relational model
* a primary key uniquely identifies one row in a relational table, or one document in a document database, or one vertex in a graph database
* other records in the database can refer to that row/document/vertex by its primary key (or ID), and the index is used to resolve such references
* in relational databases, you can create secondary indexes on the same table using the CREATE INDEX command that are often crucial for performing joins efficiently
* a secondary index can easily be constructed from a key-value index 
* the main difference is that keys are not unique; i.e., there might be many rows (documents, vertices) with the same key, which can be solved in two ways
  * either by making each value in the index a list of matching row identifiers (like a postings list in a full-text index) 
  * by making each key unique by appending a row identifier to it
* both B-trees and log-structured indexes can be used as secondary indexes. 

### Storing values within the index 
* the key in an index is the thing that queries search for, but the value can be one of two things
  * it could be the actual row (document, vertex) in question
  * it could be a reference to the row stored elsewhere
* the place where rows are stored is known as a *heap file*, and it stores data in no particular order (it may be append-only, or it may keep track of deleted rows in order to overwrite them with new data later)
* the heap file approach is common because it avoids duplicating data when multiple secondary indexes are present: each index just references a location in the heap file, and the actual data is kept in one place. 
* when updating a value without changing the key, the heap file approach can be efficient: the record can be overwritten in place, provided that the new value is not larger than the old value. 
* a *clustered index*  - when the extra hop from the index to the heap file is too much of a performance penalty for reads so it becomes desirable to store the indexed row directly within an index 
* A compromise between a clustered index (storing all row data within the index) and a nonclustered index (storing only references to the data within the index) is known as a *covering index* or *index with included columns*which stores *some* of a table’s columns within the index 
* this allows some queries to be answered by using the index alone (in which case, the index is said to *cover* the query) 
* clustered and covering indexes can speed up reads, but they require additional storage and can add overhead on writes

### Multi-column indexes 
* *concatenated index* combines several fields into one key by appending one column to another (the index definition specifies in which order the fields are concatenated) 

### Full-text search and fuzzy indexes 
* full-text search engines commonly allow a search for one word to be expanded to include synonyms of the word, to ignore grammatical variations of words, and to search for occurrences of words near each other in the same document, and support various other features that depend on linguistic analysis of the text

### Keeping everything in memory 

* disks are durable 
* disk have a lower cost per gigabyte than RAM
* files on disk can easily be backed up, inspected, and analyzed by external utilities 

## Transaction Processing or Analytics 
* *transaction* - group of reads and writes that form a logical unit
* *transaction processing* just means allowing clients to make low-latency reads and writes 
* *online transaction processing* (OLTP) - an application typically looks up a small number of records by some key, using an index, then records are inserted or updated based on the user’s input
* *online analytic processing* (OLAP) - queries for analytics for business intelligence 

## Data Warehousing 
* A *data warehouse* is a separate database that analysts can query without affecting OLTP operations
* The data warehouse contains a read-only copy of the data in all the various OLTP systems in the company
* Extract–Transform–Load (ETL) -  process where data is extracted from OLTP databases (using either a periodic data dump or a continuous stream of updates) transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse

### The divergence between OLTP databases and data warehouses
* data model of a data warehouse is most commonly relational, because SQL is generally a good fit for analytic queries
* internals of the systems can look  different, because they are optimized for very different query patterns
* Many database vendors now focus on supporting either transaction processing or analytics workloads, but not both 

## Stars and Snowflakes: Schemas for Analytics 
* in analytics, there is much less diversity of data models 
* as each row in the fact table represents an event, the dimensions represent the *who*, *what*, *where*, *when*, *how*, and *why* of the event
* “star schema” comes from the fact that when the table relationships are visualized, the fact table is in the middle, surrounded by its dimension tables; the connections to these tables are like the rays of a star
* the *snowflake schema* is a variation where dimensions are further broken down into subdimensions 

## Column-Oriented Storage
* in most OLTP databases, storage is laid out in a *row-oriented* fashion: all the values from one row of a table are stored next to each other
* document databases are similar: an entire document is typically stored as one contiguous sequence of bytes
* the idea behind *column-oriented storage* is: don’t store all the values from one row together, but store all the values from each *column* together instead 

### Column Compression
* compression techniques
  * bitmap encoding 
  * we can take a column with *n* distinct values and turn it into *n* separate bitmaps: one bitmap for each distinct value, with one bit for each row
  * the bit is 1 if the row has that value, and 0 if not 

### Memory bandwidth and vectorized processing 
* column-oriented storage layouts are also good for making efficient use of CPU cycles 
* example-the query engine can take a chunk of compressed column data that fits comfortably in the CPU’s L1 cache and iterate through it in a tight loop 
* a CPU can execute such a loop much faster than code that requires a lot of function calls and conditions for each record that is processed
* column compression allows more rows from a column to fit in the same amount of L1 cache
* operators, such as the bitwise *AND* and *OR* described previously, can be designed to operate on such chunks of compressed column data directly - a technique is known as *vectorized processing* 
## Sort Order in Column Storage 
* sorted order can help with compression of columns 
### Several Different Sort Orders
* having multiple sort orders in a column-oriented store is a bit similar to having multiple secondary indexes in a row-oriented store
* the difference is that the row- oriented store keeps every row in one place (in the heap file or a clustered index), and secondary indexes just contain pointers to the matching rows
* in a column store, there normally aren’t any pointers to data elsewhere, only columns containing values 
### Writing to Column-Oriented Storage
* column-oriented storage, compression, and sorting all help to make those read queries faster 
* an update-in-place approach, like B-trees use, is not possible with compressed columns
## Aggregation: Data Cubes and Materialized Views
* materialized view- in a relational data model, it is often defined like a standard (virtual) view: a table-like object whose contents are the results of some query
  * the difference is that a materialized view is an actual copy of the query results, written to disk, whereas a virtual view is just a shortcut for writing queries. 
  * when you read from a virtual view, the SQL engine expands it into the view’s underlying query on the fly and then processes the expanded query
* materialized views are expensive because updating 
* a common special case of a materialized view is known as a *data cube* or *OLAP cube*, which is a grid of aggregates grouped by different dimensions
* facts often have more than two dimensions
* advantage of a materialized data cube is that certain queries become very fast because they have effectively been precomputed
* disadvantage is that a data cube doesn’t have the same flexibility as querying the raw data.

## Summary 
* Objective: bottom of how databases handle storage and retrieval
* Questions to consider 
  * What happens when you store data in a database?
  * what does the database do when you query for the data again later? 
* high level - storage engines fall into two broad categories
  * optimized for transaction processing (OLTP)
  * optimized for analytics (OLAP)
  * there are big differences between the access patterns in those use cases: 
    * OLTP systems are typically user-facing, which means that they may see a huge volume of requests
      * in order to handle the load, applications usually only touch a small number of records in each query
      * the application requests records using some kind of key, and the storage engine uses an index to find the data for the requested key
      * disk seek time is often the bottleneck here
    * data warehouses and similar analytic systems are less well known, because they are primarily used by business analysts, not by end users
      * they handle a much lower volume of queries than OLTP systems, but each query is typically very demanding, requiring many millions of records to be scanned in a short time
      * disk bandwidth (not seek time) is often the bottleneck here
      * column- oriented storage is an increasingly popular solution for this kind of workload
* On the OLTP side, we saw storage engines from two main schools of thought: 
  * log-structured school - only permits appending to files and deleting obsolete files, but never updates a file that has been written
    * Bitcask, SSTables, LSM-trees, LevelDB, Cassandra, HBase, Lucene, and others belong to this group
  * update-in-place school, which treats the disk as a set of fixed-size pages that can be overwritten 
    * B-trees are the biggest example of this philosophy, being used in all major relational databases and also many nonrelational ones
* log-structured storage engines are a comparatively recent development
  * key idea is that they systematically turn random-access writes into sequential writes on disk, which enables higher write throughput due to the performance characteristics of hard drives and SSDs
* analytic workloads are so different from OLTP: when your queries require sequentially scanning across a large number of rows, indexes are much less relevant
* instead it becomes important to encode data very compactly, to minimize the amount of data that the query needs to read from disk
* column-oriented storage helps achieve this goal
* goal: if you’re armed with this knowledge about the internals of storage engines, you are in a much better position to know which tool is best suited for your particular application
  * if you need to adjust a database’s tuning parameters, this understanding allows you to imagine what effect a higher or a lower value may have
  * hopefully equipped you with enough vocabulary and ideas that you can make sense of the documentation for the database of your choice 































  
  
  









