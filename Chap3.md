# Chapter 3 - Storage and Retrieval

Storage Engines
1. Log-structured
2. Page-oriented

## Log structured Storage Engine
Basic database - _append-only_ .txt file storing key-values. To `set` a value,
append key-value to last line. To `get` a key value, search for key in file and
 return last occurrence.
For a read operation, you'll have to scan the entire file first. This will lead
 to worsening performance as size of txt file increases.

**Hash Indexes**

To avoid this, create an _index_. In this example, create a dictionary storing
 {key : location in txt file}. You'll have to update this dictionary every time
 you `set` a value but `get` operations will be faster now.
 This happens with every index - Writes are slowed down because the index needs
 to be updated every time data is written. But read times _can_ improve
 depending on the index chosen.

**Compaction**

After lot of writes, the size of the txt file will keep increasing.
But, we only ever want the latest value present in the txt file. So we break
the file into _segments_ and we _compact_ segments whenever segments cross a
certain file size. We throw away old/duplicate values of keys and only keep
the latest one. This reduces the size of the individual segments.
These segments can then be merged together into bigger + fewer segments.

Technicalities -
- Compacting/merging of old segments can be done in a background thread while
read/write requests are served as normal. After process is complete, redirect
queries to new segments and delete old segments.
- Note that every segment will have its own in-memory hash map. For `get`
 queries, check the hash-map of the latest/newest segment first, if it isn't
 present there, check the second-most-recent and so on.
- Deleting records - Use a _tombstone_ indicator to mark whether a record has
been deleted or not. While merging, discard any previous values for a key with
a _tombstone_ on it.
- Crash Recovery - Snapshot the in-memory hash maps to disk for quicker
recovery after a crash. Otherwise, they'll have to rebuilt by scanning the
whole segment after a crash.
- Partially written records - Use checksums to detect and ignore corrupted
parts.
- Concurrency - Use only 1 write threads to ensure correct order. Segment
files are append-only and can be read by multiple read threads.

Limitations -
- Hash table must fit in memory which is problematic when there are a huge
number of keys. It is difficult to keep a hash table on disk because - random
access is slower, expensive to grow and hash collisions could happen.
- You can not perform range queries on this data easily i.e.. scan over all
keys from key1 to key100. All 100 keys will need individual lookups.

## SSTable - Sorted String Table

New requirements:
- All key-value pairs in a segment file _must_ be sorted by key.
- Each key must occur at most once in each segment file. i.e. It is fine if
the key doesn't exist in 1 segment file, it just shouldn't occur more than once.

**Constructing SSTables**

Basic concept - Use a tree structure to store the sorted data in memory and
write it to disk to a new SSTable segment once the tree size crosses a
 size threshold. Data structures - red-black trees, AVL trees.

Workflow -
1. _Memtable_ - Add new write requests to the in-memory tree data structure.
2. When the memtable crosses a size threshold, write it to disk easily since
the keys are already sorted. This becomes the latest segment file. While this
is going on, write new writes to a new memtable.
3. Read queries - Check if key exists in memtable, then check most recent
segment file, then next-older one, etc
4. Every now and then, merge and compact segment files in the background.

Technicality - In a database crash, the memtable can be lost. To avoid this,
use a separate log on disk to store the latest writes for easier restoration.
Once the memtable is written to disk, delete this log file.


**Compaction**

To merge multiple segment files, use an algorithm similar to _mergesort_
algorithm. In each segment file, the first key will be the 'smallest' one
according to sort order. Start reading all input files, look at first key then
copy the lowest key, then increment pointer in that file. If a key is present
in multiple files, get the value from latest segment file.

**Hash Index**

You don't need to keep an index of _all_ the keys in memory anymore. A sparser
index will also work. This is because if you know the location of `apple` and
`dog`, if you want to find `cat`, you can start searching from the location of
`apple` and stop at `dog` (if `cat` doesn't exist as a key)

**Compression**
In above example, the records between `apple` and `dog` can be compressed to
save disk space and I/O bandwidth. This is because any read query (for a key
  between `apple` and `dog`) will need to scan over the key-value pairs in this
  segment anyways.

## Page-oriented Storage Engine : B-Trees
```
B-trees break the database down into fixed-size blocks or pages, traditionally
4 KB in size (sometimes bigger), and read or write one page at a time
```
Each page can link to other pages using pointers. By traversing these
pointers from the _root page_, we can reach a _leaf page_ which'll contain the
key value.

_Balanced_ - A tree with _n_ keys has a depth of O(log _n_)

_Branching factor_ - `The number of references to child pages in one page of the B-tree`

- To update a key, search for the page, update the value and write the page back.
- To add a new key, find the page which has the range for that key and add it
there. If the page can't accommodate the new data, split it into 2 new pages
 and update the parent page to note the 2 new ranges.

## Comparing B-Trees and LSM-Trees

TODO

# Transaction Processing or Analytics
- Transaction - `A group of reads and writes that form a logical unit`
- ACID - Atomicity, Consistency, Isolation, Durability
- OLTP - Online Transaction Processing
- OLAP - Online Analytics Processing
- Data Warehouse - `Separate database used to run analytics`

| Property      | OLTP                                    | OLAP                               |
|---------------|-----------------------------------------|------------------------------------|
| Read Pattern  | Few records per key, fetched by queries | Aggregation over *lots* of records |
| Write Pattern | User input, random-access, low-latency  | ETL pipeline or streams            |
| Primary Users | End user, customer                      | Analysts                           |
| Data time?    | Latest data                             | Historical data                    |
| Size          | GB - TB                                 | TB - PB                            |

Need for data warehouse? OLTP systems need low-latency writes for end customers. But if analysts run long-running queries, this'll slow the database down and degrade performance for end customers. So data warehouses are used instead by analysts. This is a read-only copy of the OLTP data.

ETL pipelines are used to extract data from OLTP databases, transform it and load it into the data warehouse. OLAP queries use different set of indexes than those used by OLTP. So the data warehouse can be designed to support these indexes instead to improve performance.


## Stars and Snowflakes
Dimensional modelling - fact tables and dimension tables related using keys. Fact tables can be individual events (the smallest granularity possible for maximum flexibility of analysis). Columns in fact tables can be foreign keys to tables in dimension tables.

`As each row in the fact table represents an event, the dimensions represent the who, what, where, when, how, and why of the event.`
```
The name “star schema” comes from the fact that when the table relationships are
visualized, the fact table is in the middle, surrounded by its dimension tables; the
connections to these tables are like the rays of a star.
```
Snowflake schema are a 'deeper' version of a star schema where dimension tables have subdimension tables.

## Column Oriented Storage
OLTP databases are in stored a _row-oriented_ fashion, where all values in a row are stored together.

Suppose an analyst wants to run the query `select product_id, sum(quantity) from sales group by product_id where month="Jan"`. Only the 3 columns month, product_id and quantity are required.

But in a row-oriented table, the engine will have to load all the rows from memory to disk, then filter out the rows where month != "Jan" and then proceed. This will take a long time.

In a _column-oriented_ database, the values from each _column_ are stored together, not on a _row_ basis. Therefore, if a 'column-oriented' database was used, only records from the 3 columns would be read and filtered on.

### Column Compression
#### Bitmap encoding
This can be used to compress data in a column-oriented database.

As of now, there are only 12 months in a year. A retailer probably has around 100,000 distinct products but billions of sale transactions.

```
We can now take a column with n distinct values and turn it into n separate bitmaps: one bitmap for each distinct value, with one bit for each row. The bit is 1 if the row has that value, and 0 if not.

If n is very small, those bitmaps can be stored with one bit per row. But if n is bigger, there will be a lot of zeros in most of the bitmaps (we say that they are sparse). In that case, the bitmaps can additionally be run-length encoded, resulting in even more compression.
```

These bitmaps can also help in faster queries. If the query includes `where product_id in (10,25,30)`, calculate the bitwise OR of the 3 bitmaps (product_id=10, product_id=25, product_id=30) to get the rows required.

#### Sort order
Data modeler can choose the column to sort a database by. This can make queries faster e.g. if a query wants to look at records from last 30 days, ordering the table by date will improve the speed (since the database will only need to scan the latest records)

This will also help in compression. If tables are sorted by dates, records with same dates will be together which will result in smaller run-length encoded bitmaps.

#### Writing to Column-Oriented Storage
Inserting rows in the middle of a sorted table is difficult since all column records will have to be updated. So an LSM-Tree is used - all writes go to an in-memory store which is then written to disk in bulk.

### Aggregation: Data Cubes and Materialised Views
For common aggregations like sum/count/min/max, do the queries once and write the results to disk. These can then be used directly instead of running the same queries multiple times. This can improve read performance (if the right view is used) at the cost of decreasing write performance (since the view has to be updated frequently).

This doesn't improve all queries (such as summing over a portion of data) but it can be a Performance boost for common queries.
