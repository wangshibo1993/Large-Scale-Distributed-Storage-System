# 第2章 单机存储系统

*单机存储引擎就是哈希表、B树等数据结构在机械磁盘、SSD等持久化介质上的实现。单机存储系统就是单机存储引擎的一种封装，对外提供文件、键值、表格或者关系模型。
单机存储系统的理论来源于关系数据库。数据库将一个或多个操作组成一组，称作事务，事务具有ACID特性。*

## 2.1 硬件基础

*摩尔定律：每18个月计算机等IT产品的性能会翻一番；或者说相同性能的计算机等IT产品，每18个月价钱会降一半。*

### 2.1.1 CPU架构

从单核单个到多核多个；SMP架构到NUMA架构。

### 2.1.2 IO总线

存储系统的性能瓶颈一般在于IO。

### 2.1.3 网络拓扑

传统的数据中心网络拓扑。三层：接入层（Edge）、汇聚层（Aggregation）、核心层（Core）。

Google于2018年提出扁平化拓扑结构，即三级CLOS网络。

### 2.1.4 性能参数

SSD盘、SAS盘、SATA盘差异。

### 2.1.5 存储层次架构

存储系统的性能主要包括两个维度：吞吐量以及访问延时。保证访问延时的基础上，通过最低成本实现尽可能高的吞吐量。

热数据存储到SSD中，冷数据存储到磁盘中。

## 2.2 单机存储系统

*哈希存储引擎（增、删、改、随机读，不支持顺序扫描）、B树存储引擎（增、删、读、改，支持顺序扫描）、LSM树存储引擎（增、删、读、改，支持顺序扫描）。*

### 2.2.1 哈希存储引擎

*Bitcask是一个基于哈希表结构的键值存储系统，它仅支持追加操作，即所有的写操作只追加而不修改老的数据。该系统中，每个文件有一定的大小限制，当文件达到相应的大小时，就会产生一个新的文件，老的文件只读不写。任意时刻，只有一个文件是可写的，用于数据追加，称为活跃文件。*

![Bitcask文件结构](chapter2-pic/图2.2.1-Bitcask文件结构.png)

1. 数据结构

如图所示Bitcask数据文件是一条一条写入的写入操作，每一条记录的数据项分别为主键（key）、value内容（value）、主键长度（key_sz）、value长度（value_sz）、时间戳（timestamp）以及crc校验值。

![Bitcask每条数据数据结构](chapter2-pic/图2.2.1-Bitcask每条数据数据结构.png)

哈希表结构中每一项包含了三个用于定位数据的信息，分别是文件编号（file_id），value在文件中的位置（value_pos），value长度（value_sz）。

通过读取file_id对应文件的value_pos开始的value_sz个字节，就得到了最终的value值。

写入文件时首先将Key-Value记录追加到活跃数据文件的末尾，接着更新内存哈希表。因此，每个写操作总共需要进行一次顺序的磁盘写入和一次内存操作。

![Bitcask哈希表每条记录结构](chapter2-pic/图2.2.1-Bitcask哈希表每条记录结构.png)

2. 定期合并

Bitcask系统中的记录删除或更新后，原来的记录成为垃圾数据。如果这些数据一直保存下去，文件会无限膨胀下去，为了解决这个问题，Bitcask需要定期执行合并操作实现垃圾回收。

所谓合并操作，即将所有老数据文件中的数据扫描一遍并生成新的数据文件，这里的合并其实就是对同一个key的多个操作以只保留最新一个的原则进行删除，每次合并后，新生成的数据文件就不再有冗余数据了。

3. 快速恢复

Bitcask系统中的哈希索引存储在内存中，如果不做额外的工作，服务器断电重启重建哈希表需要扫描一遍数据文件，如果数据文件很大，这是一个非常耗时的过程。Bitcask通过索引文件来提高重建哈希表的速度。

索引文件就是将内存中的哈希索引表转储到磁盘上生成的结果文件。Bitcask对老数据文件进行合并操作时，会产生新的数据文件，这个过程中还会产生一个索引文件，这个索引文件记录每一条记录的哈希索引信息。

### 2.2.2 B树存储引擎

*相比哈希存储引擎，B树存储引擎不仅支持随机读取，还支持范围扫描。关系数据库中通过索引访问数据，例如MySQL InnoDB，行的数据存于聚集索引中，组织成B+树（B树的一种）数据结构。*

1. 数据结构

B+树的根节点是常驻内存的，B+树一次检索最多需要h-1次磁盘IO，复杂度为
<img src="http://chart.googleapis.com/chart?cht=tx&chl=O(h)=O(log_dN)
" style="border:none;">。

2. 缓冲区管理

缓冲区管理器负责将可用的内存划分成缓冲区，缓冲区是与页面同等大小的区域，磁盘块的内容可以传送到缓冲区中。缓冲区管理器的关键在于替换策略，即选择将哪些页面淘汰出缓冲池。

(1) LRU

LRU算法淘汰最长时间没有读过或者写过的块。这种方法要求缓冲区管理器按照页面最后一次被访问的时间组成一个链表，每次淘汰链表尾部的页面。

(2) LIRS

LRU算法在大多数情况下表现是不错的，但存在一个问题：假如某一个查询做了一次全表扫描，将导致缓冲池中大量页面（可能包含很多很快被访问的热点页面）被替换，从而污染缓冲池。

现代数据库一般采用LIRS算法，将缓冲池分为两级，数据首先进入第一级，如果数据在较短时间被访问两次或者以上，则成为热点数据进入第二级，每一级内部还是采用LRU替换算法。

### 2.2.3 LSM树存储引擎

*LSM树（Log Structured Merge Tree）思想：将对数据的修改增量保持在内存中，达到制定的大小限制后将这些修改操作批量写入磁盘，读取时需要合并磁盘中的历史数据和内存中最近的修改操作。*

*LSM树的优势在于有效地规避了磁盘随机写入问题，但读取时可能需要访问较多的磁盘文件。*

*这里以LevelDB为例子介绍LSM树*

1. 存储结构

LevelDB存储引擎主要包括：内存中的MemTable和不可变MemTable以及磁盘上的几种主要文件：当前文件、清单文件、操作日志文件以及SSTable文件。

当应用写入一条记录时，LevelDB会首先将修改操作写入到操作日志，成功后再将修改操作应用到MemTable。

当MemTable占用的内存达到一个上限值，需要将内存的数据转储到外存文件中（LevelDB会将原来的MemTable冻结成不可变MemTable，并生成一个新的MemTable）。

LevelDB后台线程会将不可变MemTable的数据排序后转储到磁盘，形成一个新的SSTable文件，操作成为Compaction。SSTable文件是内存中的数据不断进行Compaction操作后形成的，且SSTable的所有文件是一种层级结构，第0层为Level0，第一层为Level1，以此类推。

2. 合并

LevelDB写入操作很简单，但读取操作比较复杂，需要在内存以及各级文件中按照从新到老依次查找，代价很高。为了加快速度，LevelDB内部会执行Compaction操作来对已有的记录进行压缩整理，从而删除一些不再有效的记录，减少数据规模和文件数量。

LevelDB的两种Compaction操作：minor compaction和major compaction。

## 2.3 数据模型

*存储系统的数据模型主要包括三类：文件、关系、键值模型。此外还包括关系弱化的表格模型等等。*

### 2.3.1 文件模型

文件系统以目录树的形式组织文件。POSIX（Portable Operating System Interface）是应用程序访问文件系统的API标准，定义了文件系统存储接口及操作集。POSIX主要接口：

* open/close 打开/关闭一个文件，获取文件描述符；
* read/write 读取一个文件或者往文件中写入数据；
* opendir/closedir 打开或关闭一个目录
* readdir 遍历目录。

POSIX不仅定义了文件操作接口，而且还定义了读写操作语义。例如考虑原子性，读操作能够读到之前所有写操作的结果。POSIX适合单机文件系统，出于性能考虑，分布式文件系统一般不会完全遵守这个标准。NFS（Network File System）允许客户端缓存文件数据，多个客户端并发修改同一个文件可能出现不一致的情况。例，NFS客户端A和B同时修改NFS服务器某个文件，A和B都在本地缓存了文件副本，A先提交B后提交，即使A和B修改的是文件的不同位置，也会出现B覆盖A的情况。

对象模型和文件模型比较类似，用于存储图片、视频、文档等二进制数据块，典型的系统包括Amazon Simple Storage，Taobao File System。这些系统弱化了目录树的概念，Amazon Simple Storage只支持一级目录，不支持子目录，Taobao File System甚至不支持目录结构。与文件模型不同的是，对象模型要求对象一次性写入到数据库，只能删除整个对象，不能修改其中某个部分。

### 2.3.2 关系模型

SQL包括两个重要的特性：索引和事务。

### 2.3.3 键值模型

大量的NoSQL采用了键值模型（也称为Key-Value模型）。

Key-Value模型过于简单，支持的应用场景有限，NoSQL系统中使用比较广泛的模型是表格模型。表格模型弱化了关系模型中的多表关联，支持基于多表的简单操作，典型的系统是Google Bigtable以及其开源Java实现的HBase。表格模型除了支持简单的基于主键的操作，还支持范围扫描，另外也支持基于列的操作。主要操作：insert、delete、update、get、scan。

与关系模型不同的是，表格模型一般不支持多表关联操作，Bigtable这样的系统也不支持二级索引，事务操作也比较弱，各个系统支持的功能差异较大，没有统一的标准。另外，表格模型往往还支持无模式（schema-less）特性，也就是说，不需要预先定义每行包括哪些列以及没列的类型，多行间允许包含不同列。

### 2.3.4 SQL与NoSQL

互联网的发展伴随着的是数据规模越来越大，并发量越来越高，传统的关系型数据库有时显得力不从心，非关系型数据库应运而生。

NoSQL系统带来了很多新的理念，比如良好的可扩展性，弱化数据库的设计范式，弱化一致性要求，在一定程度上解决了海量数据和高并发的问题。

关系型数据库在海量数据场景面临如下挑战：

* **事务** 分布式系统中保证ACID非常困难，如两阶段提交（性能很低，且不能容忍服务器故障）。
* **联表** 在海量数据场景中，为了避免数据库多表关联操作，往往会使用数据冗余等违反数据库范式的手段，且这些手段带来的收益远高于成本。
* **性能** 关系数据库采用B树存储引擎，更新操作性能不如LSM树这样的存储引擎。另外如果只有基于主键的增删查改操作，关系数据库的性能也不如专门定制的Key-Value存储系统。

随着数据规模的增大，可扩展性以及性能提升可以带来越来越明显的收益，NoSQL在这些方面可以表现很好。然而，NoSQL也面临如下问题：

* **缺少统一标准** 关系数据库有SQL这样的标准，并拥有完整的生态链。然而NoSQL系统使用方法各不相同，切换成本高，很难通用。
* **使用及运维复杂** NoSQL的选型及使用方式往往需要理解系统的实现。

## 2.4 事务与并发控制

### 2.4.1 事务

事务具有ACID属性。

出于性能考虑，许多数据库允许使用者选择牺牲隔离属性来换取并发度，从而获得性能的提升。

四种隔离级别：

* Read Uncommitted(RU)
* Read Committed(RC)
* Repeatable Read(RR)
* Serializable(S)

隔离级别的降低可能导致读到脏数据或者事务执行异常，例如：

* Lost Update(LU)
* Dirty Reads(DR)
* Non-Repeatable Reads(NRR)
* Second Lost Updates(SLU)
* Phantom Reads(PR)

|LU|DR|NRR|SLU|PR
:-:|:-:|:-:|:-:|:-:
RU|N|Y|Y|Y|Y
RC|N|N|Y|Y|Y
RR|N|N|N|N|Y
S|N|N|N|N|N

### 2.4.2 并发控制

1. 数据库锁

事务可以分为：读事务、写事务、读写混合事务。相应地，锁也分为两种类型：读锁、写锁。其中，写事务阻塞读事务。

2. 写时复制

写时复制（Copy-On_Write，COW）读操作不加锁，极大地提高了读取性能。

写时复制B+树执行写操作的步骤如下：

* 拷贝
* 修改
* 提交

如果读操作发生在提交前，那么，将读取老节点的数据，否则读取新节点。

3. 多版本并发控制

MVCC，即Multi-Version Concurrency Control，也能够实现读事务不加锁。MVCC对每行数据维护多个版本，无论事务的执行时间有多长，MVCC总能提供与事务开始时刻相一致的数据。

## 2.5 故障恢复

*数据库系统以及其他的分布式存储系统一般采用操作日志（有时也称为提交日志，即Commit Log）技术来实现故障恢复。操作日志分为回滚日志（UNDO log）、重做日志（REDO Log）以及UNDO/REDO日志。如果记录事务修改前的状态，则为回滚日志；如果记录修改后的状态，则为重做日志。*

### 2.5.1 操作日志

为了保证数据库的一致性，数据库操作需要持久化到磁盘，如果每次操作都随机更新磁盘的某个数据块，系统性能将会很差。因此，通过操作日志顺序记录每个数据库操作并在内存中执行这些操作，内存中的数据定期刷新到磁盘，实现将随机写请求转化为顺序写请求。

操作日志记录了事务的操作。例如，事务T对表格中的X执行加10操作，初始时X=5，更新后X=15，那么，UNDO日志记为<T,X,5>，REDO日志记为<T,X,15>，UNDO/REDO日志记为<T,X,5,15>。

### 2.5.2 重做日志

存储系统如果采用REDO日志，其写操作流程如下：

* 将REDO日志以追加的方式写入磁盘的日志文件。
* 将REDO日志的修改操作应用到内存里。
* 返回操作成功或失败。

### 2.5.3 优化手段

1. 成组提交

存储系统要求先将REDO日志刷入磁盘才可以更新内存中的数据，如果每个事务都要求将日志立即刷入磁盘，系统的吞吐量将会很差。因此，存储系统往往有一个是否立即刷入磁盘的选项，对于一致性要求很高的应用，可以设置为立即刷入；相应地，对于一致性要求不太高的应用，可以设置为不要求立即刷入，首先将REDO日志缓存到操作系统或者存储系统的内存缓冲区中，定期刷入磁盘。这种做法有一个问题，如果存储系统意外故障，可能丢失最后一部分更新操作。

成组提交（Group Commit）技术是一种有效的优化手段。REDO日志首先写入到存储系统的日志缓冲区中：

* 日志缓冲区中的数据量超过一定大小，比如512KB；
* 距离上次刷入磁盘超过一定时间，比如10ms。

当满足以上两个条件中的某一个时，将日志缓冲区中的多个事务操作一次性刷入磁盘，接着一次性将多个事务的修改操作应用到内存中并逐个返回客户端操作结果。与定期刷入磁盘不同的是，成组提交技术保证REDO日志成功刷入磁盘后才返回写操作成功，这种做法可能会牺牲写事务的延时，但大大提高了系统的吞吐量。

2. 检查点

如果所有的数据都保存在内存中，那么可能出现两个问题：

* 故障恢复时需要回放所有的REDO日志，效率较低。如果REDO日志较多，比如超过100GB，那么，故障恢复时间是无法接受的。
* 内存不足。即使内存足够大，存储系统往往也只能够缓存最近较长一段时间的更新操作，很难缓存所有的数据。

因此，需要将内存中的数据定期转储（Dump）到磁盘，这种技术称为checkpoint（检查点）技术。系统定期将内存中的操作以某种易于加载的形式（checkpoint文件）转储到磁盘中，并记录checkpoint时刻的日志回放点，以后故障恢复只需要回放checkpoint时刻的日志回放点之后的REDO日志。

## 2.6 数据压缩

*数据压缩包括有损压缩和无损压缩，有损压缩算法压缩比率高，但数据可能失真，一般用于压缩图片、音频、视频；而无损压缩算法能够完全还原原始数据，这里只讨论无损压缩。*

*早期的数据压缩是基于编码上的优化技术，其中以Huffman编码最为知名，它通过统计字符出现的频率计算最优前缀编码。1977年，以色列人Jacob Ziv和Abraham Lempel发表论文《顺序数据压缩的一个通用算法》，从此，LZ系列压缩算法几乎垄断了通用无损压缩领域。设计压缩算法不仅要考虑压缩比还要考虑压缩算法的执行效率。*

*压缩算法的核心是找重复数据，列式存储技术通过吧相同列的数据组织在一起，不仅减少了大数据分析需要查询的数据量，还大大地提高了数据的压缩比。传统的OLAP（Online Analytical Processing）数据库，如Sybase IQ、Teradata，以及Bigtable、HBase等分布式表格系统都实现了列式存储。*

### 2.6.1 压缩算法

*压缩算法没有通用的做法，需要根据数据的特点选择或者自己开发合适的算法。压缩的本质就是找数据的重复或者规律，用尽量少的字节表示。*

*Huffman编码是一种基于编码的优化技术，通过统计字符出现的频率来计算最优前缀编码。*

*LZ系列算法一般有一个窗口的概念，在窗口内部找重复并维护数据字典。*

1. Huffman编码

前缀编码要求一个字符的编码不能是另一个字符的前缀。Huffman编码需要解决的问题是，如何找出一种前缀编码方式，使得编码长度最短。

2. LZ系列压缩算法

LZ系列压缩算法是基于字典的压缩算法。LZ系列的算法是一种动态创建字典的方法，压缩过程中动态创建字典并保存在压缩信息里面。

3. BMDiff与Zippy

Google的Bigtable系统中，设计了BMDiff和Zippy（也称为Snappy）两种压缩算法，均属于LZ系列，相比于传统的LZW或者Gzip，这两种算法的压缩比例并不算高，但处理速度非常快。

### 2.6.2 列式存储

传统的行式数据库将一个个完整的数据行存储在数据页中。如果处理查询时需要用到大部分的数据列，这种方式在磁盘IO上是比较高效的。一般来说，OLTP（Online Transaction Processing，联机事务处理）应用适合采用这种方式。

一个OLTP查询可能需要访问大量数据行，且该查询往往只关心少数几个数据列。

![行式存储和列式存储](chapter2-pic/图2.6.2-行式存储和列式存储.png)

很多列式数据库还支持列组（column group，Bigtable系统中称为locality group），即将多个经常一起访问的数据列的各个值存放在一起。

由于同一个数据列的数据重复度很高，因此，列式数据库压缩时有很大的优势。比如，性别列只有两个值，“男”和“女”，可以对这一列建立位图索引。

![行式存储和列式存储](chapter2-pic/图2.6.2-位图索引示意图.png)
