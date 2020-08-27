# mysql海量数据存储与优化上笔记

第一部分 MySQL架构原理

第1节 MySQL体系架构

MySQL Server架构自顶向下大致可以分网络连接层、服务层、存储引擎层和系统文件层。

一、网络连接层

客户端连接器（Client Connectors）：提供与MySQL服务器建立的支持。目前几乎支持所有主流 的服务端编程技术，例如常见的 Java、C、Python、.NET等，它们通过各自API技术与MySQL建立 连接。

二、服务层（MySQL Server）

服务层是MySQL Server的核心，主要包含系统管理和控制工具、连接池、SQL接口、解析器、查询优 化器和缓存六个部分。

连接池（Connection Pool）：负责存储和管理客户端与数据库的连接，一个线程负责管理一个 连接。

系统管理和控制工具（Management Services & Utilities）：例如备份恢复、安全管理、集群

管理等

SQL接口（SQL Interface）：用于接受客户端发送的各种SQL命令，并且返回用户需要查询的结 果。比如DML、DDL、存储过程、视图、触发器等。

解析器（Parser）：负责将请求的SQL解析生成一个"解析树"。然后根据一些MySQL规则进一步 检查解析树是否合法。

查询优化器（Optimizer）：当“解析树”通过解析器语法检查后，将交由优化器将其转化成执行计 划，然后与存储引擎交互。

select uid,name from user where gender=1; 选取--》投影--》联接 策略 1）select先根据where语句进行选取，并不是查询出全部数据再过滤 2）select查询根据uid和name进行属性投影，并不是取出所有字段 3）将前面选取和投影联接起来最终生成查询结果 缓存（Cache&Buﬀer）： 缓存机制是由一系列小缓存组成的。比如表缓存，记录缓存，权限缓 存，引擎缓存等。如果查询缓存有命中的查询结果，查询语句就可以直接去查询缓存中取数据。

三、存储引擎层（Pluggable Storage Engines）

存储引擎负责MySQL中数据的存储与提取，与底层系统文件进行交互。MySQL存储引擎是插件式的， 服务器中的查询执行引擎通过接口与存储引擎进行通信，接口屏蔽了不同存储引擎之间的差异 。现在有 很多种存储引擎，各有各的特点，最常见的是MyISAM和InnoDB。

四、系统文件层（File System）

该层负责将数据库的数据和日志存储在文件系统之上，并完成与存储引擎的交互，是文件的物理存储 层。主要包含日志文件，数据文件，配置文件，pid 文件，socket 文件等。

日志文件

错误日志（Error log） 默认开启，show variables like '%log_error%' 通用查询日志（General query log） 记录一般查询语句，show variables like '%general%'; 二进制日志（binary log） 记录了对MySQL数据库执行的更改操作，并且记录了语句的发生时间、执行时长；但是它不 记录select、show等不修改数据库的SQL。主要用于数据库恢复和主从复制。 show variables like '%log_bin%'; //是否开启 show variables like '%binlog%'; //参数查看 show binary logs;//查看日志文件 慢查询日志（Slow query log） 记录所有执行时间超时的查询SQL，默认是10秒。

show variables like '%slow_query%'; //是否开启 show variables like '%long_query_time%'; //时长

配置文件

用于存放MySQL所有的配置信息文件，比如my.cnf、my.ini等。

数据文件

db.opt 文件：记录这个库的默认使用的字符集和校验规则。 frm 文件：存储与表相关的元数据（meta）信息，包括表结构的定义信息等，每一张表都会 有一个frm 文件。 MYD 文件：MyISAM 存储引擎专用，存放 MyISAM 表的数据（data)，每一张表都会有一个 .MYD 文件。 MYI 文件：MyISAM 存储引擎专用，存放 MyISAM 表的索引相关信息，每一张 MyISAM 表对 应一个 .MYI 文件。 ibd文件和 IBDATA 文件：存放 InnoDB 的数据文件（包括索引）。InnoDB 存储引擎有两种 表空间方式：独享表空间和共享表空间。独享表空间使用 .ibd 文件来存放数据，且每一张 InnoDB 表对应一个 .ibd 文件。共享表空间使用 .ibdata 文件，所有表共同使用一个（或多 个，自行配置）.ibdata 文件。 ibdata1 文件：系统表空间数据文件，存储表元数据、Undo日志等 。 ib_logﬁle0、ib_logﬁle1 文件：Redo log 日志文件。 pid 文件

pid 文件是 mysqld 应用程序在 Unix/Linux 环境下的一个进程文件，和许多其他 Unix/Linux 服务 端程序一样，它存放着自己的进程 id。

socket 文件

socket 文件也是在 Unix/Linux 环境下才有的，用户在 Unix/Linux 环境下客户端连接可以不通过 TCP/IP 网络而直接使用 Unix Socket 来连接 MySQL。

第2节 MySQL运行机制 ①建立连接（Connectors&Connection Pool），通过客户端/服务器通信协议与MySQL建立连 接。MySQL 客户端与服务端的通信方式是 “ 半双工 ”。对于每一个 MySQL 的连接，时刻都有一个 线程状态来标识这个连接正在做什么。

通讯机制：

全双工：能同时发送和接收数据，例如平时打电话。 半双工：指的某一时刻，要么发送数据，要么接收数据，不能同时。例如早期对讲机 单工：只能发送数据或只能接收数据。例如单行道

线程状态：

show processlist; //查看用户正在运行的线程信息，root用户能查看所有线程，其他用户只能看自 己的

id：线程ID，可以使用kill xx；

user：启动这个线程的用户

Host：发送请求的客户端的IP和端口号

db：当前命令在哪个库执行

Command：该线程正在执行的操作命令

Create DB：正在创建库操作 Drop DB：正在删除库操作 Execute：正在执行一个PreparedStatement Close Stmt：正在关闭一个PreparedStatement Query：正在执行一个语句 Sleep：正在等待客户端发送语句 Quit：正在退出 Shutdown：正在关闭服务器 Time：表示该线程处于当前状态的时间，单位是秒

State：线程状态

Updating：正在搜索匹配记录，进行修改 Sleeping：正在等待客户端发送新请求 Starting：正在执行请求处理 Checking table：正在检查数据表 Closing table : 正在将表中数据刷新到磁盘中 Locked：被其他查询锁住了记录 Sending Data：正在处理Select查询，同时将结果发送给客户端 Info：一般记录线程执行的语句，默认显示前100个字符。想查看完整的使用show full processlist;

②查询缓存（Cache&Buﬀer），这是MySQL的一个可优化查询的地方，如果开启了查询缓存且在 查询缓存过程中查询到完全相同的SQL语句，则将查询结果直接返回给客户端；如果没有开启查询 缓存或者没有查询到完全相同的 SQL 语句则会由解析器进行语法语义解析，并生成“解析树”。

缓存Select查询的结果和SQL语句

执行Select查询时，先查询缓存，判断是否存在可用的记录集，要求是否完全相同（包括参 数值），这样才会匹配缓存数据命中。

即使开启查询缓存，以下SQL也不能缓存

查询语句使用SQL_NO_CACHE 查询的结果大于query_cache_limit设置 查询中有一些不确定的参数，比如now() show variables like '%query_cache%'; //查看查询缓存是否启用，空间大小，限制等

show status like 'Qcache%'; //查看更详细的缓存参数，可用缓存空间，缓存块，缓存多少等 ③解析器（Parser）将客户端发送的SQL进行语法解析，生成"解析树"。预处理器根据一些MySQL 规则进一步检查“解析树”是否合法，例如这里将检查数据表和数据列是否存在，还会解析名字和别 名，看看它们是否有歧义，最后生成新的“解析树”。

④查询优化器（Optimizer）根据“解析树”生成最优的执行计划。MySQL使用很多优化策略生成最 优的执行计划，可以分为两类：静态优化（编译时优化）、动态优化（运行时优化）。

等价变换策略

5=5 and a>5 改成 a > 5 a < b and a=5 改成b>5 and a=5 基于联合索引，调整条件位置等 优化count、min、max等函数

InnoDB引擎min函数只需要找索引最左边 InnoDB引擎max函数只需要找索引最右边 MyISAM引擎count(*)，不需要计算，直接返回 提前终止查询

使用了limit查询，获取limit所需的数据，就不在继续遍历后面数据 in的优化

MySQL对in查询，会先进行排序，再采用二分法查找数据。比如where id in (2,1,3)，变 成 in (1,2,3) ⑤查询执行引擎负责执行 SQL 语句，此时查询执行引擎会根据 SQL 语句中表的存储引擎类型，以 及对应的API接口与底层存储引擎缓存或者物理文件的交互，得到查询结果并返回给客户端。若开 启用查询缓存，这时会将SQL 语句和结果完整地保存到查询缓存（Cache&Buﬀer）中，以后若有 相同的 SQL 语句执行则直接返回结果。

如果开启了查询缓存，先将查询结果做缓存操作 返回结果过多，采用增量模式返回

第3节 MySQL存储引擎

存储引擎在MySQL的体系架构中位于第三层，负责MySQL中的数据的存储和提取，是与文件打交道的 子系统，它是根据MySQL提供的文件访问层抽象接口定制的一种文件访问机制，这种机制就叫作存储引 擎。

使用show engines命令，就可以查看当前数据库支持的引擎信息。

在5.5版本之前默认采用MyISAM存储引擎，从5.5开始采用InnoDB存储引擎。 InnoDB：支持事务，具有提交，回滚和崩溃恢复能力，事务安全 MyISAM：不支持事务和外键，访问速度快 Memory：利用内存创建表，访问速度非常快，因为数据在内存，而且默认使用Hash索引，但是 一旦关闭，数据就会丢失 Archive：归档类型引擎，仅能支持insert和select语句 Csv：以CSV文件进行数据存储，由于文件限制，所有列必须强制指定not null，另外CSV引擎也不 支持索引和分区，适合做数据交换的中间表 BlackHole: 黑洞，只进不出，进来消失，所有插入数据都不会保存 Federated：可以访问远端MySQL数据库中的表。一个本地表，不保存数据，访问远程表内容。 MRG_MyISAM：一组MyISAM表的组合，这些MyISAM表必须结构相同，Merge表本身没有数据， 对Merge操作可以对一组MyISAM表进行操作。

3.1 InnoDB和MyISAM对比

InnoDB和MyISAM是使用MySQL时最常用的两种引擎类型，我们重点来看下两者区别。

事务和外键

InnoDB支持事务和外键，具有安全性和完整性，适合大量insert或update操作

MyISAM不支持事务和外键，它提供高速存储和检索，适合大量的select查询操作

锁机制

InnoDB支持行级锁，锁定指定记录。基于索引来加锁实现。

MyISAM支持表级锁，锁定整张表。

索引结构

InnoDB使用聚集索引（聚簇索引），索引和记录在一起存储，既缓存索引，也缓存记录。

MyISAM使用非聚集索引（非聚簇索引），索引和记录分开。

并发处理能力

MyISAM使用表锁，会导致写操作并发率低，读之间并不阻塞，读写阻塞。

InnoDB读写阻塞可以与隔离级别有关，可以采用多版本并发控制（MVCC）来支持高并发

存储文件

InnoDB表对应两个文件，一个.frm表结构文件，一个.ibd数据文件。InnoDB表最大支持64TB；

MyISAM表对应三个文件，一个.frm表结构文件，一个MYD表数据文件，一个.MYI索引文件。从 MySQL5.0开始默认限制是256TB。 适用场景

MyISAM

不需要事务支持（不支持） 并发相对较低（锁定机制问题） 数据修改相对较少，以读为主 数据一致性要求不高

InnoDB

需要事务支持（具有较好的事务特性） 行级锁定对高并发有很好的适应能力 数据更新较为频繁的场景 数据一致性要求较高 硬件设备内存较大，可以利用InnoDB较好的缓存能力来提高内存利用率，减少磁盘IO

总结

两种引擎该如何选择？

是否需要事务？有，InnoDB 是否存在并发修改？有，InnoDB 是否追求快速查询，且数据修改少？是，MyISAM 在绝大多数情况下，推荐使用InnoDB

扩展资料：各个存储引擎特性对比 3.2 InnoDB存储结构

从MySQL 5.5版本开始默认使用InnoDB作为引擎，它擅长处理事务，具有自动崩溃恢复的特性，在日 常开发中使用非常广泛。下面是官方的InnoDB引擎架构图，主要分为内存结构和磁盘结构两大部分。

一、InnoDB内存结构 内存结构主要包括Buﬀer Pool、Change Buﬀer、Adaptive Hash Index和Log Buﬀer四大组件。 Buﬀer Pool：缓冲池，简称BP。BP以Page页为单位，默认大小16K，BP的底层采用链表数 据结构管理Page。在InnoDB访问表记录和索引时会在Page页中缓存，以后使用可以减少磁 盘IO操作，提升效率。

Page管理机制 Page根据状态可以分为三种类型：

free page ： 空闲page，未被使用 clean page：被使用page，数据没有被修改过 dirty page：脏页，被使用page，数据被修改过，页中数据和磁盘的数据产生了不 一致

针对上述三种page类型，InnoDB通过三种链表结构来维护和管理

free list ：表示空闲缓冲区，管理free page ﬂush list：表示需要刷新到磁盘的缓冲区，管理dirty page，内部page按修改时间 排序。脏页即存在于ﬂush链表，也在LRU链表中，但是两种互不影响，LRU链表负 责管理page的可用性和释放，而ﬂush链表负责管理脏页的刷盘操作。 lru list：表示正在使用的缓冲区，管理clean page和dirty page，缓冲区以 midpoint为基点，前面链表称为new列表区，存放经常访问的数据，占63%；后 面的链表称为old列表区，存放使用较少数据，占37%。 改进型LRU算法维护

普通LRU：末尾淘汰法，新数据从链表头部加入，释放空间时从末尾淘汰

改性LRU：链表分为new和old两个部分，加入元素时并不是从表头插入，而是从中间 midpoint位置插入，如果数据很快被访问，那么page就会向new列表头部移动，如果 数据没有被访问，会逐步向old尾部移动，等待淘汰。

每当有新的page数据读取到buﬀer pool时，InnoDb引擎会判断是否有空闲页，是否足 够，如果有就将free page从free list列表删除，放入到LRU列表中。没有空闲页，就会 根据LRU算法淘汰LRU链表默认的页，将内存空间释放分配给新的页。

Buﬀer Pool配置参数 show variables like '%innodb_page_size%'; //查看page页大小 show variables like '%innodb_old%'; //查看lru list中old列表参数 show variables like '%innodb_buﬀer%'; //查看buﬀer pool参数 建议：将innodb_buﬀer_pool_size设置为总内存大小的60%-80%， innodb_buﬀer_pool_instances可以设置为多个，这样可以避免缓存争夺。 Change Buﬀer：写缓冲区，简称CB。在进行DML操作时，如果BP没有其相应的Page数据， 并不会立刻将磁盘页加载到缓冲池，而是在CB记录缓冲变更，等未来数据被读取时，再将数 据合并恢复到BP中。 ChangeBuﬀer占用BuﬀerPool空间，默认占25%，最大允许占50%，可以根据读写业务量来 进行调整。参数innodb_change_buﬀer_max_size; 当更新一条记录时，该记录在BuﬀerPool存在，直接在BuﬀerPool修改，一次内存操作。如 果该记录在BuﬀerPool不存在（没有命中），会直接在ChangeBuﬀer进行一次内存操作，不 用再去磁盘查询数据，避免一次磁盘IO。当下次查询记录时，会先进性磁盘读取，然后再从 ChangeBuﬀer中读取信息合并，最终载入BuﬀerPool中。

写缓冲区，仅适用于非唯一普通索引页，为什么？

如果在索引设置唯一性，在进行修改时，InnoDB必须要做唯一性校验，因此必须查询磁盘， 做一次IO操作。会直接将记录查询到BuﬀerPool中，然后在缓冲池修改，不会在 ChangeBuﬀer操作。 Adaptive Hash Index：自适应哈希索引，用于优化对BP数据的查询。InnoDB存储引擎会监 控对表索引的查找，如果观察到建立哈希索引可以带来速度的提升，则建立哈希索引，所以 称之为自适应。InnoDB存储引擎会自动根据访问的频率和模式来为某些页建立哈希索引。

Log Buﬀer：日志缓冲区，用来保存要写入磁盘上log文件（Redo/Undo）的数据，日志缓冲 区的内容定期刷新到磁盘log文件中。日志缓冲区满时会自动将其刷新到磁盘，当遇到BLOB 或多行更新的大事务操作时，增加日志缓冲区可以节省磁盘I/O。

LogBuﬀer主要是用于记录InnoDB引擎日志，在DML操作时会产生Redo和Undo日志。

LogBuﬀer空间满了，会自动写入磁盘。可以通过将innodb_log_buﬀer_size参数调大，减少 磁盘IO频率

innodb_ﬂush_log_at_trx_commit参数控制日志刷新行为，默认为1

0 ： 每隔1秒写日志文件和刷盘操作（写日志文件LogBuﬀer-->OS cache，刷盘OS cache-->磁盘文件），最多丢失1秒数据 1：事务提交，立刻写日志文件和刷盘，数据不丢失，但是会频繁IO操作 2：事务提交，立刻写日志文件，每隔1秒钟进行刷盘操作 二、InnoDB磁盘结构

InnoDB磁盘主要包含Tablespaces，InnoDB Data Dictionary，Doublewrite Buﬀer、Redo Log 和Undo Logs。

表空间（Tablespaces）：用于存储表结构和数据。表空间又分为系统表空间、独立表空间、 通用表空间、临时表空间、Undo表空间等多种类型； 系统表空间（The System Tablespace） 包含InnoDB数据字典，Doublewrite Buﬀer，Change Buﬀer，Undo Logs的存储区 域。系统表空间也默认包含任何用户在系统表空间创建的表数据和索引数据。系统表空 间是一个共享的表空间因为它是被多个表共享的。该空间的数据文件通过参数 innodb_data_ﬁle_path控制，默认值是ibdata1:12M:autoextend(文件名为ibdata1、 12MB、自动扩展)。

独立表空间（File-Per-Table Tablespaces） 默认开启，独立表空间是一个单表表空间，该表创建于自己的数据文件中，而非创建于 系统表空间中。当innodb_ﬁle_per_table选项开启时，表将被创建于表空间中。否则， innodb将被创建于系统表空间中。每个表文件表空间由一个.ibd数据文件代表，该文件 默认被创建于数据库目录中。表空间的表文件支持动态（dynamic）和压缩 （commpressed）行格式。

通用表空间（General Tablespaces） 通用表空间为通过create tablespace语法创建的共享表空间。通用表空间可以创建于 mysql数据目录外的其他表空间，其可以容纳多张表，且其支持所有的行格式。

CREATE TABLESPACE ts1 ADD DATAFILE ts1.ibd Engine=InnoDB; //创建表空

间ts1

CREATE TABLE t1 (c1 INT PRIMARY KEY) TABLESPACE ts1; //将表添加到ts1

表空间

撤销表空间（Undo Tablespaces） 撤销表空间由一个或多个包含Undo日志文件组成。在MySQL 5.7版本之前Undo占用的 是System Tablespace共享区，从5.7开始将Undo从System Tablespace分离了出来。 InnoDB使用的undo表空间由innodb_undo_tablespaces配置选项控制，默认为0。参 数值为0表示使用系统表空间ibdata1;大于0表示使用undo表空间undo_001、 undo_002等。

临时表空间（Temporary Tablespaces） 分为session temporary tablespaces 和global temporary tablespace两种。session temporary tablespaces 存储的是用户创建的临时表和磁盘内部的临时表。global temporary tablespace储存用户临时表的回滚段（rollback segments ）。mysql服务 器正常关闭或异常终止时，临时表空间将被移除，每次启动时会被重新创建。

数据字典（InnoDB Data Dictionary）

InnoDB数据字典由内部系统表组成，这些表包含用于查找表、索引和表字段等对象的元数 据。元数据物理上位于InnoDB系统表空间中。由于历史原因，数据字典元数据在一定程度上 与InnoDB表元数据文件（.frm文件）中存储的信息重叠。

双写缓冲区（Doublewrite Buﬀer）

位于系统表空间，是一个存储区域。在BuﬀerPage的page页刷新到磁盘真正的位置前，会先 将数据存在Doublewrite 缓冲区。如果在page页写入过程中出现操作系统、存储子系统或 mysqld进程崩溃，InnoDB可以在崩溃恢复期间从Doublewrite 缓冲区中找到页面的一个好 备份。在大多数情况下，默认情况下启用双写缓冲区，要禁用Doublewrite 缓冲区，可以将 innodb_doublewrite设置为0。使用Doublewrite 缓冲区时建议将innodb_ﬂush_method设 置为O_DIRECT。

MySQL的innodb_ﬂush_method这个参数控制着innodb数据文件及redo log的打开、 刷写模式。有三个值：fdatasync(默认)，O_DSYNC，O_DIRECT。设置O_DIRECT表示 数据文件写入操作会通知操作系统不要缓存数据，也不要用预读，直接从Innodb Buﬀer写到磁盘文件。

默认的fdatasync意思是先写入操作系统缓存，然后再调用fsync()函数去异步刷数据文 件与redo log的缓存信息。

重做日志（Redo Log）

重做日志是一种基于磁盘的数据结构，用于在崩溃恢复期间更正不完整事务写入的数据。 MySQL以循环方式写入重做日志文件，记录InnoDB中所有对Buﬀer Pool修改的日志。当出 现实例故障（像断电），导致数据未能更新到数据文件，则数据库重启时须redo，重新把数 据更新到数据文件。读写事务在执行的过程中，都会不断的产生redo log。默认情况下，重 做日志在磁盘上由两个名为ib_logﬁle0和ib_logﬁle1的文件物理表示。

撤销日志（Undo Logs）

撤消日志是在事务开始之前保存的被修改数据的备份，用于例外情况时回滚事务。撤消日志 属于逻辑日志，根据每行记录进行记录。撤消日志存在于系统表空间、撤消表空间和临时表 空间中。

三、新版本结构演变 MySQL 5.7 版本

将 Undo日志表空间从共享表空间 ibdata 文件中分离出来，可以在安装 MySQL 时由用 户自行指定文件大小和数量。 增加了 temporary 临时表空间，里面存储着临时表或临时查询结果集的数据。 Buﬀer Pool 大小可以动态修改，无需重启数据库实例。 MySQL 8.0 版本

将InnoDB表的数据字典和Undo都从共享表空间ibdata中彻底分离出来了，以前需要 ibdata中数据字典与独立表空间ibd文件中数据字典一致才行，8.0版本就不需要了。 temporary 临时表空间也可以配置多个物理文件，而且均为 InnoDB 存储引擎并能创建 索引，这样加快了处理的速度。 用户可以像 Oracle 数据库那样设置一些表空间，每个表空间对应多个物理文件，每个 表空间可以给多个表使用，但一个表只能存储在一个表空间中。 将Doublewrite Buﬀer从共享表空间ibdata中也分离出来了。

3.3 InnoDB线程模型 IO Thread

在InnoDB中使用了大量的AIO（Async IO）来做读写处理，这样可以极大提高数据库的性能。在 InnoDB1.0版本之前共有4个IO Thread，分别是write，read，insert buﬀer和log thread，后来 版本将read thread和write thread分别增大到了4个，一共有10个了。

read thread ： 负责读取操作，将数据从磁盘加载到缓存page页。4个 write thread：负责写操作，将缓存脏页刷新到磁盘。4个

log thread：负责将日志缓冲区内容刷新到磁盘。1个 insert buﬀer thread ：负责将写缓冲内容刷新到磁盘。1个 Purge Thread

事务提交之后，其使用的undo日志将不再需要，因此需要Purge Thread回收已经分配的undo 页。

show variables like '%innodb_purge_threads%';

Page Cleaner Thread

作用是将脏数据刷新到磁盘，脏数据刷盘后相应的redo log也就可以覆盖，即可以同步数据，又能 达到redo log循环使用的目的。会调用write thread线程处理。

show variables like '%innodb_page_cleaners%';

Master Thread

Master thread是InnoDB的主线程，负责调度其他各线程，优先级最高。作用是将缓冲池中的数 据异步刷新到磁盘 ，保证数据的一致性。包含：脏页的刷新（page cleaner thread）、undo页 回收（purge thread）、redo日志刷新（log thread）、合并写缓冲等。内部有两个主处理，分别 是每隔1秒和10秒处理。

每1秒的操作：

刷新日志缓冲区，刷到磁盘 合并写缓冲区数据，根据IO读写压力来决定是否操作 刷新脏页数据到磁盘，根据脏页比例达到75%才操作（innodb_max_dirty_pages_pct， innodb_io_capacity）

每10秒的操作：

刷新脏页数据到磁盘 合并写缓冲区数据 刷新日志缓冲区 删除无用的undo页

3.4 InnoDB数据文件

一、InnoDB文件存储结构

InnoDB数据文件存储结构：

分为一个ibd数据文件-->Segment（段）-->Extent（区）-->Page（页）-->Row（行）

Tablesapce

表空间，用于存储多个ibd数据文件，用于存储表的记录和索引。一个文件包含多个段。

Segment

段，用于管理多个Extent，分为数据段（Leaf node segment）、索引段（Non-leaf node segment）、回滚段（Rollback segment）。一个表至少会有两个segment，一个管理数 据，一个管理索引。每多创建一个索引，会多两个segment。

Extent

区，一个区固定包含64个连续的页，大小为1M。当表空间不足，需要分配新的页资源，不会 一页一页分，直接分配一个区。

Page

页，用于存储多个Row行记录，大小为16K。包含很多种页类型，比如数据页，undo页，系 统页，事务数据页，大的BLOB对象页。 Row

行，包含了记录的字段值，事务ID（Trx id）、滚动指针（Roll pointer）、字段指针（Field pointers）等信息。

Page是文件最基本的单位，无论何种类型的page，都是由page header，page trailer和page body组成。如下图所示，

二、InnoDB文件存储格式 通过 SHOW TABLE STATUS 命令

一般情况下，如果row_format为REDUNDANT、COMPACT，文件格式为Antelope；如果 row_format为DYNAMIC和COMPRESSED，文件格式为Barracuda。

通过 information_schema 查看指定表的文件格式

select * from information_schema.innodb_sys_tables; 三、File文件格式（File-Format）

在早期的InnoDB版本中，文件格式只有一种，随着InnoDB引擎的发展，出现了新文件格式，用于 支持新的功能。目前InnoDB只支持两种文件格式：Antelope 和 Barracuda。

Antelope: 先前未命名的，最原始的InnoDB文件格式，它支持两种行格式：COMPACT和 REDUNDANT，MySQL 5.6及其以前版本默认格式为Antelope。 Barracuda: 新的文件格式。它支持InnoDB的所有行格式，包括新的行格式：COMPRESSED 和 DYNAMIC。

通过innodb_ﬁle_format 配置参数可以设置InnoDB文件格式，之前默认值为Antelope，5.7版本 开始改为Barracuda。

四、Row行格式（Row_format）

表的行格式决定了它的行是如何物理存储的，这反过来又会影响查询和DML操作的性能。如果在 单个page页中容纳更多行，查询和索引查找可以更快地工作，缓冲池中所需的内存更少，写入更 新时所需的I/O更少。

InnoDB存储引擎支持四种行格式：REDUNDANT、COMPACT、DYNAMIC和COMPRESSED。

DYNAMIC和COMPRESSED新格式引入的功能有：数据压缩、增强型长列数据的页外存储和大索引

前缀。

每个表的数据分成若干页来存储，每个页中采用B树结构存储；

如果某些字段信息过长，无法存储在B树节点中，这时候会被单独分配空间，此时被称为溢出页， 该字段被称为页外列。

REDUNDANT 行格式

使用REDUNDANT行格式，表会将变长列值的前768字节存储在B树节点的索引记录中，其余 的存储在溢出页上。对于大于等于786字节的固定长度字段InnoDB会转换为变长字段，以便 能够在页外存储。

COMPACT 行格式

与REDUNDANT行格式相比，COMPACT行格式减少了约20%的行存储空间，但代价是增加了 某些操作的CPU使用量。如果系统负载是受缓存命中率和磁盘速度限制，那么COMPACT格式 可能更快。如果系统负载受到CPU速度的限制，那么COMPACT格式可能会慢一些。

DYNAMIC 行格式

使用DYNAMIC行格式，InnoDB会将表中长可变长度的列值完全存储在页外，而索引记录只 包含指向溢出页的20字节指针。大于或等于768字节的固定长度字段编码为可变长度字段。 DYNAMIC行格式支持大索引前缀，最多可以为3072字节，可通过innodb_large_preﬁx参数 控制。

COMPRESSED 行格式

COMPRESSED行格式提供与DYNAMIC行格式相同的存储特性和功能，但增加了对表和索引 数据压缩的支持。

在创建表和索引时，文件格式都被用于每个InnoDB表数据文件（其名称与*.ibd匹配）。修改文件 格式的方法是重新创建表及其索引，最简单方法是对要修改的每个表使用以下命令：

ALTER TABLE 表名 ROW_FORMAT=格式类型; 3.5 Undo Log

3.5.1 Undo Log介绍

Undo：意为撤销或取消，以撤销操作为目的，返回指定某个状态的操作。

Undo Log：数据库事务开始之前，会将要修改的记录存放到 Undo 日志里，当事务回滚时或者数 据库崩溃时，可以利用 Undo 日志，撤销未提交事务对数据库产生的影响。

Undo Log产生和销毁：Undo Log在事务开始前产生；事务在提交时，并不会立刻删除undo log，innodb会将该事务对应的undo log放入到删除列表中，后面会通过后台线程purge thread进 行回收处理。Undo Log属于逻辑日志，记录一个变化过程。例如执行一个delete，undolog会记 录一个insert；执行一个update，undolog会记录一个相反的update。

Undo Log存储：undo log采用段的方式管理和记录。在innodb数据文件中包含一种rollback segment回滚段，内部包含1024个undo log segment。可以通过下面一组参数来控制Undo log存 储。

show variables like '%innodb_undo%';

3.5.2 Undo Log作用

实现事务的原子性

Undo Log 是为了实现事务的原子性而出现的产物。事务处理过程中，如果出现了错误或者用户执 行了 ROLLBACK 语句，MySQL 可以利用 Undo Log 中的备份将数据恢复到事务开始之前的状态。

实现多版本并发控制（MVCC）

Undo Log 在 MySQL InnoDB 存储引擎中用来实现多版本并发控制。事务未提交之前，Undo Log 保存了未提交之前的版本数据，Undo Log 中的数据可作为数据旧版本快照供其他并发事务进行快 照读。

事务A手动开启事务，执行更新操作，首先会把更新命中的数据备份到 Undo Buﬀer 中。

事务B手动开启事务，执行查询操作，会读取 Undo 日志数据返回，进行快照读

3.6 Redo Log和Binlog

Redo Log和Binlog是MySQL日志系统中非常重要的两种机制，也有很多相似之处，下面介绍下两者细 节和区别。

3.6.1 Redo Log日志

Redo Log介绍 Redo：顾名思义就是重做。以恢复操作为目的，在数据库发生意外时重现操作。 Redo Log：指事务中修改的任何数据，将最新的数据备份存储的位置（Redo Log），被称为重做 日志。 Redo Log 的生成和释放：随着事务操作的执行，就会生成Redo Log，在事务提交时会将产生 Redo Log写入Log Buﬀer，并不是随着事务的提交就立刻写入磁盘文件。等事务操作的脏页写入 到磁盘之后，Redo Log 的使命也就完成了，Redo Log占用的空间就可以重用（被覆盖写入）。

Redo Log工作原理 Redo Log 是为了实现事务的持久性而出现的产物。防止在发生故障的时间点，尚有脏页未写入表 的 IBD 文件中，在重启 MySQL 服务的时候，根据 Redo Log 进行重做，从而达到事务的未入磁盘 数据进行持久化这一特性。

Redo Log写入机制 Redo Log 文件内容是以顺序循环的方式写入文件，写满时则回溯到第一个文件，进行覆盖写。

如图所示：

write pos 是当前记录的位置，一边写一边后移，写到最后一个文件末尾后就回到 0 号文件开 头； checkpoint 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数 据文件；

write pos 和 checkpoint 之间还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示写满，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。

Redo Log相关配置参数

每个InnoDB存储引擎至少有1个重做日志文件组（group），每个文件组至少有2个重做日志文 件，默认为ib_logﬁle0和ib_logﬁle1。可以通过下面一组参数控制Redo Log存储：

show variables like '%innodb_log%';

Redo Buﬀer 持久化到 Redo Log 的策略，可通过 Innodb_ﬂush_log_at_trx_commit 设置：

0：每秒提交 Redo buﬀer ->OS cache -> ﬂush cache to disk，可能丢失一秒内的事务数 据。由后台Master线程每隔 1秒执行一次操作。 1（默认值）：每次事务提交执行 Redo Buﬀer -> OS cache -> ﬂush cache to disk，最安 全，性能最差的方式。 2：每次事务提交执行 Redo Buﬀer -> OS cache，然后由后台Master线程再每隔1秒执行OS cache -> ﬂush cache to disk 的操作。

一般建议选择取值2，因为 MySQL 挂了数据没有损失，整个服务器挂了才会损失1秒的事务提交数 据。

3.6.2 Binlog日志

Binlog记录模式

Redo Log 是属于InnoDB引擎所特有的日志，而MySQL Server也有自己的日志，即 Binary log（二进制日志），简称Binlog。Binlog是记录所有数据库表结构变更以及表数据修改的二进制 日志，不会记录SELECT和SHOW这类操作。Binlog日志是以事件形式记录，还包含语句所执行的 消耗时间。开启Binlog日志有以下两个最重要的使用场景。

主从复制：在主库中开启Binlog功能，这样主库就可以把Binlog传递给从库，从库拿到 Binlog后实现数据恢复达到主从数据一致性。 数据恢复：通过mysqlbinlog工具来恢复数据。

Binlog文件名默认为“主机名_binlog-序列号”格式，例如oak_binlog-000001，也可以在配置文件 中指定名称。文件记录模式有STATEMENT、ROW和MIXED三种，具体含义如下。

ROW（row-based replication, RBR）：日志中会记录每一行数据被修改的情况，然后在 slave端对相同的数据进行修改。

优点：能清楚记录每一个行数据的修改细节，能完全实现主从数据同步和数据的恢复。

缺点：批量操作，会产生大量的日志，尤其是alter table会让日志暴涨。 STATMENT（statement-based replication, SBR）：每一条被修改数据的SQL都会记录到 master的Binlog中，slave在复制的时候SQL进程会解析成和原来master端执行过的相同的 SQL再次执行。简称SQL语句复制。 优点：日志量小，减少磁盘IO，提升存储和恢复速度

缺点：在某些情况下会导致主从数据不一致，比如last_insert_id()、now()等函数。

MIXED（mixed-based replication, MBR）：以上两种模式的混合使用，一般会使用 STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存 binlog，MySQL会根据执行的SQL语句选择写入模式。

Binlog文件结构 MySQL的binlog文件中记录的是对数据库的各种修改操作，用来表示修改操作的数据结构是Log event。不同的修改操作对应的不同的log event。比较常用的log event有：Query event、Row event、Xid event等。binlog文件的内容就是各种Log event的集合。 Binlog文件中Log event结构如下图所示：

Binlog写入机制

根据记录模式和操作触发event事件生成log event（事件触发执行机制） 将事务执行过程中产生log event写入缓冲区，每个事务线程都有一个缓冲区 Log Event保存在一个binlog_cache_mngr数据结构中，在该结构中有两个缓冲区，一个是 stmt_cache，用于存放不支持事务的信息；另一个是trx_cache，用于存放支持事务的信息。

事务在提交阶段会将产生的log event写入到外部binlog文件中。 不同事务以串行方式将log event写入binlog文件中，所以一个事务包含的log event信息在 binlog文件中是连续的，中间不会插入其他事务的log event。 Binlog文件操作 Binlog状态查看 show variables like 'log_bin';

开启Binlog功能

mysql> set global log_bin=mysqllogbin; ERROR 1238 (HY000): Variable 'log_bin' is a read only variable

需要修改my.cnf或my.ini配置文件，在[mysqld]下面增加log_bin=mysql_bin_log，重启 MySQL服务。

#log-bin=ON #log-bin-basename=mysqlbinlog binlog-format=ROW log-bin=mysqlbinlog

使用show binlog events命令

show binary logs; //等价于show master logs; show master status; show binlog events; show binlog events in 'mysqlbinlog.000001';

使用mysqlbinlog 命令

mysqlbinlog "文件名"

mysqlbinlog "文件名" > "test.sql"

使用 binlog 恢复数据

//按指定时间恢复 mysqlbinlog --start-datetime="2020-04-25 18:00:00" --stopdatetime="2020-04-26 00:00:00" mysqlbinlog.000002 | mysql -uroot -p1234 //按事件位置号恢复 mysqlbinlog --start-position=154 --stop-position=957 mysqlbinlog.000002 | mysql -uroot -p1234

mysqldump：定期全部备份数据库数据。mysqlbinlog可以做增量备份和恢复操作。

删除Binlog文件

purge binary logs to 'mysqlbinlog.000001'; //删除指定文件 purge binary logs before '2020-04-28 00:00:00'; //删除指定时间之前的文件 reset master; //清除所有文件

可以通过设置expire_logs_days参数来启动自动清理功能。默认值为0表示没启用。设置为1表示超 出1天binlog文件会自动删除掉。

Redo Log和Binlog区别

Redo Log是属于InnoDB引擎功能，Binlog是属于MySQL Server自带功能，并且是以二进制 文件记录。 Redo Log属于物理日志，记录该数据页更新状态内容，Binlog是逻辑日志，记录更新过程。 Redo Log日志是循环写，日志空间大小是固定，Binlog是追加写入，写完一个写下一个，不 会覆盖使用。 Redo Log作为服务器异常宕机后事务数据自动恢复使用，Binlog可以作为主从复制和数据恢 复使用。Binlog没有自动crash-safe能力。

第二部分 MySQL索引原理

第1节 索引类型

索引可以提升查询速度，会影响where查询，以及order by排序。MySQL索引类型如下：

从索引存储结构划分：B Tree索引、Hash索引、FULLTEXT全文索引、R Tree索引 从应用层次划分：普通索引、唯一索引、主键索引、复合索引 从索引键值类型划分：主键索引、辅助索引（二级索引） 从数据存储和索引键值逻辑关系划分：聚集索引（聚簇索引）、非聚集索引（非聚簇索引）

1.1 普通索引

这是最基本的索引类型，基于普通字段建立的索引，没有任何限制。

创建普通索引的方法如下：

CREATE INDEX <索引的名字> ON tablename (字段名); ALTER TABLE tablename ADD INDEX [索引的名字] (字段名); CREATE TABLE tablename ( [...], INDEX [索引的名字] (字段名) );

1.2 唯一索引

与"普通索引"类似，不同的就是：索引字段的值必须唯一，但允许有空值 。在创建或修改表时追加唯一 约束，就会自动创建对应的唯一索引。

创建唯一索引的方法如下：

CREATE UNIQUE INDEX <索引的名字> ON tablename (字段名); ALTER TABLE tablename ADD UNIQUE INDEX [索引的名字] (字段名); CREATE TABLE tablename ( [...], UNIQUE [索引的名字] (字段名) ;

1.3 主键索引

它是一种特殊的唯一索引，不允许有空值。在创建或修改表时追加主键约束即可，每个表只能有一个主 键。

创建主键索引的方法如下：

CREATE TABLE tablename ( [...], PRIMARY KEY (字段名) ); ALTER TABLE tablename ADD PRIMARY KEY (字段名);

1.4 复合索引

单一索引是指索引列为一列的情况，即新建索引的语句只实施在一列上；用户可以在多个列上建立索 引，这种索引叫做组复合索引（组合索引）。复合索引可以代替多个单一索引，相比多个单一索引复合 索引所需的开销更小。

索引同时有两个概念叫做窄索引和宽索引，窄索引是指索引列为1-2列的索引，宽索引也就是索引列超 过2列的索引，设计索引的一个重要原则就是能用窄索引不用宽索引，因为窄索引往往比组合索引更有 效。

创建组合索引的方法如下：

CREATE INDEX <索引的名字> ON tablename (字段名1，字段名2...); ALTER TABLE tablename ADD INDEX [索引的名字] (字段名1，字段名2...); CREATE TABLE tablename ( [...], INDEX [索引的名字] (字段名1，字段名2...) );

复合索引使用注意事项：

何时使用复合索引，要根据where条件建索引，注意不要过多使用索引，过多使用会对更新操作效 率有很大影响。 如果表已经建立了(col1，col2)，就没有必要再单独建立（col1）；如果现在有(col1)索引，如果查 询需要col1和col2条件，可以建立(col1,col2)复合索引，对于查询有一定提高。

1.5 全文索引

查询操作在数据量比较少时，可以使用like模糊查询，但是对于大量的文本数据检索，效率很低。如果 使用全文索引，查询速度会比like快很多倍。在MySQL 5.6 以前的版本，只有MyISAM存储引擎支持全 文索引，从MySQL 5.6开始MyISAM和InnoDB存储引擎均支持。

创建全文索引的方法如下：

CREATE FULLTEXT INDEX <索引的名字> ON tablename (字段名); ALTER TABLE tablename ADD FULLTEXT [索引的名字] (字段名); CREATE TABLE tablename ( [...], FULLTEXT KEY [索引的名字] (字段名) ;

和常用的like模糊查询不同，全文索引有自己的语法格式，使用 match 和 against 关键字，比如

select * from user where match(name) against('aaa');

全文索引使用注意事项：

全文索引必须在字符串、文本字段上建立。

全文索引字段值必须在最小字符和最大字符之间的才会有效。（innodb：3-84；myisam：484）

全文索引字段值要进行切词处理，按syntax字符进行切割，例如b+aaa，切分成b和aaa

全文索引匹配查询，默认使用的是等值匹配，例如a匹配a，不会匹配ab,ac。如果想匹配可以在布 尔模式下搜索a*

select * from user where match(name) against('a*' in boolean mode);

第2节 索引原理

MySQL官方对索引定义：是存储引擎用于快速查找记录的一种数据结构。需要额外开辟空间和数据维护 工作。

索引是物理数据页存储，在数据文件中（InnoDB，ibd文件），利用数据页(page)存储。 索引可以加快检索速度，但是同时也会降低增删改操作速度，索引维护需要代价。

索引涉及的理论知识：二分查找法、Hash和B+Tree。

2.1 二分查找法

二分查找法也叫作折半查找法，它是在有序数组中查找指定数据的搜索算法。它的优点是等值查询、范 围查询性能优秀，缺点是更新数据、新增数据、删除数据维护成本高。

首先定位left和right两个指针 计算(left+right)/2 判断除2后索引位置值与目标值的大小比对 索引位置值大于目标值就-1，right移动；如果小于目标值就+1，left移动

举个例子，下面的有序数组有17 个值，查找的目标值是7，过程如下：

第一次查找

第二次查找

第三次查找

第四次查找 2.2 Hash结构

Hash底层实现是由Hash表来实现的，是根据键值 <key,value> 存储数据的结构。非常适合根据key查找 value值，也就是单个key查询，或者说等值查询。其结构如下所示：

从上面结构可以看出，Hash索引可以方便的提供等值查询，但是对于范围查询就需要全表扫描了。 Hash索引在MySQL 中Hash结构主要应用在Memory原生的Hash索引 、InnoDB 自适应哈希索引。

InnoDB提供的自适应哈希索引功能强大，接下来重点描述下InnoDB 自适应哈希索引。

InnoDB自适应哈希索引是为了提升查询效率，InnoDB存储引擎会监控表上各个索引页的查询，当 InnoDB注意到某些索引值访问非常频繁时，会在内存中基于B+Tree索引再创建一个哈希索引，使得内 存中的 B+Tree 索引具备哈希索引的功能，即能够快速定值访问频繁访问的索引页。

InnoDB自适应哈希索引：在使用Hash索引访问时，一次性查找就能定位数据，等值查询效率要优于 B+Tree。

自适应哈希索引的建立使得InnoDB存储引擎能自动根据索引页访问的频率和模式自动地为某些热点页 建立哈希索引来加速访问。另外InnoDB自适应哈希索引的功能，用户只能选择开启或关闭功能，无法 进行人工干涉。

show engine innodb status \G; show variables like '%innodb_adaptive%'; 2.3 B+Tree结构

MySQL数据库索引采用的是B+Tree结构，在B-Tree结构上做了优化改造。

B-Tree结构

索引值和data数据分布在整棵树结构中 每个节点可以存放多个索引值及对应的data数据 树节点中的多个索引值从左到右升序排列

B树的搜索：从根节点开始，对节点内的索引值序列采用二分法查找，如果命中就结束查找。没有 命中会进入子节点重复查找过程，直到所对应的的节点指针为空，或已经是叶子节点了才结束。

B+Tree结构

非叶子节点不存储data数据，只存储索引值，这样便于存储更多的索引值 叶子节点包含了所有的索引值和data数据 叶子节点用指针连接，提高区间的访问性能

相比B树，B+树进行范围查找时，只需要查找定位两个节点的索引值，然后利用叶子节点的指针进 行遍历即可。而B树需要遍历范围内所有的节点和数据，显然B+Tree效率高。

2.4 聚簇索引和辅助索引

聚簇索引和非聚簇索引：B+Tree的叶子节点存放主键索引值和行记录就属于聚簇索引；如果索引值和行 记录分开存放就属于非聚簇索引。

主键索引和辅助索引：B+Tree的叶子节点存放的是主键字段值就属于主键索引；如果存放的是非主键值 就属于辅助索引（二级索引）。

在InnoDB引擎中，主键索引采用的就是聚簇索引结构存储。

聚簇索引（聚集索引） 聚簇索引是一种数据存储方式，InnoDB的聚簇索引就是按照主键顺序构建 B+Tree结构。B+Tree 的叶子节点就是行记录，行记录和主键值紧凑地存储在一起。 这也意味着 InnoDB 的主键索引就 是数据表本身，它按主键顺序存放了整张表的数据，占用的空间就是整个表数据量的大小。通常说 的主键索引就是聚集索引。

InnoDB的表要求必须要有聚簇索引：

如果表定义了主键，则主键索引就是聚簇索引 如果表没有定义主键，则第一个非空unique列作为聚簇索引 否则InnoDB会从建一个隐藏的row-id作为聚簇索引 辅助索引

InnoDB辅助索引，也叫作二级索引，是根据索引列构建 B+Tree结构。但在 B+Tree 的叶子节点中 只存了索引列和主键的信息。二级索引占用的空间会比聚簇索引小很多， 通常创建辅助索引就是 为了提升查询效率。一个表InnoDB只能创建一个聚簇索引，但可以创建多个辅助索引。

非聚簇索引

与InnoDB表存储不同，MyISAM数据表的索引文件和数据文件是分开的，被称为非聚簇索引结 构。 第3节 索引分析与优化

3.1 EXPLAIN

MySQL 提供了一个 EXPLAIN 命令，它可以对 SELECT 语句进行分析，并输出 SELECT 执行的详细信 息，供开发人员有针对性的优化。例如：

EXPLAIN SELECT * from user WHERE id < 3;

EXPLAIN 命令的输出内容大致如下：

select_type

表示查询的类型。常用的值如下：

SIMPLE ： 表示查询语句不包含子查询或union PRIMARY：表示此查询是最外层的查询 UNION：表示此查询是UNION的第二个或后续的查询 DEPENDENT UNION：UNION中的第二个或后续的查询语句，使用了外面查询结果 UNION RESULT：UNION的结果 SUBQUERY：SELECT子查询语句 DEPENDENT SUBQUERY：SELECT子查询语句依赖外层查询的结果。 最常见的查询类型是SIMPLE，表示我们的查询没有子查询也没用到UNION查询。 type 表示存储引擎查询数据时采用的方式。比较重要的一个属性，通过它可以判断出查询是全表扫描还 是基于索引的部分扫描。常用属性值如下，从上至下效率依次增强。

ALL：表示全表扫描，性能最差。 index：表示基于索引的全表扫描，先扫描索引再扫描全表数据。 range：表示使用索引范围查询。使用>、>=、<、<=、in等等。 ref：表示使用非唯一索引进行单值查询。 eq_ref：一般情况下出现在多表join查询，表示前面表的每一个记录，都只能匹配后面表的一 行结果。 const：表示使用主键或唯一索引做等值查询，常量查询。 NULL：表示不用访问表，速度最快。 possible_keys

表示查询时能够使用到的索引。注意并不一定会真正使用，显示的是索引名称。

key 表示查询时真正使用到的索引，显示的是索引名称。

rows MySQL查询优化器会根据统计信息，估算SQL要查询到结果需要扫描多少行记录。原则上rows是 越少效率越高，可以直观的了解到SQL效率高低。

key_len 表示查询使用了索引的字节数量。可以判断是否全部使用了组合索引。

key_len的计算规则如下：

字符串类型 字符串长度跟字符集有关：latin1=1、gbk=2、utf8=3、utf8mb4=4

char(n)：n*字符集长度 varchar(n)：n * 字符集长度 + 2字节 数值类型 TINYINT：1个字节 SMALLINT：2个字节 MEDIUMINT：3个字节 INT、FLOAT：4个字节 BIGINT、DOUBLE：8个字节 时间类型 DATE：3个字节 TIMESTAMP：4个字节 DATETIME：8个字节 字段属性 NULL属性占用1个字节，如果一个字段设置了NOT NULL，则没有此项。 Extra

Extra表示很多额外的信息，各种操作会在Extra提示相关信息，常见几种如下：

Using where

表示查询需要通过索引回表查询数据。

Using index

表示查询需要通过索引，索引就可以满足所需数据。

Using ﬁlesort

表示查询出来的结果需要额外排序，数据量小在内存，大的话在磁盘，因此有Using ﬁlesort 建议优化。

Using temprorary

查询使用到了临时表，一般出现于去重、分组等操作。

3.2 回表查询

在之前介绍过，InnoDB索引有聚簇索引和辅助索引。聚簇索引的叶子节点存储行记录，InnoDB必须要 有，且只有一个。辅助索引的叶子节点存储的是主键值和索引字段值，通过辅助索引无法直接定位行记 录，通常情况下，需要扫码两遍索引树。先通过辅助索引定位主键值，然后再通过聚簇索引定位行记 录，这就叫做回表查询，它的性能比扫一遍索引树低。

总结：通过索引查询主键值，然后再去聚簇索引查询记录信息

3.3 覆盖索引

在SQL-Server官网的介绍如下：

在MySQL官网，类似的说法出现在explain查询计划优化章节，即explain的输出结果Extra字段为Using index时，能够触发索引覆盖。

不管是SQL-Server官网，还是MySQL官网，都表达了：只需要在一棵索引树上就能获取SQL所需的所 有列数据，无需回表，速度更快，这就叫做索引覆盖。

实现索引覆盖最常见的方法就是：将被查询的字段，建立到组合索引。

3.4 最左前缀原则

复合索引使用时遵循最左前缀原则，最左前缀顾名思义，就是最左优先，即查询中使用到最左边的列， 那么查询就会使用到索引，如果从索引的第二列开始查找，索引将失效。 3.5 LIKE查询

面试题：MySQL在使用like模糊查询时，索引能不能起作用？

回答：MySQL在使用Like模糊查询时，索引是可以被使用的，只有把%字符写在后面才会使用到索引。

select * from user where name like '%o%'; //不起作用 select * from user where name like 'o%'; //起作用 select * from user where name like '%o'; //不起作用

3.6 NULL查询

面试题：如果MySQL表的某一列含有NULL值，那么包含该列的索引是否有效？

对MySQL来说，NULL是一个特殊的值，从概念上讲，NULL意味着“一个未知值”，它的处理方式与其他 值有些不同。比如：不能使用=，<，>这样的运算符，对NULL做算术运算的结果都是NULL，count时 不会包括NULL行等，NULL比空字符串需要更多的存储空间等。

“NULL columns require additional space in the row to record whether their values are NULL. For MyISAM tables, each NULL column takes one bit extra, rounded up to the nearest byte.”

NULL列需要增加额外空间来记录其值是否为NULL。对于MyISAM表，每一个空列额外占用一位，四舍 五入到最接近的字节。

虽然MySQL可以在含有NULL的列上使用索引，但NULL和其他数据还是有区别的，不建议列上允许为 NULL。最好设置NOT NULL，并给一个默认值，比如0和 ‘’ 空字符串等，如果是datetime类型，也可以 设置系统当前时间或某个固定的特殊值，例如'1970-01-01 00:00:00'。 3.7 索引与排序

MySQL查询支持ﬁlesort和index两种方式的排序，ﬁlesort是先把结果查出，然后在缓存或磁盘进行排序 操作，效率较低。使用index是指利用索引自动实现排序，不需另做排序操作，效率会比较高。

ﬁlesort有两种排序算法：双路排序和单路排序。

双路排序：需要两次磁盘扫描读取，最终得到用户数据。第一次将排序字段读取出来，然后排序；第二 次去读取其他字段数据。

单路排序：从磁盘查询所需的所有列数据，然后在内存排序将结果返回。如果查询数据超出缓存 sort_buﬀer，会导致多次磁盘读取操作，并创建临时表，最后产生了多次IO，反而会增加负担。解决方 案：少使用select *；增加sort_buﬀer_size容量和max_length_for_sort_data容量。

如果我们Explain分析SQL，结果中Extra属性显示Using ﬁlesort，表示使用了ﬁlesort排序方式，需要优 化。如果Extra属性显示Using index时，表示覆盖索引，也表示所有操作在索引上完成，也可以使用 index排序方式，建议大家尽可能采用覆盖索引。

以下几种情况，会使用index方式的排序。

ORDER BY 子句索引列组合满足索引最左前列

explain select id from user order by id;

//对应(id)、(id,name)索引有效

WHERE子句+ORDER BY子句索引列组合满足索引最左前列

explain select id from user where age=18 order by name; (age,name)索引

//对应

以下几种情况，会使用ﬁlesort方式的排序。

对索引列同时使用了ASC和DESC

explain select id from user order by age asc,name desc; (age,name)索引

//对应

WHERE子句和ORDER BY子句满足最左前缀，但where子句使用了范围查询（例如>、<、in 等）

explain select id from user where age>10 order by name; (age,name)索引

//对应

ORDER BY或者WHERE+ORDER BY索引列没有满足索引最左前列

explain select id from user order by name;

//对应(age,name)索引

使用了不同的索引，MySQL每次只采用一个索引，ORDER BY涉及了两个索引

explain select id from user order by name,age; 引

//对应(name)、(age)两个索

WHERE子句与ORDER BY子句，使用了不同的索引

explain select id from user where name='tom' order by age; //对应 (name)、(age)索引 WHERE子句或者ORDER BY子句中索引列使用了表达式，包括函数表达式

explain select id from user order by abs(age);

//对应(age)索引

第4节 查询优化

4.1 慢查询定位

开启慢查询日志

查看 MySQL 数据库是否开启了慢查询日志和慢查询日志文件的存储位置的命令如下：

SHOW VARIABLES LIKE 'slow_query_log%'

通过如下命令开启慢查询日志：

SET global slow_query_log = ON; SET global slow_query_log_file = 'OAK-slow.log'; SET global log_queries_not_using_indexes = ON; SET long_query_time = 10;

long_query_time：指定慢查询的阀值，单位秒。如果SQL执行时间超过阀值，就属于慢查询 记录到日志文件中。 log_queries_not_using_indexes：表示会记录没有使用索引的查询SQL。前提是slow_query_log 的值为ON，否则不会奏效。

查看慢查询日志

文本方式查看

直接使用文本编辑器打开slow.log日志即可。

time：日志记录的时间 User@Host：执行的用户及主机 Query_time：执行的时间 Lock_time：锁表时间 Rows_sent：发送给请求方的记录数，结果数量 Rows_examined：语句扫描的记录条数 SET timestamp：语句执行的时间点 select....：执行的具体的SQL语句 使用mysqldumpslow查看

MySQL 提供了一个慢查询日志分析工具mysqldumpslow，可以通过该工具分析慢查询日志 内容。

在 MySQL bin目录下执行下面命令可以查看该使用格式。

perl mysqldumpslow.pl --help

运行如下命令查看慢查询日志信息： perl mysqldumpslow.pl -t 5 -s at C:\ProgramData\MySQL\Data\OAK-slow.log

除了使用mysqldumpslow工具，也可以使用第三方分析工具，比如pt-query-digest、 mysqlsla等。

4.2 慢查询优化

索引和慢查询

如何判断是否为慢查询？

MySQL判断一条语句是否为慢查询语句，主要依据SQL语句的执行时间，它把当前语句的执 行时间跟 long_query_time 参数做比较，如果语句的执行时间 > long_query_time，就会把 这条执行语句记录到慢查询日志里面。long_query_time 参数的默认值是 10s，该参数值可 以根据自己的业务需要进行调整。

如何判断是否应用了索引？

SQL语句是否使用了索引，可根据SQL语句执行过程中有没有用到表的索引，可通过 explain 命令分析查看，检查结果中的 key 值，是否为NULL。

应用了索引是否一定快？

下面我们来看看下面语句的 explain 的结果，你觉得这条语句有用上索引吗？比如

select * from user where id>0;

虽然使用了索引，但是还是从主键索引的最左边的叶节点开始向右扫描整个索引树，进行了 全表扫描，此时索引就失去了意义。

而像 select * from user where id = 2; 这样的语句，才是我们平时说的使用了索引。它表示 的意思是，我们使用了索引的快速搜索功能，并且有效地减少了扫描行数。

查询是否使用索引，只是表示一个SQL语句的执行过程；而是否为慢查询，是由它执行的时间决定 的，也就是说是否使用了索引和是否是慢查询两者之间没有必然的联系。

我们在使用索引时，不要只关注是否起作用，应该关心索引是否减少了查询扫描的数据行数，如果 扫描行数减少了，效率才会得到提升。对于一个大表，不止要创建索引，还要考虑索引过滤性，过 滤性好，执行速度才会快。

提高索引过滤性

假如有一个5000万记录的用户表，通过sex='男'索引过滤后，还需要定位3000万，SQL执行速度也 不会很快。其实这个问题涉及到索引的过滤性，比如1万条记录利用索引过滤后定位10条、100 条、1000条，那他们过滤性是不同的。索引过滤性与索引字段、表的数据量、表设计结构都有关 系。

下面我们看一个案例：

表：student 字段：id,name,sex,age 造数据：insert into student (name,sex,age) select name,sex,age from student; SQL案例：select * from student where age=18 and name like '张%';（全表扫 描）

优化1

alter table student add index(name); //追加name索引 - 优化2

alter table student add index(age,name); //追加age,name索引

优化3

可以看到，index condition pushdown 优化的效果还是很不错的。再进一步优化，我们可以把名 字的第一个字和年龄做一个联合索引，这里可以使用 MySQL 5.7 引入的虚拟列来实现。

//为user表添加first_name虚拟列，以及联合索引(first_name,age) alter table student add first_name varchar(2) generated always as (left(name, 1)), add index(first_name, age);

explain select * from student where first_name='张' and age=18;

慢查询原因总结

全表扫描：explain分析type属性all 全索引扫描：explain分析type属性index 索引过滤性不好：靠索引字段选型、数据量和状态、表设计 频繁的回表查询开销：尽量少用select *，使用覆盖索引

4.3 分页查询优化

一般性分页

般的分页查询使用简单的 limit 子句就可以实现。limit格式如下：

SELECT * FROM 表名 LIMIT [offset,] rows

第一个参数指定第一个返回记录行的偏移量，注意从0开始； 第二个参数指定返回记录行的最大数目； 如果只给定一个参数，它表示返回最大的记录行数目；

思考1：如果偏移量固定，返回记录量对执行时间有什么影响？

select * from user limit 10000,1; select * from user limit 10000,10; select * from user limit 10000,100; select * from user limit 10000,1000; select * from user limit 10000,10000;

结果：在查询记录时，返回记录量低于100条，查询时间基本没有变化，差距不大。随着查询记录 量越大，所花费的时间也会越来越多。

思考2：如果查询偏移量变化，返回记录数固定对执行时间有什么影响？ select * from user limit 1,100; select * from user limit 10,100; select * from user limit 100,100; select * from user limit 1000,100; select * from user limit 10000,100;

结果：在查询记录时，如果查询记录量相同，偏移量超过100后就开始随着偏移量增大，查询时间 急剧的增加。（这种分页查询机制，每次都会从数据库第一条记录开始扫描，越往后查询越慢，而 且查询的数据越多，也会拖慢总查询速度。）

分页优化方案

第一步：利用覆盖索引优化

select * from user limit 10000,100; select id from user limit 10000,100;

第二步：利用子查询优化

select * from user limit 10000,100; select * from user where id>= (select id from user limit 10000,1) limit 100;

原因：使用了id做主键比较(id>=)，并且子查询使用了覆盖索引进行优化。

第三部分 MySQL事务和锁

第1节 ACID 特性

在关系型数据库管理系统中，一个逻辑工作单元要成为事务，必须满足这 4 个特性，即所谓的 ACID： 原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability）。

1.1 原子性

原子性：事务是一个原子操作单元，其对数据的修改，要么全都执行，要么全都不执行。

修改---》Buﬀer Pool修改---》刷盘。可能会有下面两种情况：

事务提交了，如果此时Buﬀer Pool的脏页没有刷盘，如何保证修改的数据生效？ Redo 如果事务没提交，但是Buﬀer Pool的脏页刷盘了，如何保证不该存在的数据撤销？Undo

每一个写事务，都会修改BuﬀerPool，从而产生相应的Redo/Undo日志，在Buﬀer Pool 中的页被刷到 磁盘之前，这些日志信息都会先写入到日志文件中，如果 Buﬀer Pool 中的脏页没有刷成功，此时数据 库挂了，那在数据库再次启动之后，可以通过 Redo 日志将其恢复出来，以保证脏页写的数据不会丢 失。如果脏页刷新成功，此时数据库挂了，就需要通过Undo来实现了。

1.2 持久性

持久性：指的是一个事务一旦提交，它对数据库中数据的改变就应该是永久性的，后续的操作或故障不 应该对其有任何影响，不会丢失。

如下图所示，一个“提交”动作触发的操作有：binlog落地、发送binlog、存储引擎提交、ﬂush_logs， check_point、事务提交标记等。这些都是数据库保证其数据完整性、持久性的手段。 MySQL的持久性也与WAL技术相关，redo log在系统Crash重启之类的情况时，可以修复数据，从而保 障事务的持久性。通过原子性可以保证逻辑上的持久性，通过存储引擎的数据刷盘可以保证物理上的持 久性。

1.3 隔离性

隔离性：指的是一个事务的执行不能被其他事务干扰，即一个事务内部的操作及使用的数据对其他的并 发事务是隔离的。

InnoDB 支持的隔离性有 4 种，隔离性从低到高分别为：读未提交、读提交、可重复读、可串行化。锁 和多版本控制（MVCC）技术就是用于保障隔离性的（后面课程详解）。

1.4 一致性

一致性：指的是事务开始之前和事务结束之后，数据库的完整性限制未被破坏。一致性包括两方面的内 容，分别是约束一致性和数据一致性。

约束一致性：创建表结构时所指定的外键、Check、唯一索引等约束，可惜在 MySQL 中不支持 Check 。 数据一致性：是一个综合性的规定，因为它是由原子性、持久性、隔离性共同保证的结果，而不是 单单依赖于某一种技术。

一致性也可以理解为数据的完整性。数据的完整性是通过原子性、隔离性、持久性来保证的，而这3个 特性又是通过 Redo/Undo 来保证的。逻辑上的一致性，包括唯一索引、外键约束、check 约束，这属 于业务逻辑范畴。

ACID 及它们之间的关系如下图所示，4个特性中有3个与 WAL 有关系，都需要通过 Redo、Undo 日志

来保证等。

WAL的全称为Write-Ahead Logging，先写日志，再写磁盘。 第2节 事务控制的演进

2.1 并发事务

事务并发处理可能会带来一些问题，比如：更新丢失、脏读、不可重复读、幻读等。

更新丢失 当两个或多个事务更新同一行记录，会产生更新丢失现象。可以分为回滚覆盖和提交覆盖。

回滚覆盖：一个事务回滚操作，把其他事务已提交的数据给覆盖了。

提交覆盖：一个事务提交操作，把其他事务已提交的数据给覆盖了。 脏读 一个事务读取到了另一个事务修改但未提交的数据。 不可重复读 一个事务中多次读取同一行记录不一致，后面读取的跟前面读取的不一致。 幻读 一个事务中多次按相同条件查询，结果不一致。后续查询的结果和面前查询结果不同，多了或少了 几行记录。

2.3 排队

最简单的方法，就是完全顺序执行所有事务的数据库操作，不需要加锁，简单的说就是全局排队。序列 化执行所有的事务单元，数据库某个时刻只处理一个事务操作，特点是强一致性，处理性能低。

2.2 排他锁 引入锁之后就可以支持并发处理事务，如果事务之间涉及到相同的数据项时，会使用排他锁，或叫互斥 锁，先进入的事务独占数据项以后，其他事务被阻塞，等待前面的事务释放锁。

注意，在整个事务1结束之前，锁是不会被释放的，所以，事务2必须等到事务1结束之后开始。

2.3 读写锁

读和写操作：读读、写写、读写、写读。

读写锁就是进一步细化锁的颗粒度，区分读操作和写操作，让读和读之间不加锁，这样下面的两个事务 就可以同时被执行了。

读写锁，可以让读和读并行，而读和写、写和读、写和写这几种之间还是要加排他锁。

2.4 MVCC

多版本控制MVCC，也就是Copy on Write的思想。MVCC除了支持读和读并行，还支持读和写、写和读 的并行，但为了保证一致性，写和写是无法并行的。 在事务1开始写操作的时候会copy一个记录的副本，其他事务读操作会读取这个记录副本，因此不会影 响其他事务对此记录的读取，实现写和读并行。

一、MVCC概念

MVCC（Multi Version Concurrency Control）被称为多版本控制，是指在数据库中为了实现高并发的 数据访问，对数据进行多版本处理，并通过事务的可见性来保证事务能看到自己应该看到的数据版本。 多版本控制很巧妙地将稀缺资源的独占互斥转换为并发，大大提高了数据库的吞吐量及读写性能。

如何生成的多版本？每次事务修改操作之前，都会在Undo日志中记录修改之前的数据状态和事务号， 该备份记录可以用于其他事务的读取，也可以进行必要时的数据回滚。

二、MVCC实现原理

MVCC最大的好处是读不加锁，读写不冲突。在读多写少的系统应用中，读写不冲突是非常重要的，极 大的提升系统的并发性能，这也是为什么现阶段几乎所有的关系型数据库都支持 MVCC 的原因，不过目 前MVCC只在 Read Commited 和 Repeatable Read 两种隔离级别下工作。

在 MVCC 并发控制中，读操作可以分为两类: 快照读（Snapshot Read）与当前读 （Current Read）。 快照读：读取的是记录的快照版本（有可能是历史版本），不用加锁。（select） 当前读：读取的是记录的最新版本，并且当前读返回的记录，都会加锁，保证其他事务不会再并发 修改这条记录。（select... for update 或lock in share mode，insert/delete/update）

为了让大家更直观地理解 MVCC 的实现原理，举一个记录更新的案例来讲解 MVCC 中多版本的实现。

假设 F1～F6 是表中字段的名字，1～6 是其对应的数据。后面三个隐含字段分别对应该行的隐含ID、事 务号和回滚指针，如下图所示。 具体的更新过程如下：

假如一条数据是刚 INSERT 的，DB_ROW_ID 为 1，其他两个字段为空。当事务 1 更改该行的数据值 时，会进行如下操作，如下图所示。

用排他锁锁定该行；记录 Redo log； 把该行修改前的值复制到 Undo log，即图中下面的行； 修改当前行的值，填写事务编号，使回滚指针指向 Undo log 中修改前的行。

接下来事务2操作，过程与事务 1 相同，此时 Undo log 中会有两行记录，并且通过回滚指针连在一 起，通过当前记录的回滚指针回溯到该行创建时的初始内容，如下图所示。

MVCC已经实现了读读、读写、写读并发处理，如果想进一步解决写写冲突，可以采用下面两种方案：

乐观锁 悲观锁

第3节 事务隔离级别

3.1 隔离级别类型

前面提到的“更新丢失”、”脏读”、“不可重复读”和“幻读”等并发事务问题，其实都是数据库一致性问题， 为了解决这些问题，MySQL数据库是通过事务隔离级别来解决的，数据库系统提供了以下 4 种事务隔 离级别供用户选择。 读未提交

Read Uncommitted 读未提交：解决了回滚覆盖类型的更新丢失，但可能发生脏读现象，也就是 可能读取到其他会话中未提交事务修改的数据。

已提交读

Read Committed 读已提交：只能读取到其他会话中已经提交的数据，解决了脏读。但可能发生 不可重复读现象，也就是可能在一个事务中两次查询结果不一致。

可重复度

Repeatable Read 可重复读：解决了不可重复读，它确保同一事务的多个实例在并发读取数据 时，会看到同样的数据行。不过理论上会出现幻读，简单的说幻读指的的当用户读取某一范围的数 据行时，另一个事务又在该范围插入了新行，当用户在读取该范围的数据时会发现有新的幻影行。

可串行化

Serializable 串行化：所有的增删改查串行执行。它通过强制事务排序，解决相互冲突，从而解决 幻度的问题。这个级别可能导致大量的超时现象的和锁竞争，效率低下。

数据库的事务隔离级别越高，并发问题就越小，但是并发处理能力越差（代价）。读未提交隔离级别最 低，并发问题多，但是并发处理能力好。以后使用时，可以根据系统特点来选择一个合适的隔离级别， 比如对不可重复读和幻读并不敏感，更多关心数据库并发处理能力，此时可以使用Read Commited隔 离级别。

事务隔离级别，针对Innodb引擎，支持事务的功能。像MyISAM引擎没有关系。

事务隔离级别和锁的关系

1）事务隔离级别是SQL92定制的标准，相当于事务并发控制的整体解决方案，本质上是对锁和MVCC使 用的封装，隐藏了底层细节。

2）锁是数据库实现并发控制的基础，事务隔离性是采用锁来实现，对相应操作加不同的锁，就可以防 止其他事务同时对数据进行读写操作。

3）对用户来讲，首先选择使用隔离级别，当选用的隔离级别不能解决并发问题或需求时，才有必要在 开发中手动的设置锁。

MySQL默认隔离级别：可重复读 Oracle、SQLServer默认隔离级别：读已提交 一般使用时，建议采用默认隔离级别，然后存在的一些并发问题，可以通过悲观锁、乐观锁等实现处 理。

3.2 MySQL隔离级别控制

MySQL默认的事务隔离级别是Repeatable Read，查看MySQL当前数据库的事务隔离级别命令如下：

show variables like 'tx_isolation';

或 select @@tx_isolation;

设置事务隔离级别可以如下命令：

set tx_isolation='READ-UNCOMMITTED'; set tx_isolation='READ-COMMITTED'; set tx_isolation='REPEATABLE-READ'; set tx_isolation='SERIALIZABLE';

第4节 锁机制和实战

4.1 锁分类

在 MySQL中锁有很多不同的分类。

从操作的粒度可分为表级锁、行级锁和页级锁。

表级锁：每次操作锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低。应用在 MyISAM、InnoDB、BDB 等存储引擎中。 行级锁：每次操作锁住一行数据。锁定粒度最小，发生锁冲突的概率最低，并发度最高。应 用在InnoDB 存储引擎中。 页级锁：每次锁定相邻的一组记录，锁定粒度界于表锁和行锁之间，开销和加锁时间界于表 锁和行锁之间，并发度一般。应用在BDB 存储引擎中。

从操作的类型可分为读锁和写锁。

读锁（S锁）：共享锁，针对同一份数据，多个读操作可以同时进行而不会互相影响。 写锁（X锁）：排他锁，当前写操作没有完成前，它会阻断其他写锁和读锁。

IS锁、IX锁：意向读锁、意向写锁，属于表级锁，S和X主要针对行级锁。在对表记录添加S或X锁之 前，会先对表添加IS或IX锁。

S锁：事务A对记录添加了S锁，可以对记录进行读操作，不能做修改，其他事务可以对该记录追加 S锁，但是不能追加X锁，需要追加X锁，需要等记录的S锁全部释放。

X锁：事务A对记录添加了X锁，可以对记录进行读和修改操作，其他事务不能对记录做读和修改操 作。

从操作的性能可分为乐观锁和悲观锁。

乐观锁：一般的实现方式是对记录数据版本进行比对，在数据更新提交的时候才会进行冲突 检测，如果发现冲突了，则提示错误信息。 悲观锁：在对一条数据修改的时候，为了避免同时被其他人修改，在修改数据之前先锁定， 再修改的控制方式。共享锁和排他锁是悲观锁的不同实现，但都属于悲观锁范畴。

4.2 行锁原理

在InnoDB引擎中，我们可以使用行锁和表锁，其中行锁又分为共享锁和排他锁。InnoDB行锁是通过对 索引数据页上的记录加锁实现的，主要实现算法有 3 种：Record Lock、Gap Lock 和 Next-key Lock。 RecordLock锁：锁定单个行记录的锁。（记录锁，RC、RR隔离级别都支持） GapLock锁：间隙锁，锁定索引记录间隙，确保索引记录的间隙不变。（范围锁，RR隔离级别支 持） Next-key Lock 锁：记录锁和间隙锁组合，同时锁住数据，并且锁住数据前后范围。（记录锁+范 围锁，RR隔离级别支持）

在RR隔离级别，InnoDB对于记录加锁行为都是先采用Next-Key Lock，但是当SQL操作含有唯一索引 时，Innodb会对Next-Key Lock进行优化，降级为RecordLock，仅锁住索引本身而非范围。

1）select ... from 语句：InnoDB引擎采用MVCC机制实现非阻塞读，所以对于普通的select语句， InnoDB不加锁

2）select ... from lock in share mode语句：追加了共享锁，InnoDB会使用Next-Key Lock锁进行处 理，如果扫描发现唯一索引，可以降级为RecordLock锁。

3）select ... from for update语句：追加了排他锁，InnoDB会使用Next-Key Lock锁进行处理，如果扫 描发现唯一索引，可以降级为RecordLock锁。

4）update ... where 语句：InnoDB会使用Next-Key Lock锁进行处理，如果扫描发现唯一索引，可以 降级为RecordLock锁。

5）delete ... where 语句：InnoDB会使用Next-Key Lock锁进行处理，如果扫描发现唯一索引，可以降 级为RecordLock锁。

6）insert语句：InnoDB会在将要插入的那一行设置一个排他的RecordLock锁。

下面以“update t1 set name=‘XX’ where id=10”操作为例，举例子分析下 InnoDB 对不同索引的加锁行 为，以RR隔离级别为例。

主键加锁

加锁行为：仅在id=10的主键索引记录上加X锁。

唯一键加锁 加锁行为：现在唯一索引id上加X锁，然后在id=10的主键索引记录上加X锁。

非唯一键加锁

加锁行为：对满足id=10条件的记录和主键分别加X锁，然后在(6,c)-(10,b)、(10,b)-(10,d)、(10,d)(11,f)范围分别加Gap Lock。

无索引加锁 加锁行为：表里所有行和间隙都会加X锁。（当没有索引时，会导致全表锁定，因为InnoDB引擎 锁机制是基于索引实现的记录锁定）。

4.3 悲观锁

悲观锁（Pessimistic Locking），是指在数据处理过程，将数据处于锁定状态，一般使用数据库的锁机 制实现。从广义上来讲，前面提到的行锁、表锁、读锁、写锁、共享锁、排他锁等，这些都属于悲观锁 范畴。

表级锁

表级锁每次操作都锁住整张表，并发度最低。常用命令如下：

手动增加表锁

lock table 表名称 read|write,表名称2 read|write;

查看表上加过的锁

show open tables;

删除表锁

unlock tables;

表级读锁：当前表追加read锁，当前连接和其他的连接都可以读操作；但是当前连接增删改操作 会报错，其他连接增删改会被阻塞。 表级写锁：当前表追加write锁，当前连接可以对表做增删改查操作，其他连接对该表所有操作都 被阻塞（包括查询）。

总结：表级读锁会阻塞写操作，但是不会阻塞读操作。而写锁则会把读和写操作都阻塞。

共享锁（行级锁-读锁）

共享锁又称为读锁，简称S锁。共享锁就是多个事务对于同一数据可以共享一把锁，都能访问到数 据，但是只能读不能修改。使用共享锁的方法是在select ... lock in share mode，只适用查询语 句。

总结：事务使用了共享锁（读锁），只能读取，不能修改，修改操作被阻塞。

排他锁（行级锁-写锁）

排他锁又称为写锁，简称X锁。排他锁就是不能与其他锁并存，如一个事务获取了一个数据行的排 他锁，其他事务就不能对该行记录做其他操作，也不能获取该行的锁。

使用排他锁的方法是在SQL末尾加上for update，innodb引擎默认会在update，delete语句加上 for update。行级锁的实现其实是依靠其对应的索引，所以如果操作没用到索引的查询，那么会锁 住全表记录。

总结：事务使用了排他锁（写锁），当前事务可以读取和修改，其他事务不能修改，也不能获取记录 锁（select... for update）。如果查询没有使用到索引，将会锁住整个表记录。

4.4 乐观锁

乐观锁是相对于悲观锁而言的，它不是数据库提供的功能，需要开发者自己去实现。在数据库操作时， 想法很乐观，认为这次的操作不会导致冲突，因此在数据库操作时并不做任何的特殊处理，即不加锁， 而是在进行事务提交时再去判断是否有冲突了。

乐观锁实现的关键点：冲突的检测。

悲观锁和乐观锁都可以解决事务写写并发，在应用中可以根据并发处理能力选择区分，比如对并发率要 求高的选择乐观锁；对于并发率要求低的可以选择悲观锁。

乐观锁实现原理

使用版本字段（version）

先给数据表增加一个版本(version) 字段，每操作一次，将那条记录的版本号加 1。version 是用来查看被读的记录有无变化，作用是防止记录在业务处理期间被其他事务修改。 使用时间戳（Timestamp）

与使用version版本字段相似，同样需要给在数据表增加一个字段，字段类型使用timestamp 时间戳。也是在更新提交的时候检查当前数据库中数据的时间戳和自己更新前取到的时间戳 进行对比，如果一致则提交更新，否则就是版本冲突，取消操作。

乐观锁案例

下面我们使用下单过程作为案例，描述下乐观锁的使用。

第一步：查询商品信息

select (quantity,version) from products where id=1;

第二部：根据商品信息生成订单

insert into orders ... insert into items ...

第三部：修改商品库存

update products set quantity=quantity-1,version=version+1 where id=1 and version=#{version};

除了自己手动实现乐观锁之外，许多数据库访问框架也封装了乐观锁的实现，比如 hibernate框架。MyBatis框架大家可以使用OptimisticLocker插件来扩展。

4.5 死锁与解决方案

下面介绍几种常见的死锁现象和解决方案：

一、表锁死锁

产生原因： 用户A访问表A（锁住了表A），然后又访问表B；另一个用户B访问表B（锁住了表B），然后企图 访问表A；这时用户A由于用户B已经锁住表B，它必须等待用户B释放表B才能继续，同样用户B要 等用户A释放表A才能继续，这就死锁就产生了。

用户A--》A表（表锁）--》B表（表锁）

用户B--》B表（表锁）--》A表（表锁）

解决方案：

这种死锁比较常见，是由于程序的BUG产生的，除了调整的程序的逻辑没有其它的办法。仔细分 析程序的逻辑，对于数据库的多表操作时，尽量按照相同的顺序进行处理，尽量避免同时锁定两个 资源，如操作A和B两张表时，总是按先A后B的顺序处理， 必须同时锁定两个资源时，要保证在任 何时刻都应该按照相同的顺序来锁定资源。

二、行级锁死锁

产生原因1：

如果在事务中执行了一条没有索引条件的查询，引发全表扫描，把行级锁上升为全表记录锁定（等 价于表级锁），多个这样的事务执行后，就很容易产生死锁和阻塞，最终应用系统会越来越慢，发 生阻塞或死锁。

解决方案1：

SQL语句中不要使用太复杂的关联多表的查询；使用explain“执行计划"对SQL语句进行分析，对于 有全表扫描和全表锁定的SQL语句，建立相应的索引进行优化。

产生原因2：

两个事务分别想拿到对方持有的锁，互相等待，于是产生死锁。

解决方案2：

在同一个事务中，尽可能做到一次锁定所需要的所有资源 按照id对资源排序，然后按顺序进行处理 三、共享锁转换为排他锁 产生原因：

事务A 查询一条纪录，然后更新该条纪录；此时事务B 也更新该条纪录，这时事务B 的排他锁由于 事务A 有共享锁，必须等A 释放共享锁后才可以获取，只能排队等待。事务A 再执行更新操作时， 此处发生死锁，因为事务A 需要排他锁来做更新操作。但是，无法授予该锁请求，因为事务B 已经 有一个排他锁请求，并且正在等待事务A 释放其共享锁。

事务A: select * from dept where deptno=1 lock in share mode; //共享锁,1

update dept set dname='java' where deptno=1;//排他锁,3

事务B: update dept set dname='Java' where deptno=1;//由于1有共享锁，没法获取排他锁，需 等待，2

解决方案：

对于按钮等控件，点击立刻失效，不让用户重复点击，避免引发同时对同一条记录多次操 作； 使用乐观锁进行控制。乐观锁机制避免了长事务中的数据库加锁开销，大大提升了大并发量 下的系统性能。需要注意的是，由于乐观锁机制是在我们的系统中实现，来自外部系统的用 户更新操作不受我们系统的控制，因此可能会造成脏数据被更新到数据库中； 四、死锁排查

MySQL提供了几个与锁有关的参数和命令，可以辅助我们优化锁操作，减少死锁发生。

查看死锁日志

通过show engine innodb status\G命令查看近期死锁日志信息。

使用方法：1、查看近期死锁日志信息；2、使用explain查看下SQL执行计划

查看锁状态变量

通过show status like'innodb_row_lock%‘命令检查状态变量，分析系统中的行锁的争夺

情况

Innodb_row_lock_current_waits：当前正在等待锁的数量 Innodb_row_lock_time：从系统启动到现在锁定总时间长度 Innodb_row_lock_time_avg： 每次等待锁的平均时间 Innodb_row_lock_time_max：从系统启动到现在等待最长的一次锁的时间 Innodb_row_lock_waits：系统启动后到现在总共等待的次数

如果等待次数高，而且每次等待时间长，需要分析系统中为什么会有如此多的等待，然后着 手定制优化。

第四部分 MySQL集群架构

第1节 集群架构设计

1.1 架构设计理念

在集群架构设计时，主要遵从下面三个维度：

可用性 扩展性 一致性

1.2 可用性设计

站点高可用，冗余站点 服务高可用，冗余服务 数据高可用，冗余数据 保证高可用的方法是冗余。但是数据冗余带来的问题是数据一致性问题。

实现高可用的方案有以下几种架构模式：

主从模式 简单灵活，能满足多种需求。比较主流的用法，但是写操作高可用需要自行处理。 双主模式 互为主从，有双主双写、双主单写两种方式，建议使用双主单写

1.3 扩展性设计

扩展性主要围绕着读操作扩展和写操作扩展展开。

如何扩展以提高读性能 加从库 简单易操作，方案成熟。 从库过多会引发主库性能损耗。建议不要作为长期的扩充方案，应该设法用良好的设计避免 持续加从库来缓解读性能问题。 分库分表 可以分为垂直拆分和水平拆分，垂直拆分可以缓解部分压力，水平拆分理论上可以无限扩 展。 如何扩展以提高写性能 分库分表

1.4 一致性设计

一致性主要考虑集群中各数据库数据同步以及同步延迟问题。可以采用的方案如下： 不使用从库 扩展读性能问题需要单独考虑，否则容易出现系统瓶颈。 增加访问路由层 可以先得到主从同步最长时间t，在数据发生修改后的t时间内，先访问主库。

第2节 主从模式

2.1 适用场景

MySQL主从模式是指数据可以从一个MySQL数据库服务器主节点复制到一个或多个从节点。MySQL 默 认采用异步复制方式，这样从节点不用一直访问主服务器来更新自己的数据，从节点可以复制主数据库 中的所有数据库，或者特定的数据库，或者特定的表。 mysql主从复制用途：

实时灾备，用于故障切换（高可用） 读写分离，提供查询服务（读扩展） 数据备份，避免影响业务（高可用）

主从部署必要条件：

从库服务器能连通主库 主库开启binlog日志（设置log-bin参数） 主从server-id不同

2.2 实现原理

2.2.1 主从复制

下图是主从复制的原理图。

主从复制整体分为以下三个步骤：

主库将数据库的变更操作记录到Binlog日志文件中 从库读取主库中的Binlog日志文件信息写入到从库的Relay Log中继日志中 从库读取中继日志信息在从库中进行Replay,更新从库数据信息 在上述三个过程中，涉及了Master的BinlogDump Thread和Slave的I/O Thread、SQL Thread，它们 的作用如下： Master服务器对数据库更改操作记录在Binlog中，BinlogDump Thread接到写入请求后，读取 Binlog信息推送给Slave的I/O Thread。 Slave的I/O Thread将读取到的Binlog信息写入到本地Relay Log中。 Slave的SQL Thread检测到Relay Log的变更请求，解析relay log中内容在从库上执行。 上述过程都是异步操作，俗称异步复制，存在数据延迟现象。 下图是异步复制的时序图。

mysql主从复制存在的问题：

主库宕机后，数据可能丢失 从库只有一个SQL Thread，主库写压力大，复制很可能延时 解决方法： 半同步复制---解决数据丢失的问题 并行复制----解决从库复制延迟的问题 2.2.2 半同步复制 为了提升数据安全，MySQL让Master在某一个时间点等待Slave节点的 ACK（Acknowledge character）消息，接收到ACK消息后才进行事务提交，这也是半同步复制的基础，MySQL从5.5版本开 始引入了半同步复制机制来降低数据丢失的概率。 介绍半同步复制之前先快速过一下 MySQL 事务写入碰到主从复制时的完整过程，主库事务写入分为 4 个步骤： InnoDB Redo File Write (Prepare Write) Binlog File Flush & Sync to Binlog File InnoDB Redo File Commit（Commit Write） Send Binlog to Slave

当Master不需要关注Slave是否接受到Binlog Event时，即为传统的主从复制。 当Master需要在第三步等待Slave返回ACK时，即为 after-commit，半同步复制（MySQL 5.5引入）。 当Master需要在第二步等待 Slave 返回 ACK 时，即为 after-sync，增强半同步（MySQL 5.7引入）。 下图是 MySQL 官方对于半同步复制的时序图，主库等待从库写入 relay log 并返回 ACK 后才进行 Engine Commit。 2.3 并行复制

MySQL的主从复制延迟一直是受开发者最为关注的问题之一，MySQL从5.6版本开始追加了并行复制功 能，目的就是为了改善复制延迟问题，并行复制称为enhanced multi-threaded slave（简称MTS）。

在从库中有两个线程IO Thread和SQL Thread，都是单线程模式工作，因此有了延迟问题，我们可以采 用多线程机制来加强，减少从库复制延迟。（IO Thread多线程意义不大，主要指的是SQL Thread多线 程）

在MySQL的5.6、5.7、8.0版本上，都是基于上述SQL Thread多线程思想，不断优化，减少复制延迟。

2.3.1 MySQL 5.6并行复制原理

MySQL 5.6版本也支持所谓的并行复制，但是其并行只是基于库的。如果用户的MySQL数据库中是多个 库，对于从库复制的速度的确可以有比较大的帮助。 基于库的并行复制，实现相对简单，使用也相对简单些。基于库的并行复制遇到单库多表使用场景就发 挥不出优势了，另外对事务并行处理的执行顺序也是个大问题。

2.3.2 MySQL 5.7并行复制原理

MySQL 5.7是基于组提交的并行复制，MySQL 5.7才可称为真正的并行复制，这其中最为主要的原因就 是slave服务器的回放与master服务器是一致的，即master服务器上是怎么并行执行的slave上就怎样进 行并行回放。不再有库的并行复制限制。

MySQL 5.7中组提交的并行复制究竟是如何实现的？

MySQL 5.7是通过对事务进行分组，当事务提交时，它们将在单个操作中写入到二进制日志中。如果多 个事务能同时提交成功，那么它们意味着没有冲突，因此可以在Slave上并行执行，所以通过在主库上 的二进制日志中添加组提交信息。

MySQL 5.7的并行复制基于一个前提，即所有已经处于prepare阶段的事务，都是可以并行提交的。这 些当然也可以在从库中并行提交，因为处理这个阶段的事务都是没有冲突的。在一个组里提交的事务， 一定不会修改同一行。这是一种新的并行复制思路，完全摆脱了原来一直致力于为了防止冲突而做的分 发算法，等待策略等复杂的而又效率底下的工作。

InnoDB事务提交采用的是两阶段提交模式。一个阶段是prepare，另一个是commit。

为了兼容MySQL 5.6基于库的并行复制，5.7引入了新的变量slave-parallel-type，其可以配置的值有： DATABASE（默认值，基于库的并行复制方式）、LOGICAL_CLOCK（基于组提交的并行复制方式）。

那么如何知道事务是否在同一组中，生成的Binlog内容如何告诉Slave哪些事务是可以并行复制的？

在MySQL 5.7版本中，其设计方式是将组提交的信息存放在GTID中。为了避免用户没有开启GTID功能 （gtid_mode=OFF），MySQL 5.7又引入了称之为Anonymous_Gtid的二进制日志event类型 ANONYMOUS_GTID_LOG_EVENT。

通过mysqlbinlog工具分析binlog日志，就可以发现组提交的内部信息。

可以发现MySQL 5.7二进制日志较之原来的二进制日志内容多了last_committed和

sequence_number，last_committed表示事务提交的时候，上次事务提交的编号，如果事务具有相同 的last_committed，表示这些事务都在一组内，可以进行并行的回放。

2.3.3 MySQL8.0 并行复制

MySQL8.0 是基于write-set的并行复制。MySQL会有一个集合变量来存储事务修改的记录信息（主键哈

希值），所有已经提交的事务所修改的主键值经过hash后都会与那个变量的集合进行对比，来判断改行 是否与其冲突，并以此来确定依赖关系，没有冲突即可并行。这样的粒度，就到了 row级别了，此时并 行的粒度更加精细，并行的速度会更快。

2.3.4 并行复制配置与调优

binlog_transaction_dependency_history_size 用于控制集合变量的大小。

binlog_transaction_depandency_tracking

用于控制binlog文件中事务之间的依赖关系，即last_committed值。

COMMIT_ORDERE: 基于组提交机制 WRITESET: 基于写集合机制 WRITESET_SESSION: 基于写集合，比writeset多了一个约束，同一个session中的事务 last_committed按先后顺序递增 transaction_write_set_extraction

用于控制事务的检测算法，参数值为：OFF、 XXHASH64、MURMUR32

master_info_repository

开启MTS功能后，务必将参数master_info_repostitory设置为TABLE，这样性能可以有50%~80% 的提升。这是因为并行复制开启后对于元master.info这个文件的更新将会大幅提升，资源的竞争 也会变大。

slave_parallel_workers

若将slave_parallel_workers设置为0，则MySQL 5.7退化为原单线程复制，但将 slave_parallel_workers设置为1，则SQL线程功能转化为coordinator线程，但是只有1个worker 线程进行回放，也是单线程复制。然而，这两种性能却又有一些的区别，因为多了一次 coordinator线程的转发，因此slave_parallel_workers=1的性能反而比0还要差。

slave_preserve_commit_order

MySQL 5.7后的MTS可以实现更小粒度的并行复制，但需要将slave_parallel_type设置为 LOGICAL_CLOCK，但仅仅设置为LOGICAL_CLOCK也会存在问题，因为此时在slave上应用事务的 顺序是无序的，和relay log中记录的事务顺序不一样，这样数据一致性是无法保证的，为了保证事 务是按照relay log中记录的顺序来回放，就需要开启参数slave_preserve_commit_order。

要开启enhanced multi-threaded slave其实很简单，只需根据如下设置：

slave-parallel-type=LOGICAL_CLOCK slave-parallel-workers=16 slave_pending_jobs_size_max = 2147483648 slave_preserve_commit_order=1 master_info_repository=TABLE relay_log_info_repository=TABLE relay_log_recovery=ON

2.3.5 并行复制监控

在使用了MTS后，复制的监控依旧可以通过SHOW SLAVE STATUS\G，但是MySQL 5.7在 performance_schema库中提供了很多元数据表，可以更详细的监控并行复制过程。 mysql> show tables like 'replication%'; +---------------------------------------------+ | Tables_in_performance_schema (replication%) | +---------------------------------------------+ | replication_applier_configuration | | replication_applier_status | | replication_applier_status_by_coordinator | | replication_applier_status_by_worker | | replication_connection_configuration | | replication_connection_status | | replication_group_member_stats | | replication_group_members | +---------------------------------------------+

通过replication_applier_status_by_worker可以看到worker进程的工作情况：

mysql> select * from replication_applier_status_by_worker; +--------------+-----------+-----------+---------------+------------------------

--------------------+-------------------+--------------------+------------------

----+ | CHANNEL_NAME | WORKER_ID | THREAD_ID | SERVICE_STATE | LAST_SEEN_TRANSACTION | LAST_ERROR_NUMBER | LAST_ERROR_MESSAGE | LAST_ERROR_TIMESTAMP | +--------------+-----------+-----------+---------------+------------------------

--------------------+-------------------+--------------------+------------------

----+ | | 1 | 32 | ON | 0d8513d8-00a4-11e6a510-f4ce46861268:96604 | 0 | | 0000-00-00 00:00:00 | | | 2 | 33 | ON | 0d8513d8-00a4-11e6a510-f4ce46861268:97760 | 0 | | 0000-00-00 00:00:00 | +--------------+-----------+-----------+---------------+------------------------

--------------------+-------------------+--------------------+------------------

----+

2 rows in set (0.00 sec)

最后，如果MySQL 5.7要使用MTS功能，建议使用新版本，最少升级到5.7.19版本，修复了很多Bug。

2.4 读写分离

2.4.1 读写分离引入时机

大多数互联网业务中，往往读多写少，这时候数据库的读会首先成为数据库的瓶颈。如果我们已经优化 了SQL，但是读依旧还是瓶颈时，这时就可以选择“读写分离”架构了。 读写分离首先需要将数据库分为主从库，一个主库用于写数据，多个从库完成读数据的操作，主从库之 间通过主从复制机制进行数据的同步，如图所示。 在应用中可以在从库追加多个索引来优化查询，主库这些索引可以不加，用于提升写效率。 读写分离架构也能够消除读写锁冲突从而提升数据库的读写性能。使用读写分离架构需要注意：主从同 步延迟和读写分配机制问题 2.4.2 主从同步延迟

使用读写分离架构时，数据库主从同步具有延迟性，数据一致性会有影响，对于一些实时性要求比较高 的操作，可以采用以下解决方案。

写后立刻读 在写入数据库后，某个时间段内读操作就去主库，之后读操作访问从库。 二次查询 先去从库读取数据，找不到时就去主库进行数据读取。该操作容易将读压力返还给主库，为了避免 恶意攻击，建议对数据库访问API操作进行封装，有利于安全和低耦合。 根据业务特殊处理 根据业务特点和重要程度进行调整，比如重要的，实时性要求高的业务数据读写可以放在主库。对 于次要的业务，实时性要求不高可以进行读写分离，查询时去从库查询。

2.4.3 读写分离落地

读写路由分配机制是实现读写分离架构最关键的一个环节，就是控制何时去主库写，何时去从库读。目 前较为常见的实现方案分为以下两种： 基于编程和配置实现（应用端） 程序员在代码中封装数据库的操作，代码中可以根据操作类型进行路由分配，增删改时操作主库， 查询时操作从库。这类方法也是目前生产环境下应用最广泛的。优点是实现简单，因为程序在代码 中实现，不需要增加额外的硬件开支，缺点是需要开发人员来实现，运维人员无从下手，如果其中 一个数据库宕机了，就需要修改配置重启项目。

基于服务器端代理实现（服务器端）

中间件代理一般介于应用服务器和数据库服务器之间，从图中可以看到，应用服务器并不直接进入 到master数据库或者slave数据库，而是进入MySQL proxy代理服务器。代理服务器接收到应用服 务器的请求后，先进行判断然后转发到后端master和slave数据库。

目前有很多性能不错的数据库中间件，常用的有MySQL Proxy、MyCat以及Shardingsphere等等。

MySQL Proxy：是官方提供的MySQL中间件产品可以实现负载平衡、读写分离等。

MyCat：MyCat是一款基于阿里开源产品Cobar而研发的，基于 Java 语言编写的开源数据库中间 件。 ShardingSphere：ShardingSphere是一套开源的分布式数据库中间件解决方案，它由ShardingJDBC、Sharding-Proxy和Sharding-Sidecar（计划中）这3款相互独立的产品组成。已经在2020 年4月16日从Apache孵化器毕业，成为Apache顶级项目。 Atlas：Atlas是由 Qihoo 360公司Web平台部基础架构团队开发维护的一个数据库中间件。 Amoeba：变形虫，该开源框架于2008年开始发布一款 Amoeba for MySQL软件。

第3节 双主模式

3.1 适用场景

很多企业刚开始都是使用MySQL主从模式，一主多从、读写分离等。但是单主如果发生单点故障，从库 切换成主库还需要作改动。因此，如果是双主或者多主，就会增加MySQL入口，提升了主库的可用性。 因此随着业务的发展，数据库架构可以由主从模式演变为双主模式。双主模式是指两台服务器互为主 从，任何一台服务器数据变更，都会通过复制应用到另外一方的数据库中。 使用双主双写还是双主单写？

建议大家使用双主单写，因为双主双写存在以下问题：

ID冲突

在A主库写入，当A数据未同步到B主库时，对B主库写入，如果采用自动递增容易发生ID主键的冲 突。

可以采用MySQL自身的自动增长步长来解决，例如A的主键为1,3,5,7...，B的主键为2,4,6,8... ，但 是对数据库运维、扩展都不友好。

更新丢失

同一条记录在两个主库中进行更新，会发生前面覆盖后面的更新丢失。

高可用架构如下图所示，其中一个Master提供线上服务，另一个Master作为备胎供高可用切换， Master下游挂载Slave承担读请求。 随着业务发展，架构会从主从模式演变为双主模式，建议用双主单写，再引入高可用组件，例如 Keepalived和MMM等工具，实现主库故障自动切换。

3.2 MMM架构

MMM（Master-Master Replication Manager for MySQL）是一套用来管理和监控双主复制，支持双 主故障切换 的第三方软件。MMM 使用Perl语言开发，虽然是双主架构，但是业务上同一时间只允许一 个节点进行写入操作。下图是基于MMM实现的双主高可用架构。 MMM故障处理机制 MMM 包含writer和reader两类角色，分别对应写节点和读节点。 当 writer节点出现故障，程序会自动移除该节点上的VIP 写操作切换到 Master2，并将Master2设置为writer 将所有Slave节点会指向Master2

除了管理双主节点，MMM 也会管理 Slave 节点，在出现宕机、复制延迟或复制错误，MMM 会移 除该节点的 VIP，直到节点恢复正常。

MMM监控机制 MMM 包含monitor和agent两类程序，功能如下：

monitor：监控集群内数据库的状态，在出现异常时发布切换命令，一般和数据库分开部 署。 agent：运行在每个 MySQL 服务器上的代理进程，monitor 命令的执行者，完成监控的探针 工作和具体服务设置，例如设置 VIP（虚拟IP）、指向新同步节点。

3.3 MHA架构

MHA（Master High Availability）是一套比较成熟的 MySQL 高可用方案，也是一款优秀的故障切换和 主从提升的高可用软件。在MySQL故障切换过程中，MHA能做到在30秒之内自动完成数据库的故障切 换操作，并且在进行故障切换的过程中，MHA能在最大程度上保证数据的一致性，以达到真正意义上的 高可用。MHA还支持在线快速将Master切换到其他主机，通常只需0.5－2秒。 目前MHA主要支持一主多从的架构，要搭建MHA，要求一个复制集群中必须最少有三台数据库服务 器。

MHA由两部分组成：MHA Manager（管理节点）和MHA Node（数据节点）。

MHA Manager可以单独部署在一台独立的机器上管理多个master-slave集群，也可以部署在一台 slave节点上。负责检测master是否宕机、控制故障转移、检查MySQL复制状况等。 MHA Node运行在每台MySQL服务器上，不管是Master角色，还是Slave角色，都称为Node，是 被监控管理的对象节点，负责保存和复制master的二进制日志、识别差异的中继日志事件并将其 差异的事件应用于其他的slave、清除中继日志。

MHA Manager会定时探测集群中的master节点，当master出现故障时，它可以自动将最新数据的 slave提升为新的master，然后将所有其他的slave重新指向新的master，整个故障转移过程对应用程序 完全透明。

MHA故障处理机制：

把宕机master的binlog保存下来 根据binlog位置点找到最新的slave 用最新slave的relay log修复其它slave 将保存下来的binlog在最新的slave上恢复 将最新的slave提升为master 将其它slave重新指向新提升的master，并开启主从复制

MHA优点：

自动故障转移快 主库崩溃不存在数据一致性问题 性能优秀，支持半同步复制和异步复制 一个Manager监控节点可以监控多个集群

3.4 主备切换 主备切换是指将备库变为主库，主库变为备库，有可靠性优先和可用性优先两种策略。

主备延迟问题 主备延迟是由主从数据同步延迟导致的，与数据同步有关的时间点主要包括以下三个：

主库 A 执行完成一个事务，写入 binlog，我们把这个时刻记为 T1;

之后将binlog传给备库 B，我们把备库 B 接收完 binlog 的时刻记为 T2;

备库 B 执行完成这个binlog复制，我们把这个时刻记为 T3。 所谓主备延迟，就是同一个事务，在备库执行完成的时间和主库执行完成的时间之间的差值，也就 是 T3-T1。 在备库上执行show slave status命令，它可以返回结果信息，seconds_behind_master表示当前 备库延迟了多少秒。 同步延迟主要原因如下：

备库机器性能问题

机器性能差，甚至一台机器充当多个主库的备库。

分工问题

备库提供了读操作，或者执行一些后台分析处理的操作，消耗大量的CPU资源。

大事务操作

大事务耗费的时间比较长，导致主备复制时间长。比如一些大量数据的delete或大表DDL操

作都可能会引发大事务。 可靠性优先 主备切换过程一般由专门的HA高可用组件完成，但是切换过程中会存在短时间不可用，因为在切 换过程中某一时刻主库A和从库B都处于只读状态。如下图所示：

主库由A切换到B，切换的具体流程如下： 判断从库B的Seconds_Behind_Master值，当小于某个值才继续下一步 把主库A改为只读状态（readonly=true） 等待从库B的Seconds_Behind_Master值降为 0 把从库B改为可读写状态（readonly=false） 把业务请求切换至从库B 可用性优先 不等主从同步完成， 直接把业务请求切换至从库B ，并且让 从库B可读写 ，这样几乎不存在不可 用时间，但可能会数据不一致。 如上图所示，在A切换到B过程中，执行两个INSERT操作，过程如下：

主库A执行完 INSERT c=4 ，得到 (4,4) ，然后开始执行 主从切换 主从之间有5S的同步延迟，从库B会先执行 INSERT c=5 ，得到 (4,5) 从库B执行主库A传过来的binlog日志 INSERT c=4 ，得到 (5,4) 主库A执行从库B传过来的binlog日志 INSERT c=5 ，得到 (5,5) 此时主库A和从库B会有 两行 不一致的数据

通过上面介绍了解到，主备切换采用可用性优先策略，由于可能会导致数据不一致，所以大多数情 况下，优先选择可靠性优先策略。在满足数据可靠性的前提下，MySQL的可用性依赖于同步延时 的大小，同步延时越小，可用性就越高。

第4节 分库分表

互联网系统需要处理大量用户的请求。比如微信日活用户破10亿，海量的用户每天产生海量的数量；美 团外卖，每天都是几千万的订单，那这些系统的用户表、订单表、交易流水表等是如何处理呢？

数据量只增不减，历史数据又必须要留存，非常容易成为性能的瓶颈，而要解决这样的数据库瓶颈问 题，“读写分离”和缓存往往都不合适，目前比较普遍的方案就是使用NoSQL/NewSQL或者采用分库分 表。

使用分库分表时，主要有垂直拆分和水平拆分两种拆分模式，都属于物理空间的拆分。

分库分表方案：只分库、只分表、分库又分表。

垂直拆分：由于表数量多导致的单个库大。将表拆分到多个库中。

水平拆分：由于表记录多导致的单个库大。将表记录拆分到多个表中。

4.1 拆分方式

垂直拆分

垂直拆分又称为纵向拆分，垂直拆分是将表按库进行分离，或者修改表结构按照访问的差异将某些 列拆分出去。应用时有垂直分库和垂直分表两种方式，一般谈到的垂直拆分主要指的是垂直分库。 如下图所示，采用垂直分库，将用户表和订单表拆分到不同的数据库中。

垂直分表就是将一张表中不常用的字段拆分到另一张表中，从而保证第一张表中的字段较少，避免 出现数据库跨页存储的问题，从而提升查询效率。

解决：一个表中字段过多，还有有些字段经常使用，有些字段不经常使用，或者还有text等字段信 息。可以考虑使用垂直分表方案。 按列进行垂直拆分，即把一条记录分开多个地方保存，每个子表的行数相同。把主键和一些列放到 一个表，然后把主键和另外的列放到另一个表中。

垂直拆分优点：

拆分后业务清晰，拆分规则明确； 易于数据的维护和扩展； 可以使得行数据变小，一个数据块 (Block) 就能存放更多的数据，在查询时就会减少 I/O 次 数； 可以达到最大化利用 Cache 的目的，具体在垂直拆分的时候可以将不常变的字段放一起，将 经常改变的放一起； 便于实现冷热分离的数据表设计模式。

垂直拆分缺点：

主键出现冗余，需要管理冗余列； 会引起表连接 JOIN 操作，可以通过在业务服务器上进行 join 来减少数据库压力，提高了系 统的复杂度； 依然存在单表数据量过大的问题； 事务处理复杂。 水平拆分

水平拆分又称为横向拆分。 相对于垂直拆分，它不再将数据根据业务逻辑分类，而是通过某个字 段（或某几个字段），根据某种规则将数据分散至多个库或表中，每个表仅包含数据的一部分，如 下图所示。

水平分表是将一张含有很多记录数的表水平切分，不同的记录可以分开保存，拆分成几张结构相同 的表。如果一张表中的记录数过多，那么会对数据库的读写性能产生较大的影响，虽然此时仍然能 够正确地读写，但读写的速度已经到了业务无法忍受的地步，此时就需要使用水平分表来解决这个 问题。

水平拆分：解决表中记录过多问题。

垂直拆分：解决表过多或者是表字段过多问题。 水平拆分重点考虑拆分规则：例如范围、时间或Hash算法等。

水平拆分优点：

拆分规则设计好，join 操作基本可以数据库做； 不存在单库大数据，高并发的性能瓶颈； 切分的表的结构相同，应用层改造较少，只需要增加路由规则即可； 提高了系统的稳定性和负载能力。

水平拆分缺点：

拆分规则难以抽象； 跨库Join性能较差； 分片事务的一致性难以解决； 数据扩容的难度和维护量极大。

日常工作中，我们通常会同时使用两种拆分方式，垂直拆分更偏向于产品/业务/功能拆分的过程，在技 术上我们更关注水平拆分的方案。

4.2 主键策略

在很多中小项目中，我们往往直接使用数据库自增特性来生成主键ID，这样确实比较简单。而在分库分 表的环境中，数据分布在不同的数据表中，不能再借助数据库自增长特性直接生成，否则会造成不同数 据表主键重复。下面介绍几种ID生成算法。

UUID

UUID是通用唯一识别码（Universally Unique Identiﬁer）的缩写，长度是16个字节，被表示为 32个十六进制数字，以“ - ”分隔的五组来显示，格式为8-4-4-4-12，共36个字符，例如： 550e8400-e29b-41d4-a716-446655440000。UUID在生成时使用到了以太网卡地址、纳秒级时 间、芯片ID码和随机数等信息，目的是让分布式系统中的所有元素都能有唯一的识别信息。

使用UUID做主键，可以在本地生成，没有网络消耗，所以生成性能高。但是UUID比较长，没有规 律性，耗费存储空间。

All indexes other than the clustered index are known as secondary indexes. In InnoDB, each record in a secondary index contains the primary key columns for the row, as well as the columns speciﬁed for the secondary index. InnoDB uses this primary key value to search for the row in the clustered index. If the primary key is long, the secondary indexes use more space, so it is advantageous to have a short primary key.

除聚集索引以外的所有索引都称为辅助索引。在InnoDB中，二级索引中的每条记录都包含行的主 键列，以及为二级索引指定的列。InnoDB使用这个主键值来搜索聚集索引中的行。如果主键是长 的，则次索引使用更多的空间，因此主键短是有利的。

如果UUID作为数据库主键，在InnoDB引擎下，UUID的无序性可能会引起数据位置频繁变动，影响性 能。

COMB（UUID变种）

COMB（combine）型是数据库特有的一种设计思想，可以理解为一种改进的GUID，它通过组合 GUID和系统时间，以使其在索引和检索事有更优的性能。数据库中没有COMB类型，它是Jimmy Nilsson在他的“The Cost of GUIDs as Primary Keys”一文中设计出来的。 COMB设计思路是这样的：既然UniqueIdentiﬁer数据因毫无规律可言造成索引效率低下，影响了 系统的性能，那么我们能不能通过组合的方式，保留UniqueIdentiﬁer的前10个字节，用后6个字 节表示GUID生成的时间（DateTime），这样我们将时间信息与UniqueIdentiﬁer组合起来，在保 留UniqueIdentiﬁer的唯一性的同时增加了有序性，以此来提高索引效率。解决UUID无序的问 题，性能优于UUID。 SNOWFLAKE

有些时候我们希望能使用一种简单一些的ID，并且希望ID能够按照时间有序生成，SnowFlake解决 了这种需求。SnowFlake是Twitter开源的分布式ID生成算法，结果是一个long型的ID，long型是8 个字节，64-bit。其核心思想是：使用41bit作为毫秒数，10bit作为机器的ID（5个bit是数据中 心，5个bit的机器ID），12bit作为毫秒内的流水号，最后还有一个符号位，永远是0。如下图所 示：

SnowFlake生成的ID整体上按照时间自增排序，并且整个分布式系统内不会产生ID重复，并且效率 较高。经测试SnowFlake每秒能够产生26万个ID。缺点是强依赖机器时钟，如果多台机器环境时 钟没同步，或时钟回拨，会导致发号重复或者服务会处于不可用状态。因此一些互联网公司也基于 上述的方案做了封装，例如百度的uidgenerator（基于SnowFlake）和美团的leaf（基于数据库和 SnowFlake）等。

数据库ID表

比如A表分表为A1表和A2表，我们可以单独的创建一个MySQL数据库，在这个数据库中创建一张 表，这张表的ID设置为自动递增，其他地方需要全局唯一ID的时候，就先向这个这张表中模拟插 入一条记录，此时ID就会自动递增，然后我们获取刚生成的ID后再进行A1和A2表的插入。 例如，下面DISTRIBUTE_ID就是我们创建要负责ID生成的表，结构如下：

CREATE TABLE DISTRIBUTE_ID ( id bigint(32) NOT NULL AUTO_INCREMENT COMMENT '主键', createtime datetime DEFAULT NULL, PRIMARY KEY (id) ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

当分布式集群环境中哪个应用需要获取一个全局唯一的分布式ID的时候，就可以使用代码连接这 个数据库实例，执行如下SQL语句即可。

insert into DISTRIBUTE_ID(createtime) values(NOW()); select LAST_INSERT_ID()；

注意：

这里的createtime字段无实际意义，是为了随便插入一条数据以至于能够自动递增ID。 使用独立的MySQL实例生成分布式ID，虽然可行，但是性能和可靠性都不够好，因为你需要 代 码连接到数据库才能获取到ID，性能无法保障，另外mysql数据库实例挂掉了，那么就无法 获取分 布式ID了。 Redis生成ID

当使用数据库来生成ID性能不够要求的时候，我们可以尝试使用Redis来生成ID。这主要依赖于 Redis是单线程的，所以也可以用生成全局唯一的ID。可以用Redis的原子操作 INCR和INCRBY来 实现。

也可以使用Redis集群来获取更高的吞吐量。假如一个集群中有5台Redis。可以初始化每台Redis 的值分别是1,2,3,4,5，然后步长都是5。各个Redis生成的ID为：

A：1,6,11,16,21 B：2,7,12,17,22 C：3,8,13,18,23 D：4,9,14,19,24 E：5,10,15,20,25

4.3 分片策略

4.3.1 分片概念

分片（Sharding）就是用来确定数据在多台存储设备上分布的技术。Shard这个词的意思是“碎片”，如 果将一个数据库当作一块大玻璃，将这块玻璃打碎，那么每一小块都称为数据库的碎片（Database Sharding）。将一个数据库打碎成多个的过程就叫做分片，分片是属于横向扩展方案。

分片：表示分配过程，是一个逻辑上概念，表示如何实现

分库分表：表示分配结果，是一个物理上概念，表示最终实现的结果

数据库扩展方案：

横向扩展：一个库变多个库，加机器数量 纵向扩展：一个库还是一个库，优化机器性能，加高配CPU或内存

在分布式存储系统中，数据需要分散存储在多台设备上，分片就是把数据库横向扩展到多个数据库服务 器上的一种有效的方式，其主要目的就是为突破单节点数据库服务器的 I/O 能力限制，解决数据库扩展 性问题。

4.3.2 分片策略

数据分片是根据指定的分片键和分片策略将数据水平拆分，拆分成多个数据片后分散到多个数据存储节 点中。分片键是用于划分和定位表的字段，一般使用ID或者时间字段。而分片策略是指分片的规则，常 用规则有以下几种。

基于范围分片

根据特定字段的范围进行拆分，比如用户ID、订单时间、产品价格等。例如：

{[1 - 100] => Cluster A, [101 - 199] => Cluster B}

优点：新的数据可以落在新的存储节点上，如果集群扩容，数据无需迁移。

缺点：数据热点分布不均，数据冷热不均匀，导致节点负荷不均。

哈希取模分片

整型的Key可直接对设备数量取模，其他类型的字段可以先计算Key的哈希值，然后再对设备数量 取模。假设有n台设备，编号为0 ~ n-1，通过Hash(Key) % n就可以确定数据所在的设备编号。该 模式也称为离散分片。 优点：实现简单，数据分配比较均匀，不容易出现冷热不均，负荷不均的情况。

缺点：扩容时会产生大量的数据迁移，比如从n台设备扩容到n+1，绝大部分数据需要重新分配和 迁移。

一致性哈希分片

采用Hash取模的方式进行拆分，后期集群扩容需要迁移旧的数据。使用一致性Hash算法能够很大 程度的避免这个问题，所以很多中间件的集群分片都会采用一致性Hash算法。

一致性Hash是将数据按照特征值映射到一个首尾相接的Hash环上，同时也将节点（按照IP地址或 者机器名Hash）映射到这个环上。对于数据，从数据在环上的位置开始，顺时针找到的第一个节 点即为数据的存储节点。Hash环示意图与数据的分布如下：

一致性Hash在增加或者删除节点的时候，受到影响的数据是比较有限的，只会影响到Hash环相邻的节 点，不会发生大规模的数据迁移。

4.4 扩容方案

当系统用户进入了高速增长期时，即便是对数据进行分库分表，但数据库的容量，还有表的数据量也总 会达到天花板。当现有数据库达到承受极限时，就需要增加新服务器节点数量进行横向扩容。

首先来思考一下，横向扩展会有什么技术难度？

数据迁移问题 分片规则改变 数据同步、时间点、数据一致性

遇到上述问题时，我们可以使用以下两种方案：

4.4.1 停机扩容

这是一种很多人初期都会使用的方案，尤其是初期只有几台数据库的时候。停机扩容的具体步骤如下：

站点发布一个公告，例如：“为了为广大用户提供更好的服务，本站点将在今晚00:00-2:00之间升 级，给您带来不便抱歉"； 时间到了，停止所有对外服务； 新增n个数据库，然后写一个数据迁移程序，将原有x个库的数据导入到最新的y个库中。比如分片 规则由%x变为%y； 数据迁移完成，修改数据库服务配置，原来x个库的配置升级为y个库的配置 重启服务，连接新库重新对外提供服务

回滚方案：万一数据迁移失败，需要将配置和数据回滚，改天再挂公告。

优点：简单

缺点：

停止服务，缺乏高可用 程序员压力山大，需要在指定时间完成 如果有问题没有及时测试出来启动了服务，运行后发现问题，数据会丢失一部分，难以回滚。

适用场景：

小型网站 大部分游戏 对高可用要求不高的服务

4.4.2 平滑扩容

数据库扩容的过程中，如果想要持续对外提供服务，保证服务的可用性，平滑扩容方案是最好的选择。 平滑扩容就是将数据库数量扩容成原来的2倍，比如：由2个数据库扩容到4个数据库，具体步骤如下：

新增2个数据库

配置双主进行数据同步（先测试、后上线）

数据同步完成之后，配置双主双写（同步因为有延迟，如果时时刻刻都有写和更新操作，会存在不 准确问题） 数据同步完成后，删除双主同步，修改数据库配置，并重启；

此时已经扩容完成，但此时的数据并没有减少，新增的数据库跟旧的数据库一样多的数据，此时还 需要写一个程序，清空数据库中多余的数据，如：

User1去除 uid % 4 = 2的数据； User3去除 uid % 4 = 0的数据； User2去除 uid % 4 = 3的数据； User4去除 uid % 4 = 1的数据；

平滑扩容方案能够实现n库扩2n库的平滑扩容，增加数据库服务能力，降低单库一半的数据量。其核心 原理是：成倍扩容，避免数据迁移。

优点：

扩容期间，服务正常进行，保证高可用 相对停机扩容，时间长，项目组压力没那么大，出错率低 扩容期间遇到问题，随时解决，不怕影响线上服务 可以将每个数据库数据量减少一半

缺点：

程序复杂、配置双主同步、双主双写、检测数据同步等 后期数据库扩容，比如成千上万，代价比较高

适用场景：

大型网站 对高可用要求高的服务