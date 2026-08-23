# Chapter 4: Storage and Retrieval

## Overview

### Two Core Database Functions

1. **Storage**: Store data given to database
2. **Retrieval**: Return data when requested

### Why Understanding Storage Matters

**For application developers**: 
- Not implementing storage engines from scratch
- Need to select appropriate storage engine for application
- Configure storage engine for specific workload
- Need rough idea of what happens under the hood

### Key Distinction

**OLTP vs OLAP** (introduced in Chapter 1):
- **OLTP storage engines**: Optimized for transactional workloads (high volume small reads/writes)
- **OLAP storage engines**: Optimized for analytics (complex queries on large data)

### Chapter Structure

1. OLTP storage engines
   - Log-structured (append-only)
   - B-trees (update-in-place)
2. Indexes for complex queries
3. Analytics storage

---

## Simplest Database: Append-Only Log

### Implementation in Bash

```bash
db_set() {
  echo "$1,$2" >> database
}

db_get() {
  grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
}
```

**How it works**:
- `db_set key value`: Appends key-value pair to file (CSV-like format)
- `db_get key`: Scans file for most recent value of key

### Performance Characteristics

**Write performance**: Excellent
- Appending to file very efficient
- Simple, matches how real databases work internally
- Logs: append-only data files used by many databases

**Read performance**: Terrible
- Must scan entire file for each lookup
- O(n) algorithmic complexity
- Doubles in time if data volume doubles

### Log Definition

**In this book**: Append-only sequence of records on disk
- Not application logs (text descriptions)
- Can be binary, internal-only
- General sense of sequential disk record

---

## Index Problem and Solution

### The Challenge

**Trade-off**: Appending to file fast for writes, but slow for reads

**Solution**: Use an index
- Additional data structure derived from primary data
- Doesn't affect database contents
- Affects only query performance
- Can be added/removed without changing data

### Index Trade-offs

**Benefits**:
- Well-chosen indexes speed up read queries

**Costs**:
- Additional disk space
- Slows down writes (index must be updated)
- Sometimes substantially

### Index Strategy

**Database practice**:
- Don't index everything by default
- Require manual selection of indexes
- Use knowledge of typical query patterns
- Choose indexes giving greatest benefit without excessive write overhead

---

## Log-Structured Storage

### In-Memory Hash Map Approach

**Concept**: Keep hash map of all keys → byte offset in log file

**How it works**:
1. Maintain hash map in memory
2. Append new key-value pair to log file
3. Update hash map with new offset
4. For reads: Use hash map to find offset, seek to location, read value

**Advantages**:
- Much faster than scanning entire file
- If data in filesystem cache, no disk I/O needed

**Problems**:

1. **Disk space**: Never free disk space for overwritten entries
   - Keep writing → eventually run out of space

2. **Restarts**: Hash map not persisted
   - Must rebuild by scanning entire log
   - Slow if large amount of data

3. **Memory limitation**: Hash map must fit in memory
   - In-disk hash maps perform poorly (random I/O, expensive to grow, collision handling complex)

4. **Range queries**: Inefficient
   - Can't easily scan keys 10000-19999
   - Must look up each key individually

---

## SSTable (Sorted Strings Table)

### Definition and Format

**Structure**: Key-value pairs stored in sorted-by-key format
- Each key appears only once in file
- Keys sorted

**Advantages over hash map**:
- Can use sparse index (store only first key of each block)
- Fewer keys must be kept in memory
- Supports efficient range queries

### Sparse Index Implementation

**Concept**: Group key-value pairs into kilobyte-sized blocks

**Sparse index**:
- Stores first key of each block
- Uses small amount of memory
- Separate index structure (immutable B-tree, trie, etc.)

**Example**:
- Block 1 starts with "handbag"
- Block 2 starts with "handsome"
- Looking for "handiwork" (not in index)
- Know it's between handbag and handsome
- Seek to handbag offset, scan forward

**Block scanning**: Few kilobytes scanned quickly

### Compression in SSTables

**Benefit**: Saves disk space, reduces I/O bandwidth

**Cost**: Additional CPU time

---

## Constructing and Merging SSTables

### The Problem with Simple SSTable Writes

**Issue**: Can't append to end of file (would lose sort order)

**Solution**: Log-structured approach (hybrid between append-only log and sorted file)

### Log-Structured Algorithm

**Step 1: In-memory memtable**
- Write comes in → add to in-memory ordered structure
- Structures: red-black tree, skip list, trie
- Can insert keys in any order
- Efficient lookups and sorted output
- Called "memtable"

**Step 2: Disk flush**
- When memtable exceeds threshold (few MB typically)
- Write to disk as sorted SSTable file
- New file = most recent database segment
- Separate file alongside older segments
- Each segment has separate index
- Old memtable freed, new one created for writes

**Step 3: Read algorithm**
1. Check memtable
2. Check most recent on-disk segment
3. Keep looking in next-older segment until found or end
4. If not in any segment, doesn't exist

**Step 4: Background compaction**
- Periodically merge segment files
- Discard overwritten/deleted values

### Merging Algorithm

**Process**: Similar to mergesort
1. Read input files side-by-side
2. Look at first key in each file
3. Copy lowest key to output file
4. Repeat
5. If same key in multiple files, keep only newest value

**Benefits**:
- Produces new merged segment (sorted by key)
- One value per key
- Minimal memory usage (iterate one key at a time)

**Illustrated in Figure 4-3**: Merging maintains sort order, latest values

### Crash Recovery and Deletion

**Memtable durability**:
- Separate append-only log on disk
- Every write appended immediately
- Restores memtable after crash
- Log discardable after memtable written to SSTable

**Deletion**:
- Append special deletion record (tombstone) to data file
- During merge: Tombstone tells process to discard previous values for deleted key
- Once tombstone merged to oldest segment, can be dropped

### LSM-Tree Algorithm

**Full name**: Log-Structured Merge-tree

**Publications**:
- Originally published 1996 with LSM-tree name
- Based on earlier work on log-structured filesystems

**Implementations**: RocksDB, Cassandra, ScyllaDB, HBase (all inspired by Google Bigtable)
- Bigtable paper introduced SSTable and memtable terms

### Immutability and Crash Recovery

**Key characteristic**: Segment files written once, then immutable

**Advantages**:
- Merging and compaction happen in background thread
- Continue serving reads during merge (use input segments)
- Switch to merged segment when complete, delete input files
- Crash during merge: Delete unfinished SSTable, start fresh
- Log checksums detect corrupted/incomplete entries

### Object Storage Compatibility

**Modern approach**: Segment files well-suited for object storage
- Examples: SlateDB, Delta Lake
- Immutability fits object storage model

---

## Bloom Filters

### Purpose

**Problem**: Slow to read key not found recently, or doesn't exist
- Storage engine checks several segment files

**Solution**: Bloom filter provides fast approximate check

### How Bloom Filters Work

**Basic structure** (Figure 4-4):
- Array of bits (16 in example, more in practice)
- For each key: Compute hash → set of bit indexes
- Set bits corresponding to those indexes to 1

**Example**:
- Key "handbag" hashes to (2, 9, 4)
- Set bits 2, 9, 4 to 1
- Store bitmap with sparse index

### Bloom Filter Queries

**To check if key exists**:
1. Compute same hash of key
2. Check bits at those indexes

**Interpretation**:
- At least one bit is 0 → Key definitely NOT in SSTable
- All bits are 1 → Key likely in SSTable (but false positive possible)

**False positive**: Key looks present even though it isn't
- No harm: Check sparse index and decode block anyway
- No harm: Continue search with next segment

### False Positive Probability

**Factors**:
- Number of keys
- Number of bits per key
- Total bits in filter

**Tuning**: Online calculators available

**Rule of thumb**:
- 10 bits per key → 1% false positive rate
- 5 additional bits per key → 10× reduction in false positives

### LSM Bloom Filter Strategy

**Advantages**:
- If Bloom filter says key NOT present → safely skip SSTable (definitely not there)
- If Bloom filter says key present → check sparse index/decode block

---

## Compaction Strategies

### Context

**Important detail**: When to perform compaction, which SSTables to include

**Configurability**: Many LSM systems allow strategy selection

### Size-Tiered Compaction

**Algorithm**:
- Merge newer, smaller SSTables into older, larger ones
- Example: Four 256 MB SSTables → one 898 MB SSTable (smaller due to deletions)
- Older SSTables get very large

**Characteristics**:
- Merging requires significant temporary disk space
- Handles very high write throughput
- Most data rewritten only few times in larger sequential merges

**Best for**: Write-heavy workloads with few reads

### Leveled Compaction

**Algorithm**:
- Keep fixed SSTable sizes
- Group into increasing levels (L0, L1, L2, ...)
- L0 = most recent data
- Beyond L0: Key range-partitioned SSTables
- Each level larger than previous
- When level exceeds size limit: Merge with next level

**Example organization**:
- L1: Two SSTables (keys a-m, n-z)
- L2: Larger, more SSTables
- Each level has size limit

**Characteristics**:
- Compaction more incremental
- Less temporary disk space than size-tiered
- More efficient for reads (fewer SSTables to check)

### Strategy Selection

**Rule of thumb**:
- **Size-tiered**: Better if mostly writes, few reads
- **Leveled**: Better if dominated by reads
- **Intermediate**: Write small number of keys frequently, large number rarely

**Implementations**: Most LSM-trees provide multiple strategies

---

## Embedded Storage Engines

### Definition

**Contrast to networked databases**: 
- Run as services accepting queries over network
- Embedded databases: Libraries in same process as application code
- Read/write local disk files
- Interact through normal function calls

### Examples

RocksDB, SQLite, LMDB, DuckDB, KùzuDB

### Use Cases

**Mobile apps**: Common for storing local user data

**Backend systems**:
- Appropriate if data small enough for single machine
- Not many concurrent transactions
- Multitenant systems (each tenant small, separate) possible
- One embedded instance per tenant

### Note

**Storage/retrieval methods** discussed in this chapter used in both embedded and client/server databases
- Chapters 6, 7 discuss scaling across multiple machines

---

## B-Trees

### Historical Context

**Introduced**: 1970

**Adoption**: Called "ubiquitous" less than 10 years later

**Current status**:
- Standard index implementation in almost all relational databases
- Many nonrelational databases use them too
- Stood test of time very well

### Similarity to SSTables

- Keep key-value pairs sorted
- Efficient key-value lookups
- Support range queries

### Key Differences

**Log-structured**:
- Variable-size segments (several MB+)
- Written once, then immutable

**B-tree**:
- Fixed-size blocks (pages)
- 4 KiB traditional (PostgreSQL 8 KiB, MySQL 16 KiB default)
- May overwrite page in place

### B-Tree Structure

**Fundamentals**:
- Each page identifiable by page number
- One page references another using page number (like disk pointer)
- Page number × page size = byte offset in file

**Hierarchy**:
- Root page: Starting point for lookups
- Contains keys and references to child pages
- Each child responsible for continuous key range
- Keys between references indicate range boundaries

**Example** (Figure 4-5):
- Looking for key 251
- Root shows 200-300 range
- Follow reference to that page
- Further subdivides into 250-270 range
- Eventually reach leaf page with individual keys

**Branching factor**: Number of references to child pages
- Example: 6 in Figure 4-5
- Typically several hundred in practice

### B-Tree Updates

**Update existing key**:
1. Search for leaf page containing key
2. Overwrite page on disk with new value

**Add new key**:
1. Find page whose range encompasses key
2. Add to that page
3. If not enough space → split page into two half-full pages
4. Update parent page with new subdivision boundaries

### Page Splitting

**Process** (Figure 4-6):
- Want to insert key 334 in page for range 333-345 (already full)
- Split into: 333-337 (with new key 334) and 337-345
- Update parent page with references to both children, boundary value 337
- If parent lacks space → parent split continues recursively
- Splits can propagate to root
- Root split → new root created above

**Result**: Tree remains balanced
- B-tree with n keys always has depth O(log n)
- Most databases fit into 3-4 level tree
- 4-level tree of 4 KiB pages, branching factor 500 → 250 TB max

### Key Deletion

**More complex**: May require node merging (beyond scope here)

---

## Making B-Trees Reliable

### Fundamental Write Operation

**Assumption**: Overwrite of page on disk doesn't change location
- All references to page remain intact
- Different from log-structured (never modify, only append)

### The Danger

**Multi-page operation risk**: Page split overwrites multiple pages
- If crash after some pages written → corrupted tree
- Orphan pages (not child of any parent)
- Torn page: Hardware can't atomically write entire page

### Write-Ahead Log (WAL)

**Solution**: Append-only file recording every B-tree modification

**How it works**:
1. Every B-tree modification written to WAL
2. Only then applied to tree pages
3. On recovery from crash: Use log to restore consistent state

**Filesystem equivalent**: Journaling

### Performance Optimization

**Strategy**:
- Don't immediately write every modified page to disk
- Buffer B-tree pages in memory
- WAL ensures data not lost in crash
- Once data written to WAL and flushed with fsync, durable

---

## B-Tree Variants

### Copy-on-Write Scheme

**Alternative to WAL**:
- Modified page written to different location
- New parent page versions created pointing to new location

**Examples**: LMDB

**Benefits**: Also useful for concurrency control (Chapter 11)

### Key Abbreviation

**Optimization**: Don't store entire key, abbreviate it
- Interior pages only need enough to distinguish key ranges
- Pack more keys into page → higher branching factor → fewer levels

### Leaf Page Ordering

**Optimization**: Layout leaf pages in sequential disk order
- Reduces disk seeks for sorted scans
- Difficult to maintain as tree grows

### Sibling Pointers

**Enhancement**: Add pointers to left/right sibling leaf pages
- Scan keys in order without jumping to parents

---

## Comparing B-Trees and LSM-Trees

### General Rule of Thumb

- **LSM-trees**: Better for write-heavy applications
- **B-trees**: Faster for reads

**Important caveat**: Benchmarks sensitive to workload details
- Must test with your particular workload
- Not strict either/or (hybrid approaches exist)

### Read Performance

**B-tree reads**:
- One page per level
- Usually small number of levels
- Generally fast, predictable performance

**LSM reads**:
- Check multiple SSTables at different compaction stages
- Bloom filters reduce disk I/O needed
- Both approaches can perform well depending on workload

**Range queries**:
- B-trees: Simple and fast (use sorted structure)
- LSM: Can use SSTable sorting, but scan all segments in parallel, combine results
- Bloom filters don't help range queries (can't hash every possible key)
- Range queries more expensive in LSM

**Read throughput**:
- Modern SSDs (NVMe via PCIe, not SATA) perform many independent reads in parallel
- Both LSM and B-trees can provide high throughput with careful design

**Latency spikes**:
- High write throughput can cause LSM latency spikes
- If memtable fills faster than compaction can keep up
- Storage engines apply backpressure: Suspend reads/writes until memtable flushed

### Sequential vs Random Writes

**B-tree pattern**: Scattered writes across key space
- Disk operations also scattered randomly
- Small, scattered writes = "random writes"

**LSM pattern**: Entire segment files at once
- Larger writes = "sequential writes"
- Disks have higher sequential write throughput

**Disk type impact**:
- **HDDs**: Difference large (mechanical limitations)
- **SSDs**: Difference smaller but noticeable

### Sequential vs Random Writes on SSDs

**HDD**: Sequential much faster (mechanical disk head movement)

**SSD characteristics**: 
- Read/write in 4 KiB pages
- Erase only in 512 KiB blocks
- Some pages contain valid data, others obsolete
- Before erasing block: Move valid pages to other blocks (garbage collection)

**Sequential writes**:
- Larger chunks at once
- Whole 512 KiB block likely from single file
- When file deleted → whole block erased, no GC needed

**Random writes**:
- Block contains mix of valid/invalid data
- GC must do more work before block erasure
- GC consumes write bandwidth
- GC contributes to wear

### Write Amplification

**Definition**: Total bytes written to disk / bytes you'd write with append-only log

**LSM-trees**:
- Value written to log, then memtable written, then rewritten during each compaction
- Large values stored separately (reduces overhead)

**B-trees**:
- Data written twice: Once to WAL, once to tree page
- Sometimes write entire page even if few bytes changed (crash recovery)

**Lower write amplification**: LSM-trees typically better
- Don't write entire pages
- Can compress SSTable chunks

**Measurement**: Run experiments long enough for effects visible
- Empty LSM-tree: No compaction yet, full bandwidth to writes
- Growing database: Compaction shares bandwidth

**Importance**: 
- Write-heavy bottleneck: Affects throughput
- SSD wear: Lower write amplification = less wear

### Disk Space Usage

**B-tree fragmentation**:
- Delete many keys → pages unused, in middle of file
- Can't easily return to OS (in middle of file)
- Background process (PostgreSQL vacuum) moves pages

**LSM-tree advantages**:
- Compaction periodically rewrites files
- No unused pages in SSTables
- Better compression possible
- Overwritten data stays until compaction removes it (low overhead with leveled)
- Size-tiered uses more space, especially during compaction

**GDPR/data deletion**:
- Problem: Multiple disk copies of data
- LSM challenge: Deleted data persists until tombstone through all levels
- Specialized designs can propagate deletions faster

**Snapshots**:
- LSM advantage: Immutable files
- Write memtable, record which segments existed
- No need to copy, just preserve segment files
- B-tree difficulty: Pages overwritten, snapshot less efficient

---

## Multicolumn and Secondary Indexes

### Primary Key vs Secondary Index

**Primary key index**:
- Uniquely identifies one row (relational), document (document DB), vertex (graph DB)
- References from other records use primary key
- Like discussed so far

**Secondary indexes**:
- Search by columns other than primary key
- Very common in relational databases
- CREATE INDEX command in SQL

### Example

**Relational schema** (Figure 3-1):
- Secondary indexes on user_id in positions, education, contact tables
- Find all rows belonging to same user across tables

### Construction

**From key-value index**:
- Main difference: Values not necessarily unique
- Multiple rows under same index entry

**Solutions**:
1. Each value = list of row identifiers (postings list, like full-text)
2. Make unique by appending row identifier

**Storage engines**: Both in-place (B-trees) and log-structured can implement

---

## Storing Values Within Index

### Clustered Index

**Definition**: Actual data stored directly within index structure

**Examples**:
- MySQL InnoDB: Primary key always clustered index
- SQL Server: Can specify one clustered index per table

### Heap File Approach

**Alternative**: Value = reference to actual data
- Either: Primary key of row
- Or: Direct disk location reference
- Heap file: Where rows stored, no particular order
- Can be append-only or track deleted rows for reuse

**Example**: PostgreSQL uses heap file approach

### Covering Index / Index with Included Columns

**Middle ground**: Store some table columns within index
- Plus full row on heap or primary-key clustered index
- Some queries answered by index alone (covers query)
- No heap lookup needed

**Trade-offs**:
- Faster queries
- More disk space (data duplication)
- Slower writes

### Updating Without Key Changes

**Heap file approach**: Can overwrite in place if new value ≤ old value

**Problem**: If new value larger
- Must move to new heap location
- All indexes must update to point to new location
- Or leave forwarding pointer at old location

---

## Keeping Everything in Memory

### Historical Context

**Motivation for disk storage**:
- Durability (contents not lost when power off)
- Lower cost per gigabyte than RAM

**Current trend**: RAM getting cheaper
- Many datasets not large
- Feasible to keep entire dataset in memory
- Potentially distributed across machines

### In-Memory Database Types

**Caching-only** (Memcached):
- Data loss acceptable on restart
- No durability

**Durable in-memory databases**:
- Durability via special hardware (battery-backed RAM)
- OR: Append-only disk log, periodic snapshots, replication
- Reload from disk/network replica on restart

**Examples**: VoltDB, SingleStore, Oracle TimesTen, RAMCloud, Redis, Couchbase

### Performance Advantage Misconception

**Common claim**: Faster because no disk reads

**Reality**: 
- Disk-based engine may never read disk (OS caches in memory if enough RAM)
- Real advantage: Avoid encoding data structures in disk-writable form

**In-memory advantage**: Overheads of managing on-disk data structures removed

### Interesting Use Cases

**Complex data models**:
- Difficult with disk-based indexes
- Example: Redis offers database-like interface to data structures
- Priority queues, sets
- In-memory simpler implementation

---

## Data Storage for Analytics

### Context

**Data warehouse model**: Usually relational (SQL good fit)

**Tools**: Graphical analysis tools generate SQL, visualize, explore (drill-down, slicing/dicing)

### OLTP vs OLAP Internals

**Surface similarity**: Both have SQL interface

**Internals different**: Optimized for different patterns

**Vendor trend**: Focus on either transaction processing OR analytics (not both)

### HTAP Databases

**Examples**: Microsoft SQL Server, SAP HANA, SingleStore

**Support**: Both transaction processing and data warehousing

**Implementation**: Increasingly two separate storage/query engines
- Common SQL interface
- Different underlying systems

---

## Cloud Data Warehouses

### Traditional Vendors

**Examples**: Teradata, Vertica, SAP HANA
- On-premises commercial licenses
- Cloud-based solutions

### Cloud-Only Options

**Examples**: Google BigQuery, Amazon Redshift, Snowflake

**Advantages**:
- Scalable cloud infrastructure (object storage, serverless)
- Better cloud service integration
- Automatic log ingestion
- Easy integration with frameworks (Dataflow, Kinesis)
- More elastic (compute/storage decoupled)

**Architecture**:
- Persist data in object storage (not local disks)
- Easy to adjust capacity/compute independently

### Open Source Evolution

**Traditional**: Integrated single systems (Hive)

**Modern**: Components separated
- Data moved to data lakes on object storage

**Separate components**:

1. **Query engine** (Trino, DataFusion, Presto)
   - Parse SQL, optimize to execution plans
   - Distributed parallel processing
   - Built-in or third-party execution (Spark, Flink)

2. **Storage format** (Parquet, ORC, Lance, Nimble)
   - Encode rows as bytes in file
   - Object storage or distributed filesystem
   - Accessible by other applications

3. **Table format** (Iceberg, Delta)
   - Files immutable once written
   - Support inserts/deletions
   - Schema specification
   - Features: Time travel, GC, transactions

4. **Data catalog** (Polaris, Unity Catalog, Iceberg catalog)
   - Define which tables in database
   - Create, rename, drop tables
   - Often standalone service (REST API)
   - Query engines use for reading/writing
   - Data discovery/governance integration

---

## Column-Oriented Storage

### Motivation

**Problem**: Fact tables 100+ columns, wide
- Query accesses 4-5 columns at time
- SELECT * rarely needed for analytics
- Still must load all 100 columns from disk

**Solution**: Column-oriented storage

### How It Works

**Concept**: Store all column values together
- NOT row-oriented (all row values together)

**Benefit**: Read only needed columns
- Saves work, disk bandwidth

**Example** (Figure 4-7):
- Fact table with 100 columns
- Query needs only 3-5
- Load only those columns

### Storage Organization

**Practice**: Don't store entire column as one
- Break into blocks (thousands/millions rows)
- Within each block: Separate column storage
- Blocks often by timestamp range

**Benefits**: 
- Query needs only overlapping date range blocks
- Load fewer blocks

### Current Adoption

**Ubiquitous in analytics**:
- Cloud warehouses (Snowflake)
- Single-node embedded (DuckDB)
- Product analytics (Pinot, Druid)
- Formats: Parquet, ORC, Lance, Nimble
- In-memory: Apache Arrow, Pandas/NumPy
- Time-series: InfluxDB IOx, TimescaleDB

### Wide-Column Note

**NOT to confuse with**:
- Wide-column data model (Bigtable, HBase, Accumulo)
- Rows can have thousands of columns
- No need all rows have same columns
- ACTUALLY row-oriented (all row values together)
- Despite name similarity

---

## Column Compression

### Why Compression Works

**Repetition**: Column values often repeat

**Example** (Figure 4-7): Fair amount of repetition in sequences

**Technique choice**: Depends on column data

### Bitmap Encoding

**Effective for data warehouses**

**How it works** (Figure 4-8):
- If n distinct values in column
- Create n separate bitmaps (one per value)
- Bit per row: 1 if row has value, 0 if not

**Example**:
- product_sk column with thousands of products
- Create 3-bitmap for each product
- Bit set if row has that product

**Sparse handling**:
- Bitmaps contain many 0s (sparse)
- Run-length encoding: Count consecutive 0s/1s
- Roaring bitmaps: Switch between representations (compact)

### Bitmap Index Queries

**WHERE clause examples**:

1. `WHERE product_sk IN (31, 68, 69)`:
   - Load 3 bitmaps
   - Bitwise OR efficiently

2. `WHERE product_sk = 30 AND store_sk = 3`:
   - Load 2 bitmaps
   - Bitwise AND (columns same row order, kth bits correspond)

**Graph queries**: Can find users in social network matching conditions

---

## Sort Order in Column Storage

### Key Concept

**Sorting**: Rows sorted as units, NOT columns independently
- Otherwise lose which items belong to same row
- Reconstruct row via kth item in each column

### Practical Approach

**Implementation**: Sort entire row, store by column

**Selection**: DBA chooses sort columns by query knowledge

**Example**:
- Date queries common → date_key first sort key
- Scan only last month blocks
- Product grouping beneficial → product_sk second key

### Compression Benefit

**Best on first sort key**:
- If primary key few distinct values
- After sorting → long sequences of repeated values
- Run-length encoding compresses dramatically
- Even billions of rows → few kilobytes

**Diminishing returns**:
- Second/third keys more jumbled
- Further keys randomized
- Compression less effective

---

## Writing to Column Storage

### OLAP Write Pattern

**Reads**: Aggregations over large row numbers
- Column storage + compression + sorting help

**Writes**: Bulk imports (ETL)
- Insert individual row in middle inefficient (rewrite all compressed columns after)
- Bulk write amortizes cost

### Log-Structured Approach

**Strategy**:
1. Writes go to row-oriented, sorted, in-memory store
2. Accumulate
3. Merge with column-encoded disk files
4. Write new files in bulk
5. Old files remain immutable

**Storage**: Object storage good fit

### Query Execution

**Two-level query**:
- Examine column data on disk
- Examine recent writes in memory
- Combine results
- Query engine hides distinction

**Perception**: Inserts/updates/deletes immediately reflected

**Examples**: Snowflake, Vertica, Pinot, Druid

---

## Query Execution: Compilation and Vectorization

### Challenge

**Complex SQL analytics query**: Multi-stage query plan
- Distributed across machines for parallel execution

**Optimization**: Query planner chooses operators, order, placement

### Column Operations

**Requirements**:
- Find rows where value in particular set (joins)
- Check if value > 15
- Look at multiple columns for same row (product="bananas" AND store=specific)

### CPU Cost

**Concern**: Millions of rows scanned
- Disk throughput important
- CPU time also critical

### Naive Interpreter Approach

**Problem**: Interpreter too slow
- Iterate each row
- Check data structure for operations
- Perform comparisons/calculations

### Alternative Approaches

**Query Compilation**:
1. SQL → generated code
2. Iterate rows, check columns, perform comparisons
3. Copy needed values to output buffer
4. Compile generated code to machine code (LLVM)
5. Run on loaded column data
6. Similar to JIT compilation (JVM, others)

**Vectorized Processing**:
1. Interpret query (not compile)
2. Process many values in batch
3. Predefined operators built-in
4. Pass arguments, get batch results
5. Example (Figure 4-9):
   - Pass product_sk column + product ID → equality operator → bitmap
   - Pass store_sk column + store ID → equality operator → bitmap
   - Pass 2 bitmaps to bitwise AND operator → result bitmap

### Performance Optimizations

**Both approaches achieve good performance**:

- **Sequential memory access**: Prefer over random (cache misses)
- **Tight loops**: Minimize instructions, avoid function calls (pipeline, branch misprediction)
- **Parallelism**: Multiple threads, SIMD instructions
- **Compressed data**: Operate directly without decoding (saves memory, copying)

---

## Materialized Views and Data Cubes

### Materialized Views

**Definition**: Table-like object whose contents = query results

**Difference from virtual view**:
- Virtual: Shortcut, expanded on-the-fly
- Materialized: Actual copy on disk

**Update challenge**: Must update when underlying data changes
- Some databases automatic
- Specialist systems (Materialize) focus on this

**Trade-off**: More write work, but improved read performance
- Useful for repeated same queries

### Materialized Aggregates

**Use case**: Data warehouses often aggregate (COUNT, SUM, AVG, MIN, MAX)

**Optimization**: Cache frequently-used aggregates

**Data cube (OLAP cube)**: Grid of aggregates grouped by dimensions

### Data Cube Example (Figure 4-10)

**Setup**: 2 dimensions (date_key, product_sk)

**Structure**:
- 2D table (dates × products)
- Each cell = aggregate (SUM) for that date-product combination
- Apply aggregate along rows/columns → reduced by one dimension

**Multi-dimensional**:
- Typical: 5 dimensions (date, product, store, promotion, customer)
- Principle same (hypercube)
- Each cell = aggregate for combination

**Query benefit**: Certain queries very fast
- Example: Total sales per store yesterday → look at appropriate dimension totals
- No need to scan millions of rows

### Data Cube Trade-offs

**Advantage**: Precomputed queries very fast

**Disadvantage**: Less flexible than raw data
- Example: Can't calculate proportion >$100 if price not dimension
- Data cubes designed for specific queries

**Strategy**: Keep raw data, use aggregates as performance boost

---

## Multidimensional and Full-Text Indexes

### Beyond Single Column

**B-trees, LSM-trees**: Range queries on single attribute

**Need**: Query multiple columns simultaneously

### Concatenated Index

**Definition**: Combine multiple fields into one key (append columns)

**Analogy**: Old phone book (lastname, firstname) → phone number

**Capabilities**:
- Find people with particular last name
- Find people with particular (lastname, firstname)

**Limitation**:
- Useless for finding by first name only

### Multidimensional Indexes

**Purpose**: Query multiple columns simultaneously

**Importance**: Geospatial data (latitude, longitude)

**Use case example**: Restaurant search on map
- User viewing rectangular area
- Need all restaurants within latitude/longitude ranges
- Two-dimensional range query

**Concatenated limitation**: Can't answer efficiently
- Either: All restaurants in latitude range (any longitude)
- Or: All restaurants in longitude range (any latitude)
- Not both simultaneously

### Spatial Indexing Solutions

**Space-filling curve**:
- Translate 2D location to single number
- Use regular B-tree index

**Specialized spatial indexes**:
- R-trees, Bkd-trees
- Divide space so nearby points grouped in same subtree

**Implementations**:
- PostGIS (R-trees on PostgreSQL)
- Grid-based: Regularly spaced triangles, squares, hexagons

### Multidimensional Beyond Location

**Not just geographic**:
- 3D: (red, green, blue) for color ranges (ecommerce)
- 2D: (date, temperature) for year/temperature ranges (weather data)

**Advantage**: Narrow results by multiple dimensions simultaneously
- 1D approach: Scan year (ignore temperature), then filter OR vice versa
- 2D approach: Narrow by both simultaneously

---

## Full-Text Search

### Purpose

**Goal**: Search collection of text (web pages, descriptions, etc.) by keywords

**Scope**: Keywords anywhere in text

**Challenges**: Language-specific (Asian languages no spaces), typos, grammatical forms, synonyms
- Beyond this book's scope

### Multidimensional Analogy

**Concept**: Each word (term) = dimension

**Semantics**:
- Document with term = 1 in dimension
- Document without = 0 in dimension
- "red apples" query = look for 1 in red dimension AND 1 in apples dimension

**Scaling**: Very large number of dimensions (all possible words)

### Inverted Index

**Data structure**: Key-value where key=term, value=list of document IDs (postings list)

**Representation**: Document IDs sequential
- Postings list can be sparse bitmap
- nth bit in bitmap for term = 1 if document n contains term

**Query**: Find documents with terms x and y
- Load bitmaps for x, y
- Bitwise AND efficiently
- (Similar to vectorized warehouse query, Figure 4-9)

### Implementation Examples

**Lucene**:
- Full-text indexing engine (Elasticsearch, Solr)
- SSTable-like sorted files
- Merge background (log-structured approach)

**PostgreSQL**:
- GIN index type
- Postings lists for full-text search
- Index inside JSON documents

### N-Gram Approach

**Alternative**: Find all substrings of length n

**Example**: Trigrams (n=3) of "hello": hel, ell, llo

**Benefits**:
- Search arbitrary substrings (≥n characters)
- Regular expression queries

**Downside**: Large indexes

### Typo Tolerance

**Edit distance**: Lucene can search words within certain edit distance
- Edit distance 1 = one letter added/removed/replaced

**Implementation**: 
- Finite state automaton (trie-like)
- Transform to Levenshtein automaton
- Efficient search within edit distance

---

## Vector Embeddings

### Semantic Search Beyond Strings

**Goal**: Understand document concepts, user intentions

**Challenge**: Different words, similar meaning
- "cancel subscription" vs "close account", "terminate contract"

**Application**: Retrieval-augmented generation (LLM search incorporation)

### Vector Embeddings

**Concept**: Embedding models translate text → floating-point vector

**Implementation**: Often LLMs

**Semantics**: Nearby vectors = semantically similar documents

### How It Works

**Multidimensional space**:
- Vector = point in space
- Each floating-point value = dimension position
- Embedding models generate similar vectors for similar documents

**Example**:
- Agriculture page: [0.38, 0.83, 0.41]
- Vegetables page: [0.36, 0.64, 0.67] (nearby)
- Star schema page: [0.85, 0.10, -0.52] (far)

**Real vectors**: Often 1000+ dimensions

**Principles**: Same even with many dimensions

### Embedding Models

**Models**:
- Word2Vec, BERT, GPT (text)
- Video, audio, images (specialized)
- Multimodal: Text + images (single model)

**Implementation**: Neural networks

### Semantic Search Process

1. User enters query
2. Embedding model generates query vector
3. Include user context (location, etc.)
4. Search engine finds documents with similar vectors
5. Use distance functions for similarity

**Distance functions**:
- Cosine similarity: Cosine of angle between vectors
- Euclidean distance: Straight-line distance

### Vector Index Challenges

**Problem with R-trees**: Don't work well for many dimensions

### Vector Index Types

**Flat indexes**:
- Store vectors as-is
- Query must read every vector, measure distance
- Accurate but slow

**Inverted file (IVF) indexes**:
- Cluster vector space into partitions (centroids)
- Reduce vectors to compare
- Faster than flat, approximate results
- Probes: Number of partitions to check (more=accurate but slower)

**Hierarchical Navigable Small World (HNSW) indexes** (Figure 4-11):
- Multiple vector space layers (graph structure)
- Nodes = vectors, edges = proximity
- Query starts top layer (few nodes), follows edges to closer vectors
- Move to layer below (more densely connected)
- Continue until last layer
- Approximate results (like IVF)

**Algorithm complexity**: Beyond scope, see papers

### Implementations

**Flat + IVF**: 
- Facebook's Faiss library (variations of each)

**HNSW**:
- PostgreSQL pgvector (both IVF and HNSW)

---

## Summary

### Two Main OLTP Storage Approaches

**Log-structured** (append-only):
- SSTables, LSM-trees
- RocksDB, Cassandra, HBase, ScyllaDB, Lucene
- Higher write throughput
- Merging and compaction in background

**Update-in-place** (overwrite pages):
- B-trees (ubiquitous, relational + nonrelational databases)
- Generally better for reads
- Write-ahead log for durability

### OLAP Storage Characteristics

**Optimizations**:
- Column-oriented storage layout
- Compression (bitmap, run-length)
- JIT compilation or vectorization for queries
- Minimize disk reads, CPU time

### Index Types Covered

**Range queries**: Single column (B-trees, LSM-trees)

**Multidimensional**: Geospatial (R-trees, Bkd-trees)

**Full-text search**: Inverted indexes, postings lists

**Semantic search**: Vector embeddings, vector indexes (flat, IVF, HNSW)

### Application Development Perspective

**Knowledge benefit**: Better choice of storage engine for workload
- Adjust tuning parameters with understanding
- Know what effects parameters have

**Vocabulary**: Understand database documentation better
- Make sense of options
- Configure appropriately
