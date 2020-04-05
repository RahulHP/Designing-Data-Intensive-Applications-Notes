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
