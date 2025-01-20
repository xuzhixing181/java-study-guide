执行如下的update语句，MySQL的处理流程是怎样的

```sq
update t_user set name = 'xuzhix' where id = 1;
```

- 更新语句的执行流程和查询语句的执行流程大致一样的：
  - 客户端通过连接器来**建立连接**，连接器自会**判断用户身份**
  - 由于执行的是update语句，故不需要经过查询缓存，但会清空查询缓存（MySQL8.0已移除查询缓存的功能）
  - **解析器**会通过**词法分析识别出关键字**update，表名等，**构建出语法树**，再进行**语法分析**，判断输入的语句是否符合MySQL语法
  - 预处理器会判断表和字段是否存在
  - **优化器确定执行计划（选择索引等）**，where条件中的id是主键索引，故决定要使用id这个索引
  - 执行器负责具体执行，找到对应行记录，其后执行更新语句

- 更新语句的流程会涉及undo log（回滚日志）、redo log（重做日志）和binlog（归档日志）
  - **undo log（回滚日志）**：InnoDB存储引擎层生成的日志，保证了**事务中的原子性**，主要用于**事务回滚和MVCC** 
  - **redo log（重做日志）**：InnoDB存储引擎层生成的日志，保证了**事务中的持久性**，主要用于**掉电等故障恢复**
  - **binlog（归档日志）**：Server层生成的日志，主要用于**数据备份和主从复制**

# 为什么需要undo log？

## 事务回滚

- 在每次执行事务过程中，都记录下回滚时 需要的信息到一个日志中，在事务执行中途发生了MySQL崩溃后，就不必再担心无法回滚到事务之前的数据    =》**通过 undo log可回滚到事务提交之前的数据**

- **undo log**是用于**撤销回退的日志**，在事务没提交前，MySQL会先记录更新的数据到undo log日志文件中；当事务需要回滚时，可利用undo log来回滚

  **每当InnoDB引擎操作一条记录（增删改）时，要将回滚时需要的信息都记录到undo log中**

- 在发生回滚时，就**读取undo log**中的数据，再做**原先的相反操作**，如：
  - 在插入一条记录时，将这条记录的主键值记下来，此后回滚时只需要将该主键值 对应的记录删除即可
    在删除一条记录时，将这条记录中的内容都记下来，之后回滚时再把这些内容组成的记录插入到表中即可

- 针对delete操作和update操作，会有一些特殊的处理：
  - delete操作实际上不会立即删除，而是将**要删除的数据记录打上删除标识**，最终的删除操作是由purge线程完成的
  - update操作实际上分为两种情况，update的列是否为主键列作为判断标准：
    - 如果是主键列，则先删除该记录，再插入一行对应记录
    - 如果不是主键列，在**undo log中反向记录是如何update**，便于**回滚时直接执行该update操作来恢复数据**

- 行记录的每次更新操作产生的undo log格式都有`roll_pointer`指针 和 `trx_id`事务id：
  - 通过**roll_pointer指针**可以将这些**undo log串连成一个链表**，该链表被成为**版本链**
  - 通过trx_id 可得知该记录是被哪个事务修改的

## MVCC

- undo log除了可以保证事务回滚之外，还需要通过Read View + undo log实现 MVCC（多版本并发控制）

- 【读已提交】和【可重复读】隔离级别的事务的快照读（普通select语句）是通过Read View + undo log来实现的，其区别在于创建Read View的时机不同：

  - **读已提交**：**每次select都会生成一个新Read View**，在事务期间多次读取同一条数据，前后两次读取到的数据可能会不一致（可能会有其他事务修改了该记录，并提交了事务）
  - **可重复读**：**启动事务时生成一个ReadView**，在整个事务期间都会复用该ReadView，如此可保证在事务期间读取到的数据都是事务启动前的记录

  这两个隔离级别的实现是通过 `事务 Read View中的字段` 和 `记录中的两个隐藏列（trx_id 和 roll_pointer）`的比对，如果不满足可见性，就会顺着undo log版本链中找到满足可见性的记录，从而控制并发事务 访问同一条记录时的行为，此即为MVCC（多版本并发控制）

  

undo log两大作用：

- 实现事务回滚，保证事务的原子性，在事务处理过程中，如果出现了错误或需要回滚，MySQL可利用undo log中的历史数据将数据恢复到事务开始之前的状态
- MVCC（多版本并发控制）的关键实现之一，MVCC是通过 Read View + undo log实现的，undo log会为每条记录保存多份历史数据，MySQL在执行快照读（普通select）时，会根据事务 Read View中的信息，顺着undo log的版本链找到满足其可见性的记录

## undo log如何刷盘

- undo log和数据页的刷盘策略，都是通过redo log来保证持久化的
- Buffer Pool中有undo log页，**对undo log页的修改都会记录到redo log**，redo log默认每秒刷盘，以及在提交事务时会刷盘

 # 为什么需要redo log？
- Buffer Pool可提高读写效率，但**Buffer Pool基于内存实现，一旦断电重启，还没来得及落盘的脏页数据就会丢失**
   为防止断电导致数据丢失的问题，**当有一条记录需要更新时，InnoDB引擎就会先更新内存（同时标记为脏页），再将本次对该页的修改以redo log的形式记录下来**，如此更新操作就算完成了

- 后续，InnoDB引擎会在适当的时机 由后台线程将缓存在Buffer Pool的脏页数据刷新到磁盘，这就是WAL（Write-Ahead Logging）技术  =》**WAL，MySQL的写操作会先写日志，其后在合适的时机再写到磁盘**

- **redo log是物理日志，记录了某个数据页所做的修改**，比如：对某某表空间中的 某某数据页的某偏移量的地方做了xxx更新操作，每当执行一个事务，就会产生这样的一条或多条这样的物理日志

- **在事务提交时，只要先将`redo log`持久化到磁盘即可，不需要等到将缓存在Buffer Pool中的脏页数据持久化到磁盘**

  当系统崩溃时，redo log已经持久化，在重启MySQL后，可根据redo log的内容，将所有的数据恢复到最新的状态

小结：

MySQL之所以需要redo log，有如下两个原因：

- 1）实现**事务的持久性**，让MySQL拥有Crash-Safe的能力，能够保证**MySQL在任何时间段内突然崩溃，之前提交的记录在重启后都不会丢失**
- 2）基于WAL技术，将**MySQL的写操作从`随机写`变为`顺序写`**，提升了MySQL写入磁盘的性能

**修改undo页面要记录对应的redo log？**

- 开启事务后，InnoDB层更新记录前，会先记录相应的undo log，如果是更新操作，则需要将被更新列的旧值记下来，
   生成一条undo log，undo log会写入Buffer Pool中的undo 页面
- **在内存修改该undo页面后，也需要记录对应的redo log**，undo log也要实现持久化的保证

 ## undo log 和 redo log的区别
 undo log和 redo log都属于InnoDB引擎的日志，区别如下：

- undo log记录了此次事务`修改前`的数据状态，记录的是**更新之前的数据**，主要用于**事务回滚，以保证事务的原子性**
- redo log记录了此次事务`修改后`的 数据状态，记录的是**更新之后的数据**，主要用于**事务崩溃、掉电等故障恢复，以保证事务的持久性**

如果在事务提交之前执行错误，宕机崩溃不需要通过undo log回滚，因为事务没有提交，事务的数据不会持久化，还是在内存中，宕机崩溃后数据就丢失了；如果事务提交后发生宕机崩溃，重启后会通过redo log恢复事务

基于redo log，再通过WAL技术，InnoDB就可保证即使数据库发生异常，之前已提交的记录都不会丢失，该能力称为crash-safe（崩溃恢复），由此可见，redo log可保证事务的持久性

 ## redo log和数据都要写入到磁盘，为何要多次一举

- **写入redo log的磁盘操作采用追加方式**，是**顺序写**，而**写入数据需要先找到写入位置后，再写入磁盘，磁盘操作是随机写**
   磁盘的`顺序写`比`随机写`更加高效，redo log写入磁盘的开销更小
   WAL技术的优点：使MySQL的写操作从磁盘的`随机写`变为`顺序写`，提升语句的执行性能，这是因为MySQL的写操作并不是立即更新到磁盘上，而是先记录到日志上，
   在合适的时机更新到磁盘上

 ## 产生的redo log是直接写入磁盘吗
- 执行事务过程中，如果直接写入磁盘会产生大量的IO操作，且磁盘的执行速度远远慢于内存
- redo log引入了自己的缓存：redo log buffer，**每当产生一条redo log时，会先写入到redo log buffer，后续再持久化到磁盘**
   redo log buffer默认大小为16MB，可通过**`innodb_log_buffer_size`来动态调整大小，增大其大小可让MySQL处理`大事务`时不必写入磁盘，进而提升写IO性能**

 ## redo log刷盘时机
缓存在redo log buffer中的 redo log还是处在内存中，会在如下几个时机刷盘：
 - MySQL正常关闭时
 - InnoDB的后台线程每隔1秒，会将redo log buffer持久化到磁盘；
 - 当redo log buffer中记录的写入量大于 redo log buffer内存空间的一半时，会触发落盘
 - **每次事务提交时都会将缓存在redo log buffer中的redo log 直接持久化到磁盘**（该策略可由`innodb_flush_log_at_trx_commit`参数控制）



 ## redo log文件写满怎么办
- 默认情况下，InnoDB存储引擎有一个**重做日志文件组**（redo log Group）,重做日志文件组有两个2个redo log文件（ib_logfile0、ib_logfile1）组成
- 在重做日志文件中，每个redo log file的大小是固定且一致的，每个redo log file设置的默认大小为128MB，文件组的数量默认为2，则总共可记录256MB的操作
- redo log文件是以循环写的方式工作，从头开始写，写到末尾就重新回到开头写。InnoDB存储引擎会先写`ib_logfile0`文件，当`ib_logfile0`文件写满时，会切换到`ib_logfile1`文件继续写；当`ib_logfile1`文件写满时，会切换回`ib_logfile0`文件
- **redo log是为了防止Buffer Pool中的脏页丢失而设计的**，随着系统运行，Buffer Pool中的脏页刷新到磁盘后，redo log中对应的记录就没用了，需要擦除这些旧记录，以腾出空间来记录新的更新操作

![image-20250117081618191](C:\Users\xyl\AppData\Roaming\Typora\typora-user-images\image-20250117081618191.png)

- 如上图所示，redo log采用循环写，InnoDB用`write pos`标记`redo log`当前记录写的位置，用`check point`标记当要前擦除的位置
  - ` write pos`和`check point`都是顺时针移动；
  - check point 到 write pos 之间的蓝色部分：待落盘的脏数据页记录
  - write pos 到 check point之间的红色部分：记录新的更新操作；

- 如果`write pos`追上了`check point`，则意味着redo log文件满了，MySQL不能再执行新的更新操作，MySQL会被阻塞（针对并发量大的系统，适当设置redo log的文件大小很重要），其后，会停下来，将Buffer Pool中的脏页刷新到磁盘，标记redo log哪些记录可被擦除，其后会擦除这些旧的redo log，等擦除完旧记录腾出空间后，check point会向后顺时针移动，等MySQL恢复正常运行后，继续执行新的更新操作
- 一次`check point`追赶`write pos`的过程，就是**脏页刷新到磁盘中变成干净页**，再**标记redo log中哪些记录可以被覆盖的过程**               

 # 为什么需要binlog?
- MySQL在完成一条更新操作后，Server层会生成一条binlog，等之后事务提交时，会将该事务执行过程中产生的所有binlog统一写入binlog文件
- **binlog文件会记录所有数据库表结构变更和表数据修改的日志**，不会记录查询类的操作，如：`select`和`show`操作

 ## 为什么有了binlog，还要有redo log？

- 和MySQL发展的时间线有关，最开始MySQL并没有InnoDB引擎，默认引擎为MyISAM，MyISAM没有crash-safe崩溃恢复能力，所以InnoDB使用redo log来实现crash-safe能力，binlog日志只能用于归档

- 而InnoDB是以插件的形式引入MySQL的，binlog没有crash-safe（崩溃恢复）的能力，所以，InnoDB使用redo log来实现crash-safe的能力

 ## redo log和binlog的区别

redo log和binlog的四个区别如下：

- 1）适用对象不同
  - `redo log`是InnoDB 引擎实现的日志
  - ` binlog`是MySQL的Server层实现的日志，所有存储引擎都可使用
- 2）文件格式不同
  - redo log是物理日志，记录的是在某个表空间的某个数据页中某偏移量的地方做了某修改操作
  - binlog默认是逻辑日志，有**三种格式类型，分别为STATEMENT（默认格式）、ROW、MIXED**，区别如如下：
    - `STATEMENT`：**每条修改数据的SQL都会被记录到binlog中**（binlog因此可称为逻辑日志），主从架构中slave端会根据binlog同步数据，但`STATEMENT`有动态函数的问题，now这类执行结果**随时在变的动态函数会导致主从数据库之间数据可能会不一致**
    - ROW：**记录行数据最终被修改后的样子**（row格式的binlog不能称为逻辑日志），不会出现STATEMENT下动态函数的问题，但**ROW的缺点是每行数据的变化结果都会被记录**，比如：执行批量update语句，**更新多少次行数据就会产生多少条日志记录**，可能会使binlog文件过大，而**在STATEMENT格式下只会记录一个update语句**
    - MIXED：包含了STATEMENT和MIXED模式，会根据不同的情况自动使用ROW模式和STATEMENT模式

- 3）写入方式不同：
  - redo log是循环写，日志空间大小固定，全部写满就从头开始，用于暂时保存未被刷入磁盘的脏页日志
  - binlog是追加写，写满一个文件，就创建一个新的文件继续写，不会覆盖以前的日志，保存的是全量日志
- 4）适用场景不同：
  - **redo log用于掉电、事务崩溃等故障恢复**
  - **binlog用于备份恢复、主从复制**
- 5）持久化时机不同
  - redo log可在事务还没提交前持久化到磁盘（执行事务时，redo log会直接写在redo log buffer中，**缓存在redo log buffer中的redo log会被`后台线程`每隔1秒一起持久化到磁盘**）
  - binlog必须在事务提交后，才可以持久化到磁盘

 ## 整个数据库的数据被删除，能用redo log文件来恢复数据吗
 - 不能使用redo log文件来恢复数据，只能使用binlog文件恢复
 - redo log文件是循环写，会边写边擦除日志，只记录未被刷入磁盘的数据的物理日志，已经刷入磁盘的数据都会从redo log文件中擦除
 - binlog文件保存的是全量日志，即保存了所有的数据变更，理论上只要记录在binlog上的数据，都可以恢复     









 # 主从复制如何实现
 - MySQL的主从复制依赖于binlog，即记录在MySQL上的所有变化都以二进制的形式保存在磁盘上
   主从复制的过程，即是将binlog中的数据从 主库传输到从库上
 - 主从复制的过程是异步的，即主库上执行事务操作的线程不会等待复制binlog的线程同步完成
 - MySQL集群的主从复制过程可分为如下三个部分：
    - 1）写入binlog：主库写binlog日志，提交事务，并更新本地存储数据
    - 2）同步binlog：把binlog复制到所有从库上，每个从库会将binlog写入到暂存日志中
    - 3）回放binlog：回放binlog，并更新存储引擎中的数据
  - 在完成主从复制后，可以在写数据时只写主库，在读数据时只读从库，即便写请求会锁表 或 锁记录，也不会影响读请求的执行

## 从库是不是越多越好？
从库并非越多越好

- 因为**从库数量增加，从库连接上来的IO线程会较多**，主库要创建**同样多的log dump线程来处理主从复制的请求**，对主库资源消耗较多，同时还会受限于主库的网络带宽
- 所以在实际使用中，一个主库一般会有2~3个从库（一主两从一备主），这就是一主多从的MySQL集群架构

## MySQL主从复制有哪些模型？
### 同步复制
- 同步复制：MySQL主库提交事务的线程要等待所有从库复制成功响应后，才返回给客户端结果
- 同步复制在实际项目中不推荐使用，原因如下：
     - 1）性能很差，**要复制到所有从节点才能返回响应**
     - 2）可用性很差，**主库和所有从库任何一个实例出问题，都会影响业务**
### 异步复制（默认模型）
- **异步复制：MySQL主库提交事务的线程并不会等待binlog同步到各从库，就返回给客户端**
- 异步复制模式中，一旦主库宕机，数据就会发生丢失
### 半同步复制
- 半同步复制：MySQL5.7之后新增的一种复制方式，**事务线程**不必再等待所有从库复制成功后响应，**只要一部分从库复制成功就可响应**，如：一主二从的集群，**只要数据成功复制到任意一个从库上，主库的事务线程就可返回给客户端**
- **半同步复制的方式，兼顾了同步复制和异步复制的优点，即便主库出现宕机，至少还有一个从库有最新的数据可以切换为主库**，不存在数据丢失的风险

## binlog的刷盘时机
- 事务执行过程中，要先将日志写到binlog cache（Server层的Cache），等事务提交时，再将binlog cache写入到binlog文件中 
- 一个事务的binlog不能被拆开，无论该事务有多大（有多条语句），也要保证一次性写入 
- MySQL通过分配binlog_cache来为每个线程缓冲binlog，参数 binlog_cache_size 用于控制单个线程内 binlog cache所占内存的大小，     如果超过了该参数设置的大小，则会暂存到磁盘 

## binlog cache写入binlog文件的时机 

- 在事务提交时，执行器会将binlog cache中的完整事务写入到binlog文件中，并清空binlog cache 

- 每个线程都会有自己的binlog cache，最终都会写入到同一个binlog文件   

  ![image-20250118101156478](C:\Users\xyl\AppData\Roaming\Typora\typora-user-images\image-20250118101156478.png)

- 上图中的`write`操作：将日志写入到binlog文件，但并没有将数据持久化到磁盘，因为数据还缓存在文件系统的 Page Cache，      **write不涉及磁盘IO，所以写入速度较快**   
- 图中的`fsync`操作：将数据持久化到磁盘的操作，会涉及磁盘IO，所以**频繁地fsync会导致磁盘IO升高** 

- MySQL提供了一个sync_binlog参数来控制数据库的binlog刷到磁盘的频率:   

  - sync_binlog = 0，每次提交事务都只write，不fsync，后续交由操作系统决定何时将数据持久化到磁盘   
  - sync_binlog = 1，每次提交事务都会write，再立即执行fsync   
  - sync_binlog = N（N>1），**每次提交事务都write，但积累N个事务后才会fsync** 

- 在MySQL中默认sync_binlog = 0，即不做任何强制性的刷盘操作，此时性能最好，但风险最大， 因为主机一旦发生异常重启，还没持久化到磁盘的数据就会丢失

- 当`sync_binlog`设置为1时，最安全但性能损耗最大，即使主机发生异常重启，最多只丢失一个事务的binlog，而持久化到磁盘的数据则不会有影响

- 如果允许少量事务的binlog日志丢失的风险，为提高写入性能，`sync_binlog`一般会设置为100~1000中的某个数值

  

# 为什么需要两阶段提交
- 事务提交后，redo log和binlog都要持久化到磁盘，两个独立的逻辑可能会出现半成功的状态，会造成两份日志间的逻辑不一致

- update语句在持久化redo log和binlog两个日志的过程中，可能会出现了半成功的状态，可能存在的情况如下：

  - 1）如果在将redo log刷入到磁盘后，MySQL突然宕机了，而binlog还没来得及写入，MySQL重启后，会通过redo log将Buffer Pool中的行数据恢复到最新值，但binlog中没有记录这条更新语句，在主从架构中，binlog会被复制到从库，由于binlog字段是旧值，与主库的值不一致
  - 2）如果在将binlog刷入到磁盘后，MySQL突然宕机了，而redo log还没来得及写入。由于redo log还没写，崩溃恢复后该事务无效，而binlog中已记录了该条更新语句，在主从架构中，binlog会被复制到从库，从库会执行该更新语句，如此会导致该行记录在主从库中数据不一致

- 在持久化redo log和 binlog两份日志时，如果出现半成功状态，就会造成主从库的数据不一致，因为**redo log影响主库的数据，binlog影响从库的数据**，因此，**redo log和binlog必须保持一致才能保证主从数据的一致性**

- **MySQL为避免出现两份日志间的逻辑不一致的问题，引入了两阶段提交来解决**
  两阶段提交：分布式事务一致性协议，可保证多个逻辑操作要么不全部成功，要么全部失败，不会出现半成功状态

- 两阶段提交：将单个事务的提交拆分成了两个阶段，分别是【**准备（Prepare）阶段**】和【**提交（Commit）阶段**】，

  每个阶段都由协调者（Coordinator）和参与者（Participant）共同完成（commit语句提交事务包含commit阶段）

  

## 两阶段提交的过程
- 在MySQL的InnoDB引擎中，开启binlog情况下，MySQL会同时维护binlog日志和InnoDB引擎的redo log日志，为保证这两个日志的一致性，MySQL使用了内部XA事务：**由binlog作为协调者，存储引擎作为参与者**

- 当客户端执行commit语句或在自动提交时，MySQL内部开启了一个XA事务，分两阶段来完成XA事务的提交：

  ![image-20250118104936935](C:\Users\xyl\AppData\Roaming\Typora\typora-user-images\image-20250118104936935.png)

- 将redo log的写入过程拆分成两个步骤：prepare和commit，中间再穿插写入binlog，具体如下：

  - **prepare阶段**：将**内部事务的XID写入到redo log**，同时将**redo log对应的事务状态设置为prepare**，再将**redo log持久化到磁盘**（innodb_flush_log_at_trx_commit = 1）
  - **commit阶段**：将**XID写入到binlog**，再将**binlog持久化到磁盘**（sync_binlog = 1），再**调用存储引擎的提交事务接口**，将**redo log状态设置为commit**，commit状态只需要write到文件系统的Page Cache（而不需要持久化到磁盘）  =》**只要binlog写磁盘成功，就算redo log的状态还是prepare也一样会认为事务已经执行成功**

 ## 异常重启会出现什么现象

可能会发生崩溃情况如下：

![image-20250118105837137](C:\Users\xyl\AppData\Roaming\Typora\typora-user-images\image-20250118105837137.png)

- 不管是时刻A（redo log已写入磁盘，但binlog还没写入磁盘），还是时刻B（redo log和 binlog都写入磁盘，但还没进入commit阶段）崩溃，此时的redo log会都处于prepare状态
- 在MySQL重启后会按顺序扫描redo log文件，遇到处于prepare状态的redo log，就会用redo log中的XID去binlog查看是否存在该XID：
  - 如果binlog中没有当前内部XA事务的XID，则说明redo log已完成刷盘，但binlog还没有完成刷盘，则回滚事务，对应A时刻崩溃恢复的情况
  - 如果binlog中存在当前内部XA事务的XID，则说明redo log和binlog都已完成刷盘，则提交事务，对应B时刻崩溃恢复的情况

- 对于处于prepare阶段的redo log，是提交事务 还是 回滚事务，取决于是否能在binlog中找到和redo log相同的XID，如果能找到XID，则提交事务，反之则回滚事务，如此可保证redo log和binlog日志的一致性

- **两阶段提交是以binlog写成功作为事务提交成功的标识标志**，因为binlog写成功了，即说明能在binlog中查找到与redo log相同的XID

 ## 事务没提交时，redo log会被持久化到磁盘？
- 事务执行过程中，redo log会直接写在redo log buffer中，这些缓存在redo log buffer中的redo log也会被`后台线程`每隔1秒一起持久化到磁盘
   =》 事务没提交时，redo log也可能会被持久化到磁盘内
- 如果MySQL崩溃了，还没提交事务的redo log已经持久化到磁盘了，等MySQL重启后会回滚（事务没提交时，binlog还没持久化到磁盘）
   =》 所以，**redo log可在事务还没提交前持久化到磁盘，但binlog必须在事务提交后，才可以持久化到磁盘**

 ## 两阶段提交会有什么问题
- 1）磁盘I/O次数高：针对redo log和 binlog的持久化配置（innodb_flush_log_at_trx_commit = 1、sync_binlog = 1），每个事务提交都会进行两次fsync刷盘，一次是redo log刷盘，另一次是binlog刷盘

  ```md
  innodb_flush_log_at_trx_commit = 1: 每次提交事务时，都会将缓存在redo log buffer中的redo log直接持久化到磁盘
  sync_binlog = 1: 每次提交时，都会将缓存在binlog cache中的binlog直接持久化到磁盘
  ```

- 2）锁竞争激烈：两阶段提交能保证【单事务】中两个日志一致，但在多事务的情况下，却不能保证两者的提交顺序一致，因此，在两阶段提交的基础上，需要加锁来保证提交的原子性，以保证多事务的情况下，两个日志的提交顺序一致

  在MySQL的早期版本中，可使用 `prepare_commit_mutex`锁来保证事务提交的顺序，在一个事务获取到锁后才能进入prepare阶段，一直到commit阶段结束才能释放锁，下个事务才可以继续进行prepare操作，加锁可解决事务执行顺序的问题，但并发量较大时会导致对锁的争抢，性能不好



## 组提交
### binlog组提交
- 为解决两阶段提交存在的**磁盘I/O次数高和锁竞争激烈**的问题，MySQL引入了**binlog组（group commit）机制**：

  **有多个事务提交时，会将多个binlog刷盘操作合并成一个，以减少磁盘I/O的次数**

- 引入组提交机制后，prepare阶段不变，只针对commit阶段，将commit阶段拆分为三个过程：
  - flush阶段：多个事务按进入的顺序将binlog从cache写入文件（不刷盘）
  - sync阶段：对binlog文件做 fsync操作（多个事务的binlog合并成一次刷盘操作）
  - commit阶段：各个事务按顺序依次执行commit操作

- 每个阶段都有一个队列，有锁保护，因此保证了事务的写入顺序，第一个入队列的事务会成为leader，**leader领导所在队列的所有事务，全权负责队列的操作**，**完成后通知队列内其他事务的操作结束**
- **每个阶段引入队列**后，**锁就只针对每个队列进行保护**，不再锁住提交事务的整个过程，如此以减小锁的粒度，使多个阶段可并发执行，提升执行效率

### redo log组提交
- MySQL5.6没有redo log组提交，MySQL5.7+才有redo log组提交
- 在MySQL5.6的组提交逻辑中，每个事务各自执行prepare阶段（各自将redo log刷盘），无法对redo log进行组提交
- 在MySQL5.7+中，将**redo log的刷盘时机延迟到flush阶段，在sync阶段之前，通过延迟写redo log的方式，为redo log执行一次组写入**

## 组提交全过程

下面来介绍下 `sync_binlog`和`innodb_flush_log_at_trx_commit`都配置为1的情况下，组提交的全过程：

### flush阶段

- **第一个事务会成为flush阶段的Leader，后续到来的事务都是Follower**
- 获取队列中的事务组，由事务组的Leader对redo log进行 write+fsync，即**一次性将同组事务的redo log刷盘**
- 完成prepare阶段后，将**同组事务执行过程中产生的binlog写入binlog文件**（调用write，不会调用fsync，因此不会刷盘，**binlog会缓存在操作系统的文件系统中**）
=》**flush阶段队列的作用：支撑redo log的组提交**
如果flush阶段完成后，数据库崩溃，由于binlog中没有该组事务的记录，因此MySQL会在重启后回滚该组事务

### sync阶段

- 等待该组事务的binlog写入到binlog文件后，并不会马上执行刷盘的操作，而是会等待一段时间，该等待的时长由 binlog_group_commit_sync_delay 参数控制，其目的是为了**组合更多事务的binlog，再一起刷盘**

- 在等待的过程中，如果**事务的数量提前达到了设置的 `binlog_group_commit_sync_no_delay_count`参数值，就不需要继续等待，会立即将binlog刷盘**

  =》**sync阶段队列，用于支持binlog的组提交**

- 如果要提升binlog组提交的效果，可设置如下两个参数：
  - `binlog_group_commit_sync_delay = N`，表示**等待N微秒后直接调用`fsync`，将文件系统中Page Cache中的 `binlog`刷盘，即将`binlog`文件持久化到磁盘`**
  - `binlog_group_commit_sync_no_delay_count = N`，表示**如果队列中的事务数达到N个**，就忽略`binlog_group_commit_sync_delay`参数值，**直接调用fsync，将文件系统中的 Page Cache 中的binlog刷盘**

- 如果在sync阶段完成后数据库崩溃，由于binlog中已经存在事务记录，MySQL会在重启后通过redo log刷盘的数据继续提交事务

### commit阶段

- 进入commit阶段，**调用引擎的提交事务接口，将redo log状态设置为commit**
  =》**commit阶段队列的作用：承接sync阶段的事务，存储引擎完成最后事务的提交**，使sync可尽早地处理下一组事务，最大化组提交的的效率

 ## MySQl磁盘I/O高的优化方案
 事务在提交时，需要将binlog和redo log持久化到磁盘，如果出现MySQL磁盘I/O很高的情况，可通过控制如下参数，来延迟 binlog和redo log刷盘的时机，从而降低磁盘I/O的频率
 - 设置组提交的两个参数：binlog_group_commit_sync_delay 和 binlog_group_commit_sync_no_delay_count 参数，来延迟binlog刷盘的时机，从而减少binlog的刷盘次数
   该方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但即使MySQL进程中途挂了，也没有丢失数据的风险，因为binlog早已被写入Page Cache，只要系统没有宕机，
   缓存在Page Cache中的binlog就会被持久化到磁盘
 - 设置sync_binlog = N（N>1,N通常为100~1000的值），每次提交事务都write，但累积N个事务后才fsync，相当于延迟了binlog刷盘的时机，但风险是：主机掉电后会丢N个事务的binlog日志
 - 设置innodb_flush_log_at_trx_commit = 2，每次事务提交时，都只是缓存在redo log buffer中的redo log文件中（操作系统的文件系统中有Page Cache，可用于专门缓存文件数据），
   所以，写入`redo log文件`即意味着写入到了操作系统的文件缓存中，其后再交由操作系统 控制持久化到磁盘的时机，但风险是：主机掉电会丢失数据    

## 小结
下面以update语句的执行过程来梳理undo log、redo log、binlog、组提交等知识点，
当优化器分析出成本最小的执行计划后，执行器就会按照执行计划开始更新操作，具体更新一条记录：`update t_user set name = 'xuzhix' where id = 1`，流程如下：

- 1）执行器负责具体执行，会调用存储引擎的接口，通过**主键索引树搜索**获取id=1 对应的行记录，
  - 如果id = 1 的行记录所在的数据页本就在 Buffer Pool，就直接返回给执行器更新
  - 如果id = 1 的行记录所在的数据页不在Buffer Pool中，则将数据页从磁盘读取到Buffer Pool，再返回记录给执行器

- 2）**执行器得到聚簇索引记录后，会查看更新前和更新后的行记录是否一致**：

  - 如果一致则不进行后续更新流程

  - 如果不一致则将更新前和更新后的记录都当作参数传递给InnoDB存储引擎层，让InnoDB真正地执行更新记录的操作

- 3）**开启事务，InnoDB层更新记录前，会记录相应的undo log**（生成一条undo log），**undo log会写入Buffer Pool中的undo log页**，在undo log修改该undo log页后，需要记录对应的redo log

- 4）**InnoDB层开始更新记录，会先更新内存**（同时标记为脏页），再**将记录写到redo log**中（此时更新操作就算完成了）
    为减少磁盘I/O，并不会立即将脏页写入磁盘，会**后续由后台线程在合适时机将脏页写入到磁盘**，此即为WAL技术

- 5）**在一条更新语句执行完成后，会开始记录该语句对应的binlog**，此时**记录的binlog会被保存到binlog cache**（并不会立即刷新到磁盘上的binlog文件），在**事务提交时才会统一将事务运行过程中的所有binlog刷新到磁盘**

- 6）等事务提交后，进行两阶段提交

## 扩展

MySQL中的日志除了前面提到的`undo log`、`redo log`、`binlog`日志，还有错误日志、慢查询日志、一般查询日志等。

### 错误日志（error log）

- `error log`主要记录MySQL在启动、关闭或者运行过程中的错误信息，在MySQL的配置文件`my.cnf`中，可通过`log-error=/var/log/mysqld.log`来配置MySQL错误日志的位置

### 慢查询日志（slow query log）

- MySQL的慢查询日志是MySQL提供的一种日志记录，用于记录响应时间超过阀值的SQL（运行时间超过`long_query_time`值的SQL），则会被记录到慢查询日志中
- 通过慢查询日志，结合explain执行计划进行全面分析
- 在生产环境中，如果要手动分析日志，查找、分析SQL会比较麻烦，MySQL提供了日志分析工具`mysqldumpslow`

### 一般查询日志（general log）

- `general log`记录了客户端连接信息以及执行的SQL语句信息，包括客户端连接服务器的时间、客户端发送的所有 SQL 以及其他事件，MySQL 服务启动和关闭等等。



