# File Structure

A TSM file is composed for four sections: header, blocks, index and the footer.
TSM文件由四个部分组成：Header，块，索引和Footer。
```
┌────────┬────────────────────────────────────┬─────────────┬──────────────┐
│ Header │               Blocks               │    Index    │    Footer    │
│5 bytes │              N bytes               │   N bytes   │   4 bytes    │
└────────┴────────────────────────────────────┴─────────────┴──────────────┘
```
Header is composed of a magic number to identify the file type and a version number.
标题由一个Magic和version组成，用于标识文件类型和版本号。
```
┌───────────────────┐
│      Header       │
├─────────┬─────────┤
│  Magic  │ Version │
│ 4 bytes │ 1 byte  │
└─────────┴─────────┘
```

Blocks are sequences of block CRC32 and data.  The block data is opaque to the file.  The CRC32 is used for recovery to ensure blocks have not been corrupted due to bugs outside of our control.  The length of the blocks is stored in the index.
块是块CRC32和数据的序列。 块数据对文件是不透明的。 CRC32用于恢复，以确保块由于我们控制之外的错误而未被破坏。 块的长度存储在索引中。

```
┌───────────────────────────────────────────────────────────┐
│                          Blocks                           │
├───────────────────┬───────────────────┬───────────────────┤
│      Block 1      │      Block 2      │      Block N      │
├─────────┬─────────┼─────────┬─────────┼─────────┬─────────┤
│  CRC    │  Data   │  CRC    │  Data   │  CRC    │  Data   │
│ 4 bytes │ N bytes │ 4 bytes │ N bytes │ 4 bytes │ N bytes │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

Following the blocks is the index for the blocks in the file.  The index is composed of a sequence of index entries ordered lexicographically by key and then by time.  Each index entry starts with a key length and key followed by a count of the number of blocks in the file.  Each block entry is composed of the min and max time for the block, the offset into the file where the block is located and the size of the block.
块之后是文件中块的索引。 索引由一系列索引条目组成，这些索引条目按键按字典顺序排序，然后按时间排序。 每个索引条目都以密钥长度和密钥开头，后跟文件中块数的计数。 每个块条目由块的最小和最大时间，块所在文件的偏移量以及块的大小组成。

The index structure can provide efficient access to all blocks as well as the ability to determine the cost associated with accessing a given key.  Given a key and timestamp, we know exactly which file contains the block for that timestamp as well as where that block resides and how much data to read to retrieve the block.  If we know we need to read all or multiple blocks in a file, we can use the size to determine how much to read in a given IO.
索引结构可以提供对所有块的有效访问，以及确定与访问给定密钥相关的成本的能力。 给定密钥和时间戳，我们确切地知道哪个文件包含该时间戳的块以及该块所在的位置以及要检索块的数据量。 如果我们知道我们需要读取文件中的所有或多个块，我们可以使用该大小来确定给定IO中的读取量。

_TBD: The block length stored in the block data could probably be dropped since we store it in the index._
存储在块数据中的块长度可能会被丢弃，因为我们将其存储在索引中

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                   Index                                    │
├─────────┬─────────┬──────┬───────┬─────────┬─────────┬────────┬────────┬───┤
│ Key Len │   Key   │ Type │ Count │Min Time │Max Time │ Offset │  Size  │...│
│ 2 bytes │ N bytes │1 byte│2 bytes│ 8 bytes │ 8 bytes │8 bytes │4 bytes │   │
└─────────┴─────────┴──────┴───────┴─────────┴─────────┴────────┴────────┴───┘
```

最后一部分是存储索引开头偏移量的页脚。

```
┌─────────┐
│ Footer  │
├─────────┤
│Index Ofs│
│ 8 bytes │
└─────────┘
```

# File System Layout

The file system is organized a directory per shard where each shard is an integer number. Associated with each shard directory, there is a set of other directories and files:

* a wal directory - contains a set numerically increasing files WAL segment files named #####.wal.  The wal directory is separate from the directory containing the TSM files so that different types can be used if necessary.
* .tsm files - a set of numerically increasing TSM files containing compressed series data.
* .tombstone files - files named after the corresponding TSM file as #####.tombstone.  These contain measurement and series keys that have been deleted.  These files are removed during compactions.
文件系统按每个分片组织一个目录，其中每个分片是整数。 与每个分片目录相关联，还有一组其他目录和文件：

* wal目录 - 包含一组数字增加的文件WAL段文件名为#####。wal。 wal目录与包含TSM文件的目录分开，以便在必要时可以使用不同的类型。
* .tsm文件 - 一组包含压缩系列数据的数值增加的TSM文件。
* .tombstone文件 - 以相应的TSM文件命名为#####。tombstone的文件。 这些包含已删除的度量和系列键。 压缩期间会删除这些文件。

# Data Flow

Writes are appended to the current WAL segment and are also added to the Cache.  Each WAL segment is size bounded and rolls-over to a new file after it fills up.  The cache is also size bounded; snapshots are taken and WAL compactions are initiated when the cache becomes too full. If the inbound write rate exceeds the WAL compaction rate for a sustained period, the cache may become too full in which case new writes will fail until the compaction process catches up. The WAL and Cache are separate entities and do not interact with each other.  The Engine coordinates the writes to both.
写入被附加到当前WAL段，并且还被添加到高速缓存中。 每个WAL段都有大小限制，并在填满之后翻转到新文件。 缓存也是大小有限的; 当缓存变得太满时，将执行快照并启动WAL压缩。 如果入站写入速率超过WAL压缩率持续一段时间，则缓存可能变得太满，在这种情况下，新写入将失败，直到压缩过程赶上。 WAL和Cache是独立的实体，不互相交互。 引擎协调对两者的写入。

When WAL segments fill up and have been closed, the Compactor reads the WAL entries and combines them with one or more existing TSM files.  This process runs continuously until all WAL files are compacted and there is a minimum number of TSM files.  As each TSM file is completed, it is loaded and referenced by the FileStore.
当WAL段填满并关闭时，压缩器读取WAL条目并将它们与一个或多个现有TSM文件组合。 此过程持续运行，直到所有WAL文件都被压缩并且存在最少数量的TSM文件。 在完成每个TSM文件时，它将由FileStore加载和引用。

Queries are executed by constructing Cursors for keys.  The Cursors iterate over slices of Values.  When the current Values are exhausted, a Cursor requests the next set of Values from the Engine.  The Engine returns a slice of Values by querying the FileStore and Cache.  The Values in the Cache are overlaid on top of the values returned from the FileStore.  The FileStore reads and decodes blocks of Values according to the index for the file.
通过构造键的游标来执行查询。 游标迭代了一堆值。 当前值耗尽时，Cursor从Engine请求下一组值。 Engine通过查询FileStore和Cache返回一片值。 Cache中的值覆盖在FileStore返回的值之上。 FileStore根据文件的索引读取和解码值块。


Updates (writing a newer value for a point that already exists) occur as normal writes.  Since cached values overwrite existing values, newer writes take precedence.
更新（为已存在的点写入更新的值）作为正常写入发生。 由于缓存值会覆盖现有值，因此较新的写入优先。



Deletes occur by writing a delete entry for the measurement or series to the WAL and then updating the Cache and FileStore.  The Cache evicts all relevant entries.  The FileStore writes a tombstone file for each TSM file that contains relevant data.  These tombstone files are used at startup time to ignore blocks as well as during compactions to remove deleted entries.
通过将测量或系列的删除条目写入WAL然后更新Cache和FileStore来进行删除。 缓存驱逐所有相关条目。 FileStore为包含相关数据的每个TSM文件写入一个逻辑删除文件。 这些逻辑删除文件在启动时用于忽略块以及在压缩期间删除已删除的条目。

# Compactions

Compactions are a serial and continuously running process that iteratively optimizes the storage for queries.  Specifically, it does the following:
* Converts closed WAL files into TSM files and removes the closed WAL files
* Combines smaller TSM files into larger ones to improve compression ratios
* Rewrites existing files that contain series data that has been deleted
* Rewrites existing files that contain writes with more recent data to ensure a point exists in only one TSM file.
压缩是一个连续且连续运行的过程，可以迭代地优化查询的存储。 具体来说，它执行以下操作：
*将关闭的WAL文件转换为TSM文件并删除已关闭的WAL文件
*将较小的TSM文件合并为较大的TSM文件以提高压缩率
*重写包含已删除系列数据的现有文件
*重写包含具有更新数据的写入的现有文件，以确保只有一个TSM文件中存在一个点。

The compaction algorithm is continuously running and always selects files to compact based on a priority.
压缩算法持续运行，并始终根据优先级选择要压缩的文件。

1. If there are closed WAL files, the 5 oldest WAL segments are added to the set of compaction files.
2. If any TSM files contain points with older timestamps that also exist in the WAL files, those TSM files are added to the compaction set.
3. If any TSM files have a tombstone marker, those TSM files are added to the compaction set.
1.如果存在关闭的WAL文件，则将5个最旧的WAL段添加到压缩文件集中。
2.如果任何TSM文件包含WAL文件中也存在较旧时间戳的点，那么这些TSM文件将添加到压缩集中。
3.如果任何TSM文件具有逻辑删除标记，则将这些TSM文件添加到压缩集中。


The compaction algorithm generates a set of SeriesIterators that return a sequence of `key`, `Values` where each `key` returned is lexicographically greater than the previous one.  The iterators are ordered such that WAL iterators will override any values returned by the TSM file iterators.  WAL iterators read and cache the WAL segment so that deletes later in the log can be processed correctly.  TSM file iterators use the tombstone files to ensure that deleted series are not returned during iteration.  As each key is processed, the Values slice is grown, sorted, and then written to a new block in the new TSM file.  The blocks can be split based on number of points or size of the block.  If the total size of the current TSM file would exceed the maximum file size, a new file is created.
压缩算法生成一组SeriesIterators，它返回一系列`key`，`Values`，其中返回的每个`key`按字典顺序大于前一个。 对迭代器进行排序，使得WAL迭代器将覆盖TSM文件迭代器返回的任何值。 WAL迭代器读取并缓存WAL段，以便可以正确处理日志中的后续删除。 TSM文件迭代器使用逻辑删除文件来确保在迭代期间不返回已删除的系列。 在处理每个密钥时，将对值切片进行增长，排序，然后将其写入新TSM文件中的新块。 可以基于块的数量或块的大小来划分块。 如果当前TSM文件的总大小超过最大文件大小，则会创建一个新文件。



Deletions can occur while a new file is being written.  Since the new TSM file is not complete a tombstone would not be written for it. This could result in deleted values getting written into a new file.  To prevent this, if a compaction is running and a delete occurs, the current compaction is aborted and new compaction is started.
在写入新文件时可能会发生删除。 由于新的TSM文件不完整，因此不会为其编写逻辑删除。 这可能导致删除的值被写入新文件。 为了防止这种情况，如果正在运行压缩并且发生删除，则中止当前压缩并启动新压缩。


When all WAL files in the current compaction have been processed and the new TSM files have been successfully written, the new TSM files are renamed to their final names, the WAL segments are truncated and the associated snapshots are released from the cache.
当处理了当前压缩中的所有WAL文件并且已成功写入新的TSM文件时，新的TSM文件将重命名为其最终名称，WAL段将被截断，相关的快照将从缓存中释放。

The compaction process then runs again until there are no more WAL files and the minimum number of TSM files exist that are also under the maximum file size.
然后压缩过程再次运行，直到没有更多的WAL文件，并且存在最小文件大小的最小TSM文件数。

# WAL

Currently, there is a WAL per shard.  This means all the writes in a WAL segment are for the given shard.  It also means that writes across a lot of shards append to many files which might result in more disk IO due to seeking to the end of multiple files.

Two options are being considered:
目前，每个分片都有一个WAL。 这意味着WAL段中的所有写入都是针对给定的分片。 这也意味着跨越大量分片的写入会附加到许多文件，这可能会导致更多的磁盘IO，因为它们会寻找多个文件的末尾。
正在考虑两种选择：


## WAL per Shard

This is the current behavior of the WAL.  This option is conceptually easier to reason about.  For example, compactions that read in multiple WAL segments are assured that all the WAL entries pertain to the current shard.  If it completes a compaction, it is safe to remove the WAL segment.  It is also easier to deal with shard deletions as all the WAL segments can be dropped along with the other shard files.

The drawback of this option is the potential for turning sequential write IO into random IO in the presence of multiple shards and writes to many different shards.
这是WAL的当前行为。 此选项在概念上更容易推理。 例如，在多个WAL段中读取的压缩确保所有WAL条目都与当前分片相关。 如果它完成压缩，则可以安全地删除WAL段。 处理分片删除也更容易，因为所有WAL段都可以与其他分片文件一起删除。

此选项的缺点是，在存在多个分片并写入许多不同分片时，可能会将顺序写入IO转换为随机IO。


## Single WAL

Using a single WAL adds some complexity to compactions and deletions.  Compactions will need to either sort all the WAL entries in a segment by shard first and then run compactions on each shard or the compactor needs to be able to compact multiple shards concurrently while ensuring points in existing TSM files in different shards remain separate.

Deletions would not be able to reclaim WAL segments immediately as in the case where there is a WAL per shard.  Similarly, a compaction of a WAL segment that contains writes for a deleted shard would need to be dropped.

Currently, we are moving towards a Single WAL implementation.

使用单个WAL会增加压缩和删除的复杂性。 压缩将需要先通过分片对段中的所有WAL条目进行排序，然后在每个分片上运行压缩，或者压缩器需要能够同时压缩多个分片，同时确保不同分片中现有TSM文件中的点保持分离。

删除将无法立即回收WAL段，如每个分片有一个WAL的情况。 类似地，需要删除包含已删除分片的写入的WAL段的压缩。

目前，我们正在向单一WAL实施迈进。

# Cache

The purpose of the cache is so that data in the WAL is queryable. Every time a point is written to a WAL segment, it is also written to an in-memory cache. The cache is split into two parts: a "hot" part, representing the most recent writes and a "cold" part containing snapshots for which an active WAL compaction
process is underway.
缓存的目的是使WAL中的数据可查询。 每次将一个点写入WAL段时，它也会写入内存缓存。 缓存分为两部分：表示最近写入的“热”部分和包含活动WAL压缩的快照的“冷”部分
过程正在进行中。

Queries are satisfied with values read from the cache and finalized TSM files. Points in the cache always take precedence over points in TSM files with the same timestamp. Queries are never read directly from WAL segment files which are designed to optimize write rather than read performance.
查询对从缓存和已完成的TSM文件读取的值感到满意。 缓存中的点始终优先于具有相同时间戳的TSM文件中的点。 永远不会直接从WAL段文件中读取查询，这些文件旨在优化写入而不是读取性能。

The cache tracks its size on a "point-calculated" basis. "point-calculated" means that the RAM storage footprint for a point is the determined by calling its `Size()` method. While this does not correspond directly to the actual RAM footprint in the cache, the two values are sufficiently well correlated for the purpose of controlling RAM usage.
缓存在“point-calculated”的基础上跟踪其大小。 “point-calculated”意味着一个点的RAM存储空间是通过调用它的`Size（）`方法来确定的。 虽然这不直接对应于高速缓存中的实际RAM占用量，但是为了控制RAM使用，这两个值充分良好地相关。


If the cache becomes too full, or the cache has been idle for too long, a snapshot of the cache is taken and a compaction process is initiated for the related WAL segments. When the compaction of these segments is complete, the related snapshots are released from the cache.
如果高速缓存变得太满，或者高速缓存空闲时间过长，则会获取高速缓存的快照，并为相关的WAL段启动压缩过程。 完成这些段的压缩后，将从缓存中释放相关的快照。


In cases where IO performance of the compaction process falls behind the incoming write rate, it is possible that writes might arrive at the cache while the cache is both too full and the compaction of the previous snapshot is still in progress. In this case, the cache will reject the write, causing the write to fail.
Well behaved clients should interpret write failures as back pressure and should either discard the write or back off and retry the write after a delay.
如果压缩过程的IO性能落后于传入的写入速率，则当缓存太满并且前一个快照的压缩仍在进行时，写入可能会到达缓存。 在这种情况下，缓存将拒绝写入，从而导致写入失败。
表现良好的客户端应将写入故障解释为背压，并应丢弃写入或退出，并在延迟后重试写入。

# TSM File Index

Each TSM file contains a full index of the blocks contained within the file.  The existing index structure is designed to allow for a binary search across the index to find the starting block for a key.  We would then seek to that start key and sequentially scan each block to find the location of a timestamp.
每个TSM文件都包含文件中包含的块的完整索引。 现有索引结构旨在允许跨索引进行二进制搜索以查找键的起始块。 然后，我们将寻找该开始键并顺序扫描每个块以找到时间戳的位置。


Some issues with the existing structure is that seeking to a given timestamp for a key has a unknown cost.  This can cause variability in read performance that would very difficult to fix.  Another issue is that startup times for loading a TSM file would grow in proportion to number and size of TSM files on disk since we would need to scan the entire file to find all keys contained in the file.  This could be addressed by using a separate index like file or changing the index structure.
现有结构的一些问题是，为密钥寻找给定的时间戳具有未知的成本。 这可能导致读取性能的变化，这很难修复。 另一个问题是加载TSM文件的启动时间将与磁盘上TSM文件的数量和大小成比例增长，因为我们需要扫描整个文件以查找文件中包含的所有密钥。 这可以通过使用单独的索引（如文件）或更改索引结构来解决。


We've chosen to update the block index structure to ensure a TSM file is fully self-contained, supports consistent IO characteristics for sequential and random accesses as well as provides an efficient load time regardless of file size.  The implications of these changes are that the index is slightly larger and we need to be able to search the index despite each entry being variably sized.
我们选择更新块索引结构以确保TSM文件是完全独立的，支持顺序和随机访问的一致IO特性，并且无论文件大小如何都提供有效的加载时间。 这些变化的含义是索引略大，我们需要能够搜索索引，尽管每个条目的大小都是可变的。

The following are some alternative design options to handle the cases where the index is too large to fit in memory.  We are currently planning to use an indirect MMAP indexing approach for loaded TSM files.
以下是一些替代设计选项，用于处理索引太大而无法容纳在内存中的情况。 我们目前正计划对加载的TSM文件使用间接MMAP索引方法。

### Indirect MMAP Indexing

One option is to MMAP the index into memory and record the pointers to the start of each index entry in a slice.  When searching for a given key, the pointers are used to perform a binary search on the underlying mmap data.  When the matching key is found, the block entries can be loaded and search or a subsequent binary search on the blocks can be performed.

A variation of this can also be done without MMAPs by seeking and reading in the file.  The underlying file cache will still be utilized in this approach as well.

As an example, if we have an index structure in memory such as:
一种选择是将索引MMAP到内存中，并将指针记录到切片中每个索引条目的开头。 搜索给定键时，指针用于对底层mmap数据执行二进制搜索。 当找到匹配的密钥时，可以加载块条目，并且可以执行对块的搜索或随后的二进制搜索。

通过搜索和读取文件，也可以在没有MMAP的情况下完成此变体。 底层文件缓存仍将在此方法中使用。

例如，如果我们在内存中有一个索引结构，例如：

 ```
┌────────────────────────────────────────────────────────────────────┐
│                               Index                                │
├─┬──────────────────────┬──┬───────────────────────┬───┬────────────┘
│0│                      │62│                       │145│
├─┴───────┬─────────┬────┼──┴──────┬─────────┬──────┼───┴─────┬──────┐
│Key 1 Len│   Key   │... │Key 2 Len│  Key 2  │ ...  │  Key 3  │ ...  │
│ 2 bytes │ N bytes │    │ 2 bytes │ N bytes │      │ 2 bytes │      │
└─────────┴─────────┴────┴─────────┴─────────┴──────┴─────────┴──────┘
```

We would build an `offsets` slices where each element pointers to the byte location for the first key in then index slice.
我们将构建一个“偏移”切片，其中每个元素指向索引切片中第一个键的字节位置。

```
┌────────────────────────────────────────────────────────────────────┐
│                              Offsets                               │
├────┬────┬────┬─────────────────────────────────────────────────────┘
│ 0  │ 62 │145 │
└────┴────┴────┘
 ```


Using this offset slice we can find `Key 2` by doing a binary search over the offsets slice.  Instead of comparing the value in the offsets (e.g. `62`), we use that as an index into the underlying index to retrieve the key at position `62` and perform our comparisons with that.

When we have identified the correct position in the index for a given key, we could perform another binary search or a linear scan.  This should be fast as well since each index entry is 28 bytes and all contiguous in memory.

The size of the offsets slice would be proportional to the number of unique series.  If we we limit file sizes to 4GB, we would use 4 bytes for each pointer.
使用此偏移切片，我们可以通过在偏移切片上进行二分搜索来找到“密钥2”。 我们不是比较偏移量中的值（例如“62”），而是将其用作底层索引的索引，以检索位置“62”处的键并执行与之比较。

当我们在索引中识别出给定键的正确位置时，我们可以执行另一个二进制搜索或线性扫描。 这应该很快，因为每个索引条目是28个字节并且在内存中都是连续的。

偏移切片的大小将与唯一系列的数量成比例。 如果我们将文件大小限制为4GB，我们将为每个指针使用4个字节。
### LRU/Lazy Load

A second option could be to have the index work as a memory bounded, lazy-load style cache.  When a cache miss occurs, the index structure is scanned to find the key and the entries are load and added to the cache which causes the least-recently used entries to be evicted.
第二个选项可能是使索引作为内存有界，延迟加载样式缓存。 当发生高速缓存未命中时，扫描索引结构以找到密钥，并且条目被加载并添加到高速缓存中，这导致最近最少使用的条目被逐出。
### Key Compression

Another option is compress keys using a key specific dictionary encoding.   For example,
另一种选择是使用密钥特定字典编码来压缩密钥。 例如，

```
cpu,host=server1 value=1
cpu,host=server2 value=2
memory,host=server1 value=3
```

Could be compressed by expanding the key into its respective parts: measurement, tag keys, tag values and tag fields .  For each part a unique number is assigned.  e.g.
可以通过将密钥扩展到各自的部分来进行压缩：测量，标记键，标记值和标记字段。 对于每个部件，分配唯一编号。 例如

Measurements
```
cpu = 1
memory = 2
```

Tag Keys
```
host = 1
```

Tag Values
```
server1 = 1
server2 = 2
```

Fields
```
value = 1
```

Using this encoding dictionary, the string keys could be converted to a sequence of integers:
使用此编码字典，字符串键可以转换为整数序列：
```
cpu,host=server1 value=1 -->    1,1,1,1
cpu,host=server2 value=2 -->    1,1,2,1
memory,host=server1 value=3 --> 3,1,2,1
```

These sequences of small integers list can then be compressed further using a bit packed format such as Simple9 or Simple8b.  The resulting byte slices would be a multiple of 4 or 8 bytes (using Simple9/Simple8b respectively) which could used as the (string).
然后可以使用诸如Simple9或Simple8b的比特打包格式进一步压缩这些小整数列表序列。 得到的字节切片将是4或8字节的倍数（分别使用Simple9 / Simple8b），可用作（字符串）。

### Separate Index

Another option might be to have a separate index file (BoltDB) that serves as the storage for the `FileIndex` and is transient.   This index would be recreated at startup and updated at compaction time.
另一个选择可能是有一个单独的索引文件（BoltDB）作为`FileIndex`的存储并且是瞬态的。 此索引将在启动时重新创建，并在压缩时更新。

# Components

These are some of the high-level components and their responsibilities.  These are ideas preliminary.
这些是一些高级组件及其职责。 这些是初步的想法。

## WAL

* Append-only log composed of fixed size segment files.
* Writes are appended to the current segment
* Roll-over to new segment after filling the current segment
* Closed segments are never modified and used for startup and recovery as well as compactions.
* There is a single WAL for the store as opposed to a WAL per shard.

*仅附加日志由固定大小的段文件组成。
*写入附加到当前段
*填写当前分段后翻转到新分段
*封闭的段永远不会被修改并用于启动和恢复以及压缩。
*商店只有一个WAL，而不是每个分片的WAL。

## Compactor

* Continuously running, iterative file storage optimizer
* Takes closed WAL files, existing TSM files and combines into one or more new TSM files
*连续运行，迭代文件存储优化器
*关闭WAL文件，现有TSM文件并组合成一个或多个新TSM文件


## Cache

* Hold recently written series data
* Has max size and a flushing limit
* When the flushing limit is crossed, a snapshot is taken and a compaction process for the related WAL segments is commenced.
* If a write comes in, the cache is too full, and the previous snapshot is still being compacted, the write will fail.
*保留最近写的系列数据
*具有最大尺寸和flush限制
*当超过冲洗限制时，将拍摄快照并开始相关WAL段的压缩过程。
*如果写入，缓存太满，前一个快照仍在压缩，写入将失败。

# Engine

* Maintains references to Cache, FileStore, WAL, etc..
* Creates a cursor
* Receives writes, coordinates queries
* Hides underlying files and types from clients
*维护对Cache，FileStore，WAL等的引用。
*创建一个游标
*接收写入，协调查询
*隐藏客户端的底层文件和类型


## Cursor

* Iterates forward or reverse for given key
* Requests values from Engine for key and timestamp
* Has no knowledge of TSM files or WAL - delegates to Engine to request next set of Values
*迭代给定键的前进或后退
*请求Engine的值以获取密钥和时间戳
*不了解TSM文件或WAL - 委托Engine代表请求下一组值


## FileStore
* Manages TSM files
* Maintains the file indexes and references to active files
* A TSM file that is opened entails reading in and adding the index section to the `FileIndex`.  The block data is then MMAPed up to the index offset to avoid having the index in memory twice.
*管理TSM文件
*维护文件索引和对活动文件的引用
*打开的TSM文件需要读入并将索引部分添加到`FileIndex`。 然后将块数据进行MMAP，直到索引偏移，以避免将索引存储在内存中两次。

## FileIndex
* Provides location information to a file and block for a given key and timestamp.
*为给定密钥和时间戳的文件和块提供位置信息。

## Interfaces

```
SeriesIterator returns the key and []Value such that a key is only returned
once and subsequent calls to Next() do not return the same key twice.

SeriesIterator返回键和[]值，以便仅返回键
对Next（）的一次和后续调用不会两次返回相同的键。
type SeriesIterator interface {
   func Next() (key, []Value, error)
}
```

## Types

_NOTE: the actual func names are to illustrate the type of functionality the type is responsible._
实际的func名称用于说明类型负责的功能类型

```
TSMWriter writes a sets of key and Values to a TSM file.
TSMWriter将一组键和值写入TSM文件。
type TSMWriter struct {}
func (t *TSMWriter) Write(key string, values []Value) error {}
func (t *TSMWriter) Close() error
```


```
// WALIterator returns the key and []Values for a set of WAL segment files.
WALIterator返回一组WAL段文件的键和[]值。
type WALIterator struct{
    Files *os.File
}
func (r *WALReader) Next() (key, []Value, error)
```


```
TSMIterator returns the key and values from a TSM file.
TSMIterator从TSM文件返回键和值。
type TSMIterator struct {}
func (r *TSMIterator) Next() (key, []Value, error)
```

```
type Compactor struct {}
func (c *Compactor) Compact(iters ...SeriesIterators) error
```

```
type Engine struct {
    wal *WAL
    cache *Cache
    fileStore *FileStore
    compactor *Compactor
}

func (e *Engine) ValuesBefore(key string, timestamp time.Time) ([]Value, error)
func (e *Engine) ValuesAfter(key string, timestamp time.Time) ([]Value, error)
```

```
type Cursor struct{
    engine *Engine
}
...
```

```
// FileStore maintains references
type FileStore struct {}
func (f *FileStore) ValuesBefore(key string, timestamp time.Time) ([]Value, error)
func (f *FileStore) ValuesAfter(key string, timestamp time.Time) ([]Value, error)

```

```
type FileIndex struct {}

// Returns a file and offset for a block located in the return file that contains the requested key and timestamp.
返回位于返回文件中的块的文件和偏移量，该块包含请求的键和时间戳。
func (f *FileIndex) Location(key, timestamp) (*os.File, uint64, error)
```

```
type Cache struct {}
func (c *Cache) Write(key string, values []Value, checkpoint uint64) error
func (c *Cache) SetCheckpoint(checkpoint uint64) error
func (c *Cache) Cursor(key string) tsdb.Cursor
```

```
type WAL struct {}
func (w *WAL) Write(key string, values []Value)
func (w *WAL) ClosedSegments() ([]*os.File, error)
```


# Concerns关注

## Performance性能

There are three categories of performance this design is concerned with:

* Write Throughput/Latency
* Query Throughput/Latency
* Startup time
* Compaction Throughput/Latency
* Memory Usage
该设计涉及三类性能：

*写入吞吐量/延迟
*查询吞吐量/延迟
* 启动时间
*压实吞吐量/延迟
* 内存使用情况


### Writes

Write throughput is bounded by the time to process the write on the CPU (parsing, sorting, etc..), adding and evicting to the Cache and appending the write to the WAL.  The first two items are CPU bound and can be tuned and optimized if they become a bottleneck.  The WAL write can be tuned such that in the worst case every write requires at least 2 IOPS (write + fsync) or batched so that multiple writes are queued and fsync'd in sizes matching one or more disk blocks.  Performing more work with each IO will improve throughput

Write latency is minimal for the WAL write since there are no seeks.  The latency is bounded by the time to complete any write and fsync calls.
写入吞吐量受到处理CPU上的写入（解析，排序等），添加和逐出高速缓存以及将写入附加到WAL的限制。 前两项是CPU绑定的，如果它们成为瓶颈，可以进行调整和优化。 可以调整WAL写入，使得在最坏的情况下，每次写入需要至少2 IOPS（写入+ fsync）或批处理，以便多个写入排队并且fsync'd的大小与一个或多个磁盘块匹配。 对每个IO执行更多工作将提高吞吐量

WAL写入的写入延迟是最小的，因为没有搜索。 延迟受到完成任何写入和fsync调用的时间的限制。
### Queries

Query throughput is directly related to how many blocks can be read in a period of time.  The index structure contains enough information to determine if one or multiple blocks can be read in a single IO.

Query latency is determine by how long it takes to find and read the relevant blocks.  The in-memory index structure contains the offsets and sizes of all blocks for a key.  This allows every block to be read in 2 IOPS (seek + read) regardless of position, structure or size of file.

查询吞吐量与在一段时间内可以读取的块数直接相关。 索引结构包含足够的信息以确定是否可以在单个IO中读取一个或多个块。

查询延迟取决于查找和读取相关块所需的时间。 内存中索引结构包含密钥的所有块的偏移量和大小。 无论文件的位置，结构或大小如何，这都允许以2 IOPS（搜索+读取）读取每个块。
### Startup

Startup time is proportional to the number of WAL files, TSM files and tombstone files.  WAL files can be read and process in large batches using the WALIterators.  TSM files require reading the index block into memory (5 IOPS/file).  Tombstone files are expected to be small and infrequent and would require approximately 2 IOPS/file.
启动时间与WAL文件，TSM文件和逻辑删除文件的数量成正比。 可以使用WALIterators以大批量读取和处理WAL文件。 TSM文件需要将索引块读入内存（5 IOPS /文件）。 墓碑文件预计很小且不常见，并且需要大约2 IOPS /文件。

### Compactions

Compactions are IO intensive in that they may need to read multiple, large TSM files to rewrite them.  The throughput of a compactions (MB/s) as well as the latency for each compaction is important to keep consistent even as data sizes grow.

To address these concerns, compactions prioritize old WAL files over optimizing storage/compression to avoid data being hidden during overload situations.  This also accounts for the fact that shards will eventually become cold for writes so that existing data will be able to be optimized.  To maintain consistent performance, the number of each type of file processed as well as the size of each file processed is bounded.
压缩是IO密集型的，因为它们可能需要读取多个大型TSM文件来重写它们。 压缩的吞吐量（MB / s）以及每次压缩的延迟对于在数据大小增长时保持一致非常重要。

为了解决这些问题，压缩优先考虑旧的WAL文件而不是优化存储/压缩，以避免在过载情况下隐藏数据。 这也解释了碎片最终会变冷的事实，以便能够优化现有数据。 为了保持一致的性能，处理的每种文件类型的数量以及处理的每个文件的大小都是有界的。

### Memory Footprint

The memory footprint should not grow unbounded due to additional files or series keys of large sizes or numbers.  Some options for addressing this concern is covered in the [Design Options] section.
由于额外的文件或大尺寸或数字的系列密钥，内存占用不应该无限制。 有关解决此问题的一些选项，请参见[设计选项]部分。

## Concurrency

The main concern with concurrency is that reads and writes should not block each other.  Writes add entries to the Cache and append entries to the WAL.  During queries, the contention points will be the Cache and existing TSM files.  Since the Cache and TSM file data is only accessed through the engine by the cursors, several strategies can be used to improve concurrency.

1. cached series data is returned to cursors as a copy.  Since cache snapshots are released following compaction, cursor iteration and writes to the same series could block each other.  Iterating over copies of the values can relieve some of this contention.
2. TSM data values returned by the engine are new references to Values and not access to the actual TSM files.  This means that the `Engine`, through the `FileStore` can limit contention.
3. Compactions are the only place where new TSM files are added and removed.  Since this is a serial, continuously running process, file contention is minimized.
并发性的主要问题是读取和写入不应该相互阻塞。 写入向缓存添加条目并将条目附加到WAL。 在查询期间，争用点将是Cache和现有TSM文件。 由于缓存和TSM文件数据仅由游标通过引擎访问，因此可以使用几种策略来提高并发性。

1.缓存的系列数据作为副本返回到游标。 由于缓存快照是在压缩后释放的，因此游标迭代和对同一系列的写入可能会相互阻塞。 迭代值的副本可以减轻一些争用。
2.引擎返回的TSM数据值是对值的新引用，不访问实际的TSM文件。 这意味着`Engine`，通过`FileStore`可以限制争用。
3.压缩是添加和删除新TSM文件的唯一位置。 由于这是一个连续运行的连续进程，因此文件争用最小化。


## Robustness稳健性

The two robustness concerns considered by this design are writes filling the cache and crash recovery.
这种设计考虑的两个稳健性问题是写入填充缓存和崩溃恢复。

### Cache Exhaustion衰竭

The cache is used to hold the contents of uncompacted WAL segments in memory until such time that the compaction process has had a chance to convert the write-optimised WAL segments into read-optimised TSM files.

The question arises about what to do in the case that the inbound write rate temporarily exceeds the compaction rate. There are four alternatives:

* block the write until the compaction process catches up
* cache the write and hope that the compaction process catches up before memory exhaustion occurs
* evict older cache entries to make room for new writes
* fail the write and propagate the error back to the database client as a form of back pressure

The current design chooses the last option - failing the writes. While this option reduces the apparent robustness of the database API from the perspective of the clients, it does provide a means by which the database can communicate, via back pressure, the need for clients to temporarily backoff. Well behaved clients should respond to write errors either by discarding the write or by retrying the write after a delay in the hope that the compaction process will eventually catch up. The problem with the first two options is that they may exhaust server resources. The problem with the third option is that queries (which don't touch WAL segments) might silently return incomplete results during compaction periods; with the selected option the possibility of incomplete queries is at least flagged by the presence of write errors during periods of degraded compaction performance.
缓存用于在内存中保存未压缩的WAL段的内容，直到压缩过程有机会将写入优化的WAL段转换为读取优化的TSM文件为止。

问题出现在入站写入速率暂时超过压缩率的情况下该怎么做。有四种选择：

*阻止写入，直到压缩过程赶上
*缓存写入并希望压缩过程在内存耗尽发生之前赶上
*逐出旧的缓存条目，为新写入腾出空间
*写入失败并将错误传播回数据库客户端，作为背压的一种形式

当前设计选择最后一个选项 - 写入失败。虽然此选项从客户端的角度降低了数据库API的明显稳健性，但它确实提供了一种方法，通过该方法，数据库可以通过背压来沟通客户暂时退回的需求。表现良好的客户端应该通过丢弃写入或者在延迟之后重试写入来响应写入错误，希望压缩过程最终能够赶上。前两个选项的问题是它们可能耗尽服务器资源。第三个选项的问题是查询（不接触WAL段）可能会在压缩期间静默返回不完整的结果;对于选定的选项，在压缩性能下降期间，写入错误的存在至少会标记不完整查询的可能性。
### Crash Recovery

Crash recovery is facilitated with the following two properties: the append-only nature of WAL segments and the write-once nature of TSM files. If the server crashes incomplete compactions are discarded and the cache is rebuilt from the discovered WAL segments. Compactions will then resume in the normal way. Similarly, TSM files are immutable once they have been created and registered with the file store. A compaction may replace an existing TSM file, but the replaced file is not removed from the file system until replacement file has been created and synced to disk.
通过以下两个属性促进崩溃恢复：WAL段的仅附加性质和TSM文件的一次写入性质。 如果服务器崩溃，则丢弃不完整的压缩，并从发现的WAL段重建缓存。 然后压缩将以正常方式恢复。 类似地，TSM文件一旦创建并在文件存储中注册就是不可变的。 压缩可以替换现有的TSM文件，但是在创建替换文件并将其同步到磁盘之前，不会从文件系统中删除替换的文件。

#Errata

This section is reserved for errata. In cases where the document is incorrect or inconsistent, such errata will be noted here with the contents of this section taking precedence over text elsewhere in the document in the case of discrepancies. Future full revisions of this document will fold the errata text back into the body of the document.
本节仅供勘误表使用。 如果文件不正确或不一致，此处将记录此类勘误表，如果出现差异，本节内容优先于文件其他部分的文字。 本文档的未来完整修订版将把勘误表文本折叠回文档正文。
#Revisions

##14 February, 2016

* refined description of cache behaviour and robustness to reflect current design based on snapshots. Most references to checkpoints and evictions have been removed. See discussion here - https://goo.gl/L7AzVu
* 精确描述缓存行为和稳健性，以反映基于快照的当前设计。 大多数关于检查站和驱逐的提法已被删除。 请参阅此处的讨论 - https://goo.gl/L7AzVu
##11 November, 2015

* initial design published