# File Structure
TSM文件由四个部分组成：Header，块，索引和Footer。

```
┌────────┬────────────────────────────────────┬─────────────┬──────────────┐
│ Header │               Blocks               │    Index    │    Footer    │
│5 bytes │              N bytes               │   N bytes   │   4 bytes    │
└────────┴────────────────────────────────────┴─────────────┴──────────────┘
```
标题由一个Magic和version组成，用于标识文件类型和版本号。

```
┌───────────────────┐
│      Header       │
├─────────┬─────────┤
│  Magic  │ Version │
│ 4 bytes │ 1 byte  │
└─────────┴─────────┘
```

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

块之后是文件中块的索引。 索引由一系列索引条目组成，这些索引条目按键按字典顺序排序，然后按时间排序。 每个索引条目都以密钥长度和密钥开头，后跟文件中块数的计数。 每个块条目由块的最小和最大时间，块所在文件的偏移量以及块的大小组成。

索引结构可以提供对所有块的有效访问，以及确定与访问给定密钥相关的成本的能力。 给定密钥和时间戳，我们确切地知道哪个文件包含该时间戳的块以及该块所在的位置以及要检索块的数据量。 如果我们知道我们需要读取文件中的所有或多个块，我们可以使用该大小来确定给定IO中的读取量。

_TBD: 存储在块数据中的块长度可能会被丢弃，因为我们将其存储在索引中._

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

文件系统按每个分片组织一个目录，其中每个分片是整数。 与每个分片目录相关联，还有一组其他目录和文件：

* wal目录 - 包含一组数字增加的文件WAL段文件名为#####.wal。 wal目录与包含TSM文件的目录分开，以便在必要时可以使用不同的类型。
* .tsm文件 - 一组包含压缩系列数据的数值增加的TSM文件。
* .tombstone文件 - 以相应的TSM文件命名为#####.tombstone的文件。 这些包含已删除的度量和系列键。 压缩期间会删除这些文件。

# Data Flow
写入被附加到当前WAL段，并且还被添加到高速缓存中。 每个WAL段都有大小限制，并在填满之后翻转到新文件。 缓存也是大小有限的; 当缓存变得太满时，将执行快照并启动WAL压缩。 如果入站写入速率超过WAL压缩率持续一段时间，则缓存可能变得太满，在这种情况下，新写入将失败，直到压缩过程赶上。 WAL和Cache是独立的实体，不互相交互。 引擎协调对两者的写入。

当WAL段填满并关闭时，压缩器读取WAL条目并将它们与一个或多个现有TSM文件组合。 此过程持续运行，直到所有WAL文件都被压缩并且存在最少数量的TSM文件。 在完成每个TSM文件时，它将由FileStore加载和引用。

通过构造键的游标来执行查询。 游标迭代了一堆值。 当前值耗尽时，Cursor从Engine请求下一组值。 Engine通过查询FileStore和Cache返回一片值。 Cache中的值覆盖在FileStore返回的值之上。 FileStore根据文件的索引读取和解码值块。

更新（为已存在的点写入更新的值）作为正常写入发生。 由于缓存值会覆盖现有值，因此较新的写入优先。

通过将测量或系列的删除条目写入WAL然后更新Cache和FileStore来进行删除。 缓存驱逐所有相关条目。 FileStore为包含相关数据的每个TSM文件写入一个逻辑删除文件。 这些逻辑删除文件在启动时用于忽略块以及在压缩期间删除已删除的条目。

# Compactions

压缩是一个连续且连续运行的过程，可以迭代地优化查询的存储。 具体来说，它执行以下操作：
*将关闭的WAL文件转换为TSM文件并删除已关闭的WAL文件
*将较小的TSM文件合并为较大的TSM文件以提高压缩率
*重写包含已删除系列数据的现有文件
*重写包含具有更新数据的写入的现有文件，以确保只有一个TSM文件中存在一个点。


压缩算法持续运行，并始终根据优先级选择要压缩的文件。
1.如果存在关闭的WAL文件，则将5个最旧的WAL段添加到压缩文件集中。
2.如果任何TSM文件包含WAL文件中也存在较旧时间戳的点，那么这些TSM文件将添加到压缩集中。
3.如果任何TSM文件具有逻辑删除标记，则将这些TSM文件添加到压缩集中。


压缩算法生成一组SeriesIterators，它返回一系列`key`，`Values`，其中返回的每个`key`按字典顺序大于前一个。 对迭代器进行排序，使得WAL迭代器将覆盖TSM文件迭代器返回的任何值。 WAL迭代器读取并缓存WAL段，以便可以正确处理日志中的后续删除。 TSM文件迭代器使用逻辑删除文件来确保在迭代期间不返回已删除的系列。 在处理每个密钥时，将对值切片进行增长，排序，然后将其写入新TSM文件中的新块。 可以基于块的数量或块的大小来划分块。 如果当前TSM文件的总大小超过最大文件大小，则会创建一个新文件。

在写入新文件时可能会发生删除。 由于新的TSM文件不完整，因此不会为其编写逻辑删除。 这可能导致删除的值被写入新文件。 为了防止这种情况，如果正在运行压缩并且发生删除，则中止当前压缩并启动新压缩。

当处理了当前压缩中的所有WAL文件并且已成功写入新的TSM文件时，新的TSM文件将重命名为其最终名称，WAL段将被截断，相关的快照将从缓存中释放。
然后压缩过程再次运行，直到没有更多的WAL文件，并且存在最小文件大小的最小TSM文件数。

# WAL

目前，每个分片都有一个WAL。 这意味着WAL段中的所有写入都是针对给定的分片。 这也意味着跨越大量分片的写入会附加到许多文件，这可能会导致更多的磁盘IO，因为它们会寻找多个文件的末尾。
正在考虑两种选择：

## WAL per Shard

这是WAL的当前行为。 此选项在概念上更容易推理。 例如，在多个WAL段中读取的压缩确保所有WAL条目都与当前分片相关。 如果它完成压缩，则可以安全地删除WAL段。 处理分片删除也更容易，因为所有WAL段都可以与其他分片文件一起删除。

此选项的缺点是，在存在多个分片并写入许多不同分片时，可能会将顺序写入IO转换为随机IO。

## Single WAL

使用单个WAL会增加压缩和删除的复杂性。 压缩将需要先通过分片对段中的所有WAL条目进行排序，然后在每个分片上运行压缩，或者压缩器需要能够同时压缩多个分片，同时确保不同分片中现有TSM文件中的点保持分离。

删除将无法立即回收WAL段，如每个分片有一个WAL的情况。 类似地，需要删除包含已删除分片的写入的WAL段的压缩。

目前，我们正在向单一WAL实施迈进。

# Cache

缓存的目的是使WAL中的数据可查询。 每次将一个点写入WAL段时，它也会写入内存缓存。 缓存分为两部分：表示最近写入的“热”部分和包含活动WAL压缩的快照的“冷”部分
过程正在进行中。

查询对从缓存和已完成的TSM文件读取的值感到满意。 缓存中的点始终优先于具有相同时间戳的TSM文件中的点。 永远不会直接从WAL段文件中读取查询，这些文件旨在优化写入而不是读取性能。

缓存在“point-calculated”的基础上跟踪其大小。 “point-calculated”意味着一个点的RAM存储空间是通过调用它的`Size（）`方法来确定的。 虽然这不直接对应于高速缓存中的实际RAM占用量，但是为了控制RAM使用，这两个值充分良好地相关。

如果高速缓存变得太满，或者高速缓存空闲时间过长，则会获取高速缓存的快照，并为相关的WAL段启动压缩过程。 完成这些段的压缩后，将从缓存中释放相关的快照。

In cases where IO performance of the compaction process falls behind the incoming write rate, it is possible that writes might arrive at the cache while the cache is both too full and the compaction of the previous snapshot is still in progress. In this case, the cache will reject the write, causing the write to fail.
Well behaved clients should interpret write failures as back pressure and should either discard the write or back off and retry the write after a delay.
如果压缩过程的IO性能落后于传入的写入速率，则当缓存太满并且前一个快照的压缩仍在进行时，写入可能会到达缓存。 在这种情况下，缓存将拒绝写入，从而导致写入失败。
表现良好的客户端应将写入故障解释为背压，并应丢弃写入或退出，并在延迟后重试写入。

# TSM File Index

每个TSM文件都包含文件中包含的块的完整索引。 现有索引结构旨在允许跨索引进行二进制搜索以查找键的起始块。 然后，我们将寻找该开始键并顺序扫描每个块以找到时间戳的位置。

现有结构的一些问题是，为密钥寻找给定的时间戳具有未知的成本。 这可能导致读取性能的变化，这很难修复。 另一个问题是加载TSM文件的启动时间将与磁盘上TSM文件的数量和大小成比例增长，因为我们需要扫描整个文件以查找文件中包含的所有密钥。 这可以通过使用单独的索引（如文件）或更改索引结构来解决。

我们选择更新块索引结构以确保TSM文件是完全独立的，支持顺序和随机访问的一致IO特性，并且无论文件大小如何都提供有效的加载时间。 这些变化的含义是索引略大，我们需要能够搜索索引，尽管每个条目的大小都是可变的。

以下是一些替代设计选项，用于处理索引太大而无法容纳在内存中的情况。 我们目前正计划对加载的TSM文件使用间接MMAP索引方法。

### Indirect MMAP Indexing

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

我们将构建一个“偏移”切片，其中每个元素指向索引切片中第一个键的字节位置。

```
┌────────────────────────────────────────────────────────────────────┐
│                              Offsets                               │
├────┬────┬────┬─────────────────────────────────────────────────────┘
│ 0  │ 62 │145 │
└────┴────┴────┘
 ```
使用此偏移切片，我们可以通过在偏移切片上进行二分搜索来找到“密钥2”。 我们不是比较偏移量中的值（例如“62”），而是将其用作底层索引的索引，以检索位置“62”处的键并执行与之比较。

当我们在索引中识别出给定键的正确位置时，我们可以执行另一个二进制搜索或线性扫描。 这应该很快，因为每个索引条目是28个字节并且在内存中都是连续的。

偏移切片的大小将与唯一系列的数量成比例。 如果我们将文件大小限制为4GB，我们将为每个指针使用4个字节。
### LRU/Lazy Load

第二个选项可能是使索引作为内存有界，延迟加载样式缓存。 当发生高速缓存未命中时，扫描索引结构以找到密钥，并且条目被加载并添加到高速缓存中，这导致最近最少使用的条目被逐出。

### Key Compression

另一种选择是使用密钥特定字典编码来压缩密钥。 例如，

```
cpu,host=server1 value=1
cpu,host=server2 value=2
memory,host=server1 value=3
```

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

使用此编码字典，字符串键可以转换为整数序列：

```
cpu,host=server1 value=1 -->    1,1,1,1
cpu,host=server2 value=2 -->    1,1,2,1
memory,host=server1 value=3 --> 3,1,2,1
```

然后可以使用诸如Simple9或Simple8b的比特打包格式进一步压缩这些小整数列表序列。 得到的字节切片将是4或8字节的倍数（分别使用Simple9 / Simple8b），可用作（字符串）。

### Separate Index


另一个选择可能是有一个单独的索引文件（BoltDB）作为`FileIndex`的存储并且是瞬态的。 此索引将在启动时重新创建，并在压缩时更新。

# Components

这些是一些高级组件及其职责。 这些是初步的想法。

## WAL

*仅附加日志由固定大小的段文件组成。
*写入附加到当前段
*填写当前分段后翻转到新分段
*封闭的段永远不会被修改并用于启动和恢复以及压缩。
*商店只有一个WAL，而不是每个分片的WAL。


## Compactor
*连续运行，迭代文件存储优化器
*关闭WAL文件，现有TSM文件并组合成一个或多个新TSM文件

## Cache
*保留最近写的系列数据
*具有最大尺寸和flush限制
*当超过冲洗限制时，将拍摄快照并开始相关WAL段的压缩过程。
*如果写入，缓存太满，前一个快照仍在压缩，写入将失败。

# Engine
*维护对Cache，FileStore，WAL等的引用。
*创建一个游标
*接收写入，协调查询
*隐藏客户端的底层文件和类型

## Cursor

*迭代给定键的前进或后退
*请求Engine的值以获取密钥和时间戳
*不了解TSM文件或WAL - 委托Engine代表请求下一组值

## FileStore
*管理TSM文件
*维护文件索引和对活动文件的引用
*打开的TSM文件需要读入并将索引部分添加到`FileIndex`。 然后将块数据进行MMAP，直到索引偏移，以避免将索引存储在内存中两次。

## FileIndex
*为给定密钥和时间戳的文件和块提供位置信息。

## Interfaces

```
SeriesIterator返回键和[]值，以便仅返回键
对Next（）的一次和后续调用不会两次返回相同的键。
type SeriesIterator interface {
   func Next() (key, []Value, error)
}
```

## Types

_NOTE: 实际的func名称用于说明类型负责的功能类型._

```
TSMWriter将一组键和值写入TSM文件。
type TSMWriter struct {}
func (t *TSMWriter) Write(key string, values []Value) error {}
func (t *TSMWriter) Close() error
```


```
// WALIterator返回一组WAL段文件的键和[]值。
type WALIterator struct{
    Files *os.File
}
func (r *WALReader) Next() (key, []Value, error)
```


```
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

## 性能

该设计涉及三类性能：

*写入吞吐量/延迟
*查询吞吐量/延迟
* 启动时间
*压实吞吐量/延迟
* 内存使用情况

### Writes

写入吞吐量受到处理CPU上的写入（解析，排序等），添加和逐出高速缓存以及将写入附加到WAL的限制。 前两项是CPU绑定的，如果它们成为瓶颈，可以进行调整和优化。 可以调整WAL写入，使得在最坏的情况下，每次写入需要至少2 IOPS（写入+ fsync）或批处理，以便多个写入排队并且fsync'd的大小与一个或多个磁盘块匹配。 对每个IO执行更多工作将提高吞吐量

WAL写入的写入延迟是最小的，因为没有搜索。 延迟受到完成任何写入和fsync调用的时间的限制。

### Queries

查询吞吐量与在一段时间内可以读取的块数直接相关。 索引结构包含足够的信息以确定是否可以在单个IO中读取一个或多个块。

查询延迟取决于查找和读取相关块所需的时间。 内存中索引结构包含密钥的所有块的偏移量和大小。 无论文件的位置，结构或大小如何，这都允许以2 IOPS（搜索+读取）读取每个块。

### Startup

启动时间与WAL文件，TSM文件和逻辑删除文件的数量成正比。 可以使用WALIterators以大批量读取和处理WAL文件。 TSM文件需要将索引块读入内存（5 IOPS /文件）。 墓碑文件预计很小且不常见，并且需要大约2 IOPS /文件。

### Compactions

压缩是IO密集型的，因为它们可能需要读取多个大型TSM文件来重写它们。 压缩的吞吐量（MB / s）以及每次压缩的延迟对于在数据大小增长时保持一致非常重要。

为了解决这些问题，压缩优先考虑旧的WAL文件而不是优化存储/压缩，以避免在过载情况下隐藏数据。 这也解释了碎片最终会变冷的事实，以便能够优化现有数据。 为了保持一致的性能，处理的每种文件类型的数量以及处理的每个文件的大小都是有界的。

### Memory Footprint

The memory footprint should not grow unbounded due to additional files or series keys of large sizes or numbers.  Some options for addressing this concern is covered in the [Design Options] section.

## Concurrency

并发性的主要问题是读取和写入不应该相互阻塞。 写入向缓存添加条目并将条目附加到WAL。 在查询期间，争用点将是Cache和现有TSM文件。 由于缓存和TSM文件数据仅由游标通过引擎访问，因此可以使用几种策略来提高并发性。

1.缓存的系列数据作为副本返回到游标。 由于缓存快照是在压缩后释放的，因此游标迭代和对同一系列的写入可能会相互阻塞。 迭代值的副本可以减轻一些争用。
2.引擎返回的TSM数据值是对值的新引用，不访问实际的TSM文件。 这意味着`Engine`，通过`FileStore`可以限制争用。
3.压缩是添加和删除新TSM文件的唯一位置。 由于这是一个连续运行的连续进程，因此文件争用最小化。

## Robustness稳健性

这种设计考虑的两个稳健性问题是写入填充缓存和崩溃恢复。

### Cache Exhaustion衰竭
缓存用于在内存中保存未压缩的WAL段的内容，直到压缩过程有机会将写入优化的WAL段转换为读取优化的TSM文件为止。

问题出现在入站写入速率暂时超过压缩率的情况下该怎么做。有四种选择：

*阻止写入，直到压缩过程赶上
*缓存写入并希望压缩过程在内存耗尽发生之前赶上
*逐出旧的缓存条目，为新写入腾出空间
*写入失败并将错误传播回数据库客户端，作为背压的一种形式

当前设计选择最后一个选项 - 写入失败。虽然此选项从客户端的角度降低了数据库API的明显稳健性，但它确实提供了一种方法，通过该方法，数据库可以通过背压来沟通客户暂时退回的需求。表现良好的客户端应该通过丢弃写入或者在延迟之后重试写入来响应写入错误，希望压缩过程最终能够赶上。前两个选项的问题是它们可能耗尽服务器资源。第三个选项的问题是查询（不接触WAL段）可能会在压缩期间静默返回不完整的结果;对于选定的选项，在压缩性能下降期间，写入错误的存在至少会标记不完整查询的可能性。


### Crash Recovery

通过以下两个属性促进崩溃恢复：WAL段的仅附加性质和TSM文件的一次写入性质。 如果服务器崩溃，则丢弃不完整的压缩，并从发现的WAL段重建缓存。 然后压缩将以正常方式恢复。 类似地，TSM文件一旦创建并在文件存储中注册就是不可变的。 压缩可以替换现有的TSM文件，但是在创建替换文件并将其同步到磁盘之前，不会从文件系统中删除替换的文件。

#Errata

本节仅供勘误表使用。 如果文件不正确或不一致，此处将记录此类勘误表，如果出现差异，本节内容优先于文件其他部分的文字。 本文档的未来完整修订版将把勘误表文本折叠回文档正文。

#Revisions

##14 February, 2016

* 精确描述缓存行为和稳健性，以反映基于快照的当前设计。 大多数关于检查站和驱逐的提法已被删除。 请参阅此处的讨论 - https://goo.gl/L7AzVu

##11 November, 2015

* initial design published