# MYSQL

## MYSQL架构

### MYSQL存储引擎

![](http://q8sats5bw.bkt.clouddn.com/image-20200325105353733.png)

查看存储引擎

``` sql
mysql> show engines;
```

InnoDB 和 MYISAM存储引擎的区别

![image-20200325112156259](http://q8sats5bw.bkt.clouddn.com/image-20200325112156259.png)

### MYSQL处理流程

简易处理流程

![](http://q8sats5bw.bkt.clouddn.com/image-20200325105659301.png)

详细执行流程

![image-20200325112303689](http://q8sats5bw.bkt.clouddn.com/image-20200325112303689.png)

### MYSQL日志文件

常用日志文件包括 ： 错误日志，二进制日志，查询日志，慢查询日志，事物redo日志，事物undo日志，中继日志

错误日志：5.5.7后默认开启，无法关闭

二进制日志：默认关闭，需要配置开启

```sql
### mysql-bin是二进制日志文件的basename
log-bin=mysql-bin
```

binlog记录了数据库所有的ddl语句和dml语句，但不包括select语句内容，语句以事件的形式保存，描
述了数据的变更顺序，binlog还包括了每个更新语句的执行时间信息。如果是DDL语句，则直接记录到
binlog日志，而DML语句，必须通过事务提交才能记录到binlog日志中。
binlog主要用于实现mysql主从复制、数据备份、数据恢复。  

通用查询日志： 默认关闭，建议不开启，开启方式如下：

```sql
# 启动开关
general_log={ON|OFF}
# 日志文件变量
general_log_file=/PATH/TO/file
# 记录类型
log_output={TABLE|FILE/NONE}
```

慢查询日志：默认关闭，开启方式如下：

```sql
#开启慢查询日志
slow_query_log=ON
#慢查询的阈值
long_query_time=10
#日志记录文件如果没有给出file_name值， 默认为主机名，后缀为-slow.log。如果给出了文件名，
但不是绝对路径名，文件则写入数据目录。
slow_query_log_file= file_name
1 2 3 4 5 6
```

记录执行时间超过long_query_time秒的所有查询，便于收集查询时间比较长的SQL语句  

重做日志（redo log)

作用：

确保事务的持久性，防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重
做，从而达到事务的持久性这一特性  

什么时间释放：

对应事务的脏页数据落盘后，redo log 的使命就完成了，重做日志占用的空间就可以重用（b被覆盖）

- [x] [***redo log 落盘时机选择***]()

很重要一点，redo log是什么时候写盘的？前面说了是在事物开始之后逐步写盘的。之所以说重做日
志是在事务开始之后逐步写入重做日志文件，而不一定是事务提交才写入重做日志缓存，原因就是，重
做日志有一个缓存区Innodb_log_buffer，Innodb_log_buffer的默认大小为8M(这里设置的
16M),Innodb存储引擎先将重做日志写入innodb_log_buffer中。
然后会通过以下三种方式将innodb日志缓冲区的日志刷新到磁盘
\1. Master Thread 每秒一次执行刷新Innodb_log_buffer到重做日志文件。
\2. 每个事务提交时会将重做日志刷新到重做日志文件。
\3. 当重做日志缓存可用空间 少于一半时，重做日志缓存被刷新到重做日志文件。
由此可以看出，重做日志通过不止一种方式写入到磁盘，尤其是对于第一种方式，
Innodb_log_buffer到重做日志文件是Master Thread线程的定时任务。
因此重做日志的写盘，并不一定是随着事务的提交才写入重做日志文件的，而是随着事务的开始，逐步
开始的。
另外引用《MySQL技术内幕 Innodb 存储引擎》（page37）上的原话：
即使某个事务还没有提交，Innodb存储引擎仍然每秒会将重做日志缓存刷新到重做日志文件。这一点是
必须要知道的，因为这可以很好地解释再大的事务的提交（commit）的时间也是很短暂的。  

回滚日志

作用：

保存了事务发生之前的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读。

什么时间产生：

事务开始之前，将当前是的版本生成undo log，undo 也会产生 redo 来保证undo log的可靠性  

什么时间释放：

当事务提交之后，undo log并不能立马被删除，而是放入待清理的链表，由purge线程判断是否由其他
事务在使用undo段中表的上一个事务之前的版本信息，决定是否可以清理undo log的日志空间。  

物理文件：

mysql 5.6之前，undo 表空间位于 共享表空间的回滚段中，共享表空间的默认名称是ibdata，位于数据目录文件中

MySQL5.6之后，undo表空间可以配置成独立的文件，但是提前需要在配置文件中配置，完成数据库初
始化后生效且不可改变undo log文件的个数。如果初始化数据库之前没有进行相关配置，那么就无法
配置成独立的表空间了。
关于MySQL5.7之后的独立undo 表空间配置参数如下
innodb_undo_directory = /data/undospace/ --undo独立表空间的存放目录
innodb_undo_logs = 128 --回滚段为128KB
innodb_undo_tablespaces = 4 --指定有4个undo log文件
如果undo使用的共享表空间，这个共享表空间中又不仅仅是存储了undo的信息，共享表空间的默认为
与MySQL的数据目录下面，其属性由参数innodb_data_file_path配置  

中继日志（relay log）

是在主从复制环境中产生的日志

主要作用是为了从机可以从中继日志中获取到主机同步过来的SQL语句，然后执行到从机中

数据文件（随机IO）：

查看mysql数据文件：

```sql
show variables like '%datadir%'
```

InnoDB数据文件

.frm文件：主要存放与表相关的数据信息，主要抱愧表结构定义信息

.idb：使用独享表空间存储表数据和索引信息，一张表对应一个idb文件

.ibdata文件：使用共享表空间存储表数据和索引信息，所有表共同使用一个或者多个ibdata文件

MYISAM数据文件

.frm文件：主要存储与表相关的数据信息，主要包括表结构的定义信息

.myd文件：主要用来存储表数据信息

.myi文件：主要存储表数据文件中任何索引的数据树

## MYSQL 索引

###  索引介绍

帮助MYSQL高效获取数据的数据结构，索引存储在磁盘上的文件中

聚集索引、覆盖索引、组合索引、前缀索引、唯一索引等，默认都是B+树结构

索引的优势和劣势

优势：

提高数据检索的效率，降低数据库的IO成本

通过索引列来对数据进行排序，降低数据排序的成本，降低cpu的消耗

​	被索引的列会自动进行排序，包括【单列索引】和 【组合索引】,只是组合索引的排序要复杂一些

劣势：

索引会占据磁盘空间。

虽然会提高查询的效率，但是会降低数据更新的效率。每次的增加删除修改操作不仅要保存数据，还要保存和更新对应的索引文件



### 索引分类

单列索引

​	普通索引：索引列没有必要唯一

​	唯一索引：索引列的值必须为一

​	主键索引：自增主键

组合索引

​	表中的多个字段上创建的索引

​	组合索引的使用遵循最左前缀原则

全文索引 （MYISAM引擎上用到的索引）

索引使用

创建索引： create index index_name on table(column(length))

​	alter table table_name add index index_name (column(length))

单列索引之唯一索引

create unique index index_name on table (column(length))

单列索引之全文索引

create fulltext index index_name on table (column(length))

组合索引

alter table table_name add index index_titme_time(title,time)

删除索引

drop index index_name on table 

查看索引

show index from table_name



### 索引的存储结构

页、块、扇区关系

内存以页这个单位去进行IO读取，一般大小为4K，在MySQL中可以通过Innodb_page_size设置
大小，一般设置为16K。
操作系统以块这个逻辑单位去操作磁盘，常见为4K。
磁盘以扇区这个物理最小磁盘单位去存储数据，常见为512Byte。
页大小查看： getconf PAGE_SIZE，常见为4K； 磁盘块大小查看：stat /boot/|grep “IO Block”，
常见为4K； 扇区大小查看：fdisk -l，常见为512Byte；
指针默认长度为6bit，如果key为bigint的话，为8bit，那么一个索引的话为8+6 = 14bit；  

索引实在存储引擎中实现的，也就是说不同的存储引擎会使用不通的索引数据结构

B树 和 B+树

B+ 树 和 B树最大的区别在于叶子节点是否存储数据

B树是非叶子节点和叶子节点都会存储数据

B+树只有叶子节点存储数据，而且存储的数据都是在一行上，而且这些数据都是有指针指向的，是有序链表



聚集索引 （InnoDB）

在完整的记录中，存储在主键索引中，通过主键索引，就可以获取记录所在的列，也就是数据和索引是在一起的，这就是聚集索引



主键索引

没有主键怎么办？mysql 会自动去创建一个虚拟主键

辅助索引（次要索引）

![image-20200325193646176](http://q8sats5bw.bkt.clouddn.com/image-20200325193646176.png)

![image-20200325193656772](http://q8sats5bw.bkt.clouddn.com/image-20200325193656772.png)

非聚集索引（MYISAM）

B+树叶子节点只会存储数据行（数据文件）的指针，简单来说就是数据和索引不在一起，这就是非聚集索引

非聚集索引包含主键索引和辅助索引，都会存储指针的值

主键索引

![image-20200325193958410](http://q8sats5bw.bkt.clouddn.com/image-20200325193958410.png)

![image-20200413104936687](http://q8sats5bw.bkt.clouddn.com/image-20200413104936687.png)

辅助索引（次要索引）

![image-20200325194158220](http://q8sats5bw.bkt.clouddn.com/image-20200325194158220.png)

联合索引的存储结构 

基于InnoDB的存储引擎

![image-20200325195342492](http://q8sats5bw.bkt.clouddn.com/image-20200325195342492.png)

最左前缀匹配原则

联合索引是首先使用多列索引的第一列来构建索引树，用上面的idx_t1_bcd(b,c,d)的例子就是优先用b列构建，当b列值相等后再以c列排序，c列值相等后则以d列值排序

![image-20200325195915542](http://q8sats5bw.bkt.clouddn.com/image-20200325195915542.png)

由于联合索引是上述描述方式构建索引树，所以联合索引只能从多列的第一列开始查找，如果你的条件不包含b列，例如（c,d）,(c),(d) 是无法使用到索引的，当然跨列也无法完全使用到索引，例如（b,d） 只能用到b列索引



## 查看执行计划

### explain 命令

![image-20200325200724851](http://q8sats5bw.bkt.clouddn.com/image-20200325200724851.png)

各列的含义

id: select 查询的标识符，每一个select都会自动分配一个唯一标识符

select_type: select 查询的类型

table: 查询的是哪个表

type: join类型

possible_key: 此次查询中可能用到的索引

key: 此次查询中确切用到的索引

ref: 哪一个字段或者常数与key一起被使用

rows:显示此次查询一共扫描了多少行，这个是个估计值

filtered:此次查询所过滤的数据的百分比

extra:额外的信息

几个重要参数详解：

[select_type:]() 

simple 表示不需要union操作或者不包含子查询的简单select查询。有连接查询时，外层的查询为simple，且
只有一个  

primary 一个需要union操作或者含有子查询的select，位于最外层的单位查询的select_type即为primary。且只
有一个  

union union连接的两个select查询，第一个查询是dervied派生表，除了第一个表外，第二个以后的表
select_type都是union  

dependent union 与union一样，出现在union 或union all语句中，但是这个查询要受到外部查询的影响  

union result  包含union的结果集，在union和union all语句中,因为它不需要参与查询，所以id字段为null  

subquery  除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery  

dependent subquery 除了from字句中包含的子查询外，其他地方出现的子查询都可能是subquery  

derived from字句中出现的子查询，也叫做派生表，其他数据库中可能叫做内联视图或嵌套select  

[type:]()

显示的是单位查询的连接类型或者理解为访问类型，访问性能依次从好到坏

system：只有一行数据或者空表

const：使用唯一索引或者主键索引

eq_ref ： 多表关联，等值连接，等值连接的两个表的列是唯一索引或者主键列

ref： 多表关联，等值连接，等值连接两个表的列是非唯一索引列

fulltext：全文索引检索，全文索引的优先级很高，如果全文索引和普通索引同时存在时，mysql不管代价，会优先使用到全文索引

ref_or_null：与ref 一样，只是增加了null值得比较

unique_subquery： 用于where 中 in 形式的子查询，子查询返回不重复唯一值

index_subquery：用于in形式子查询使用到辅助索引或者in常数列表，子查询可能返回重复值，可以使用索引将子查询去重

range ： 索引范围扫描，常见于使用> < null between in ,like 等运算符的查询中

index_merge：查询使用到两个以上索引，最后取交集或者并集，常见and 、or 的条件使用了不同的索引

index：select 结果中使用到了索引，会显示index，全部索引扫描一遍

All 全表扫描数据文件，然后再在server层进行过滤返回符合要求的记录

注意：除了All之外其他的type 都能用到索引

除了index_merge之外，其他的type只可以用到一个索引

最少使用到range级别



key_len:

用于处理查询的长度，如果是单列索引，那就整个索引长度算进去，如果是组合索引，那么查询不一定用到所有的索引列，具体用到多少列的索引，这里会计算进去，没有用到的不会计算进去

key_len 只计算where 条件的索引长度，而排序和分组如果也用到这个索引，是不会计算到key_len中去的



ref： 如果是等值连接，显示const，如果是连接查询，被驱动的表的查询计划这里会显示驱动表的关联字段

如果是条件使用了表达式或者函数，或者条件列发生了内部隐式转换，这里可能显示为func



extra:

这个列包含不适合在其他列中显示单十分重要的额外的信息，这个列可以显示的信息非常多，有几十种
using index（重要）
查询时不需要回表查询，直接通过索引就可以获取查询的结果数据。(索引覆盖)
表示相应的SELECT查询中使用到了覆盖索引（Covering Index），避免访问表的数据行，效率不
错！
如果同时出现Using Where ，说明索引被用来执行查找索引键值
如果没有同时出现Using Where ，表明索引用来读取数据而非执行查找动作。  

![image-20200325210129226](http://q8sats5bw.bkt.clouddn.com/image-20200325210129226.png)

using where（重要）
表示Mysql将对storage engine提取的结果进行过滤，过滤条件字段无索引；  

![image-20200325210114664](http://q8sats5bw.bkt.clouddn.com/image-20200325210114664.png)

### 索引下推 （mysql 5.6）

using index condition(重要)  ：索引下推

Using index condition 会先条件过滤索引，过滤完成之后找到所有符合索引条件的数据行，然后where子句的其他条件去过滤这些数据行



因为mysql 架构的关系，分成了 server 层 和 引擎层，这才有了所谓的索引下推的说法  Index Condition PushDown

其实就是实现了index filter技术，将原来的在server层进行的
table filter中可以进行index filter的部分，在引擎层面使用index filter进行处理，不再需要回表进行
table filter。
Index Condition Pushdown(ICP)是MySQL 5.6中新特性，是一种在存储引擎层使用索引过滤数据的
一种 优化方式。ICP可以减少存储引擎访问基表的次数以及MySQL服务器访问存储引擎的次数。  

要想深入理解 ICP 技术，必须先理解数据库是如何处理 where 中的条件的。
对 where 中过滤条件的处理，根据索引使用情况分成了三种：index key, index filter, table filter
\1. index key
用于确定SQL查询在索引中的连续范围(起始范围+结束范围)的查询条件，被称之为Index Key。由于一
个范围，至少包含一个起始与一个终止，因此Index Key也被拆分为Index First Key和Index Last
Key，分别用于定位索引查找的起始，以及索引查询的终止条件。也就是说根据索引来确定扫描的范
围。
\2. index filter
在使用 index key 确定了起始范围和介绍范围之后，在此范围之内，还有一些记录不符合where 条件，
如果这些条件可以使用索引进行过滤，那么就是 index filter。也就是说用索引来进行where条件过
滤。
\3. table filter
where 中的条件不能使用索引进行处理的，只能访问table，进行条件过滤了。
也就是说各种各样的 where 条件，在进行处理时，分成了上面三种情况，一种条件会使用索引确定扫
描的范围；一种条件可以在索引中进行过滤；一种必须回表进行过滤；
如何确定哪些where条件分别是 index key, index filter, table filter？
在 MySQL5.6 之前，并不区分Index Filter与Table Filter，统统将Index First Key与Index Last
Key范围内的索引记录，回表读取完整记录，然后返回给MySQL Server层进行过滤。
而在MySQL 5.6（包含）之后，Index Filter与Table Filter分离，Index Filter下降到InnoDB的
索引层面进行过滤，减少了回表与返回MySQL Server层的记录交互开销，提高了SQL的执行效
率。
所以所谓的 ICP 技术，其实就是 index filter 技术而已。只不过因为MySQL的架构原因，分成了
server层和引擎层，才有所谓的“下推”的说法。所以ICP其实就是实现了index filter技术，将原来
的在server层进行的table filter中可以进行index filter的部分，在引擎层面使用index filter进行处
理，不再需要回表进行table filter。
不使用ICP扫描的过程
storage层：
只将满足index key条件的索引记录对应的整行记录取出，返回给server层。
server 层：
对返回的数据，使用后面的where条件过滤，直至返回最后一行。



演示如下：  

student 建立组合索引

student 表 

![image-20200325214356179](http://q8sats5bw.bkt.clouddn.com/image-20200325214356179.png)

score 和 course 的组合索引

![image-20200325214320957](http://q8sats5bw.bkt.clouddn.com/image-20200325214320957.png)

组合索引

执行如下命令：

```sql
explain select * from student where score = 81 and  course like '%文' and name like '%燕';
```

![image-20200325214535691](http://q8sats5bw.bkt.clouddn.com/image-20200325214535691.png)

此处使用到了索引下推

ICP使用的条件

只能用于二级索引

explain 显示的执行计划的type 值 （join 类型）为range ，ref，eq_ref 或者ref_or_null

对于InnoDB表，ICP仅仅用在二级索引

### 索引失效

1.全值匹配我最爱

![image-20200325215524820](http://q8sats5bw.bkt.clouddn.com/image-20200325215524820.png)

2.最左前缀法则：带头索引不能死，中间索引不能断

3.不要在索引上做计算

![image-20200325215854724](http://q8sats5bw.bkt.clouddn.com/image-20200325215854724.png)

4.范围条件右边的列失效

索引如下：

![image-20200325220356301](http://q8sats5bw.bkt.clouddn.com/image-20200325220356301.png)

正确使用全部索引，顺序正确

![image-20200325220608077](http://q8sats5bw.bkt.clouddn.com/image-20200325220608077.png)

使用范围查询 执行结果如下：

![image-20200325220526245](http://q8sats5bw.bkt.clouddn.com/image-20200325220526245.png)

5.尽量使用索引覆盖

索引覆盖 和 最左前缀 冲突问题

这个表的所有数据都建立了索引，那么select * 会用到索引覆盖，此时将不再走最左前缀匹配原则。会发现最侧前缀不生效了，默认使用到了索引。

show index from student;

(name,score,index)

如果 select *  from student where  sorce = 81;

6.索引字段上不要使用 !=  < > 等符号，会导致索引失效，转向全表扫描

7.索引字段不要判断null

8索引字段使用like 不以通配符开头。

![image-20200325222852558](http://q8sats5bw.bkt.clouddn.com/image-20200325222852558.png)

![image-20200325222910560](http://q8sats5bw.bkt.clouddn.com/image-20200325222910560.png)

9.索引字段是字符串要加 单引号

10.索引字段不要使用 or

![image-20200325223057313](http://q8sats5bw.bkt.clouddn.com/image-20200325223057313.png)

## MYSQL锁和事务

### MYSQL锁机制

#### MYSQL锁介绍

按照锁粒度来说，主要包括三种类型的锁定机制

全局锁：database锁，由SQL Layer层来实现

表级别锁：锁定的是某个表table。由mysql 的 sql layer 层来实现的

行级锁：锁定的是某行数据，也可能是行之间的间隙。由某些存储引擎来实现，例如InnoDB

按照锁的功能来分：共享读锁 和 排他写锁

按照锁的实现方式来分：悲观锁(select for update)和乐观锁（使用某一版本列或者唯一列来进行逻辑控制）

表级锁和行级锁的区别： 	1

表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低；  

行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低，并发度也最高； 

### MYSQL行级锁

mysql行级别锁是由存储引擎来实现的，InnoDB行锁是通过给索引上的索引项加速来实现的，这就意味着，只有通过索引条件检索的数据，InnoDB才能使用行级锁，否则使用表锁

行锁按照锁定范围来说分为三种：

记录锁：锁定表中的一条记录

间隙锁：要么锁住索引记录的中间值，要么锁住第一个索引记录前面的值或者最后一个索引记录后面的值

Next-key Locks ：是索引记录上的记录锁和在索引记录之间的间隙锁的组合。

InnoDB的行级别锁，按照功能来分，分为两种：

共享锁：允许一个事务去读一行，阻止其他事务去获取相同数据集的排他锁

排他锁：允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的共享读锁和排他写
锁。  

InnoDB也实现了意向锁，意向锁是mysql 内部使用的，不需要用户干预

意向锁和排他锁可以共存，意向锁的主要作用是为了全表更新数据时的性能提升，否则在全表更新数据时，需要先检索该范围内是否某些记录有行锁

#### 二阶段锁

传统RDBMS加锁的一个原则，就是2PL (Two-Phase Locking，二阶段锁)。相对而言，2PL比较容易理
解，说的是锁操作分为两个阶段：加锁阶段与解锁阶段，并且保证加锁阶段与解锁阶段不相交。下面，
仍旧以MySQL为例，来简单看看2PL在MySQL中的实现 。

2PL就是将加速和解锁分为两个不同的阶段，

加锁阶段：只加锁，不放锁

解锁阶段：只放锁，不加锁

#### 行锁演示

InnoDB是通过给索引上的索引项加锁来实现的，因此InnoDB这种锁意味着：只有通过索引条件检索的数据，InnoDB才能使用到行级别锁，否则使用表级锁

#### 间隙锁

间隙锁防止的两种情况：

1.防止插入间隙内的数据

2.防止已经有的数据更新为间隙内的数据

### InnoDB架构分析

![image-20200413101532015](http://q8sats5bw.bkt.clouddn.com/image-20200413101532015.png)

#### InnoDB磁盘文件

系统表空间  用户表空间 redo log文件和归档文件

二进制文件binlog等文件是Mysql Server层维护的文件，所以没有加入InnoDB磁盘文件中

系统表空间只要包括 InnoDB数据字典（元数据以及相关对象）,double write buffer,change buffer,undo logs的存储区域

系统表空间也包含任何用户在系统表空间创建的表数据和索引数据

系统表空间由一个或者多个数据文件组成

默认情况下： 一个初始大小为10M,命名为ibdata1的系统数据文件在mysql的data目录下被创建。用户可以使用innodb_data_file_path对数据的大小和数量进行配置

用户表空间只存储表数据，索引以及插入缓冲bittmap等信息，其余信息还是默认存放在系统表空间中。

哪些文件是系统的重做日志文件： ib_logfile0 ib_logfile1

为了得到更高的可靠性，用户可以设置多个镜像日志组，将不同的文件组放在不同的磁盘上，以此来提高重做日志的高可用性

每个InnoDB存储引擎至少有1个重做日志文件组(group)，每个文件组下至少有2个重做日志文件，如默
认的ib_logfile0和ib_logfile1。
在日志组中每个重做日志文件的大小一致，并以循环写入的方式运行。
InnoDB存储引擎先写入重做日志文件1，当文件被写满时，会切换到重做日志文件2，再当重做日
志文件2也被写满时，再切换到重做日志文件1。 

#### Buffer pool缓冲池

InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理。但是由于CPU速度
和磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池记录来提高数据库的的整体性
能。

具体来看，缓冲池中缓存的数据页类型有：索引页、数据页、undo页、插入缓冲(insert
buffer)、自适应哈希索引(adaptive hash index)、InnoDB存储的锁信息(lock info)和数据字典信
息(data dictionary）  

在架构图上可以看到，InnoDB存储引擎的内存区域除了有缓冲池之外，还有重做日志缓冲和额外内存
池。InnoDB存储引擎首先将重做日志信息先放到这个缓冲区中，然后按照一定频率将其刷新到重做日
志文件中。重做日志缓冲一般不需要设置的很大，该值可由配置参数innodb_log_buffer_size控制。 

##### 数据页和索引页

 InnoDB存储引擎工作时，需要以Page页为最小单位去将磁盘中的数据加载到内存中，与数据库相关的
所有内容都存储在Page结构里。
Page分为几种类型，数据页和索引页就是其中最为重要的两种类型。  

##### 插入缓冲

主要是针对辅助索引的数据插入文件而设计的

我们都知道，在InnoDB引擎上进行插入操作时，一般需要按照主键顺序进行插入，这样才能获得较高
的插入性能。当一张表中存在次要索引时，在插入时，数据页的存放还是按照主键进行顺序存放，但是
对于次要索引叶节点的插入不再是顺序的了，这时就需要离散的访问次要索引页，由于随机读取的存在
导致插入操作性能下降。
InnoDB为此设计了Insert Buffer来进行插入优化。对于次要索引的插入或者更新操作，不是每一次都
直接插入到索引页中，而是先判断插入的非主键索引是否在缓冲池中，若在，则直接插入；若不在，则
先放入到一个Insert Buffer中。看似数据库这个非主键的索引已经插到叶节点，而实际没有，这时存放
在另外一个位置。然后再以一定的频率和情况进行Insert Buffer和非聚簇索引页子节点的合并操作。这
时通常能够将多个插入合并到一个操作中，这样就大大提高了对于非聚簇索引的插入性能 。

##### 自适应hash索引

哈希（hash）是一种非常快的查找方法，一般情况下查找的时间复杂度为O（1）。常用于连接（join）操作

InnoDB存储引擎会监控对表上索引的查找，如果观察到建立哈希索引可以带来速度的提升，则建立哈希索引，所以称之为自适应（adaptive）的。

自适应哈希索引通过缓冲池的B+树构造而来，因此建立的速度很快。而且不需要将整个表都建哈希索引，InnoDB存储引擎会自动根据访问的频率和模式来为某些页建立哈希索引。

#### 内存数据落盘分析

![image-20200413110918895](http://q8sats5bw.bkt.clouddn.com/image-20200413110918895.png)

InnoDB内存缓冲池中的数据要完成持久化，通过两个流程来实现，一个是脏页落盘，一个是redolog 预写日志

write ahead log  和 force log at commit 来实现事务隔离级别下数据的持久化

WAL要求数据的变更写入到磁盘前，首先必须将内存中的日志写入到磁盘；
Force-log-at-commit要求当一个事务提交时，所有产生的日志都必须刷新到磁盘上，如果日志刷
新成功后，缓冲池中的数据刷新到磁盘前数据库发生了宕机，那么重启时，数据库可以从日志中
恢复数据  

InnoDB的 innodb_flush_log_at_trx_commit属性可以控制每次事务提交时候的innodb的行为。

当属性值为0时，事务提交时，不会对重做日志进行写入操作，而是等待主线程按时写入；
当属性值为1时，事务提交时，会将重做日志写入文件系统缓存，并且调用文件系统的fsync，将文
件系统缓冲中的数据真正写入磁盘存储，确保不会出现数据丢失；
当属性值为2时，事务提交时，也会将日志文件写入文件系统缓存，但是不会调用fsync，而是让文
件系统自己去判断何时将缓存写入磁盘。  

innodb_flush_log_at_commit是InnoDB性能调优的一个基础参数，涉及InnoDB的写入效率和数
据安全。当参数值为0时，写入效率最高，但是数据安全最低；参数值为1时，写入效率最低，但
是数据安全最高；参数值为2时，二者都是中等水平。一般建议将该属性值设置为1，以获得较高
的数据安全性，而且也只有设置为1，才能保证事务的持久性。  

##### 脏页落盘

在数据库中进行读取操作，将从磁盘中读到的页放在缓冲池中，下次再读相同的页时，首先判断
该页是否在缓冲池中。若在缓冲池中，称该页在缓冲池中被命中，直接读取该页。否则，读取磁
盘上的页。
对于数据库中页的修改操作，则首先修改在缓冲池中的页，然后再以一定的频率刷新到磁盘上。
页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为CheckPoint的
机制刷新回磁盘 。

##### 重做日志落盘

InnoDB存储引擎会首先将重做日志信息先放入重做日志缓冲中，然后再按照一定频率将其刷新到
重做日志文件。重做日志缓冲一般不需要设置得很大，因为一般情况每一秒钟都会讲重做日志缓
冲刷新到日志文件中。可通过配置参数innodb_log_buffer_size控制，默认为8MB  

##### checkPoint检查点机制

什么时候触发：

数据库发生宕机

缓冲池不够用：根据LRU算法会溢出最近最少使用的页，如果此页为脏页，那么需要强制执行CheckPoint,将脏页刷回磁盘

重做日志出现不可用：当前事务数据库系统对重做日志的设计都是循环使用的，并不是让其无限制增大。重做日志可以被重用的部分是指这些重做日志已经不再需要，当数据库发生宕机时，数据库恢复操作不需要这部分的重做日志，因此这部分就可以被覆盖重用。如果重做日志还需要使用，那么必须强制Checkpoint，将缓冲池中的页至少刷新到当前重做日志的位置 。

LSN:(Log Sequence Number)来标记版本，LSN是8字节的数字，每个页有LSN，重做日志中也有LSN，Checkpoint也有LSN 。

##### checkPoint分类

在InnoDB存储引擎的内部，由两种checkPoint，分别为：sharp checkPoint,Fuzzy CheckPoint

sharp checkpoint：在关闭数据库的时候，将buffer pool中的脏页全部刷新到磁盘中。 fuzzy
checkpoint：数据库正常运行时，在不同的时机，将部分脏页写入磁盘。仅刷新部分脏页到磁
盘，也是为了避免一次刷新全部的脏页造成的性能问题。  

fuzzy checkPoint: master Thread CheckPoint,flush lru list checkPoint,async/sync flush checkPoint,dirty page too mush checkPoint



#### Double Write

insert buffer 给存储引擎带来了性能上的提升，double write则给存储引擎带来了数据页的可靠性

![image-20200413132819999](http://q8sats5bw.bkt.clouddn.com/image-20200413132819999.png)

 InnoDB 的Page Size一般是16KB，其数据校验也是针对这16KB来计算的，将数据写入到磁盘是以Page为单位进行操作的。而计算机硬件和操作系统，在极端情况下（比如断电）往往并不能保证这一操作的原子性，16K的数据，写入4K 时，发生了系统断电/os crash ，只有一部分写是成功的，这种情况下就是 partial page write 问题。
很多DBA 会想到系统恢复后，MySQL 可以根据redolog 进行恢复，而mysql在恢复的过程中是检查page的checksum，checksum就是pgae的最后事务号，发生partial page write 问题时，page已经损坏，找不到该page中的事务号，就无法恢复。

Double write 是InnoDB在tablespace上的128个页（page,一个页16k）（2个区） 即 2MB;

double write ，写的共享表空间，实在idbdata 文件中划出连续的2M空间，专门给double write 刷脏页用的，顺序写入速度快，当数据页不完整时，可以从 double write 存的共享表空间文件中类恢复数据

fsync: 函数同步内存中所有修改的数据到存储设备

### MVCC

#### 隔离级别

读未提交

会发生脏读： 一个事务读取到另一个事务没有提交的数据

读已提交

会发生不可重复读： 一个事务因为读取到另一个事务已提交的update。导致对同一条记录读取两次以上的结果不一致

可重复读

会发生幻读：一个事务因为读取到另外一个事务已经提交的delete 或者 insert 数据。导致对同一张表读取两次以上的结果不相同

串行化：读写都加锁。

丢失更新问题

数据库事务并发问题需要使用并发控制机制去解决，并发控制机制很多，最常见的时锁和MVCC



##### 版本链

回滚段undo log

insert undo log 和 update undo log

insert undo log的操作记录只对事务本身可见，其他事务记录时不可见的，所以insert undo log 可以在事务提交后直接删除而不需要进行purge 操作

update undo log: 是 update 或者 delete 操作中产生的undo log。因为会对已经存在的记录产生影响，所以提供了MVCC机制，因此 update undo log 不能在事务提交的时候就删除，而是将事务提交时放入history list 中，等待purge线程进行最后的删除操作



当事务2使用UPDATE语句修改该行数据时，会首先使用排他锁锁定改行，将该行当前的值复制到
undo log中，然后再真正地修改当前行的值，最后填写事务ID，使用回滚指针指向undo log中修
改前的行。  

![image-20200413141629959](http://q8sats5bw.bkt.clouddn.com/image-20200413141629959.png)

InnoDB行记录中有3个影藏字段：分别对应该行的rowId,事务号 db_trx_id 和 回滚指针 db_roll_ptr

![image-20200413143156975](http://q8sats5bw.bkt.clouddn.com/image-20200413143156975.png)

#### ReadView

对于使用Read UNCommitted 隔离级别的事务来说，直接读取记录的最新版本就好了

对于使用Serializable隔离级别的事务来说，使用加锁的方式来访问记录

对于使用Read Committed 和 Repeatable Read 隔离级别的事务来说，就需要使用到我们所说的版本链了

核心问题就是：需要判断一下版本链中的哪个版本是当前事务可见的。所以设计 InnoDB 的设计
者提出了一个ReadView的概念，这个 ReadView 中主要包含当前系统中还有哪些活跃的读写事
务，把它们的事务id放到一个列表中，我们把这个列表命名为为m_ids。
这样在访问某条记录时，只需要按照下边的步骤判断记录的某个版本（版本链中的版本）是否可
见：

1. 如果被访问版本的 trx_id 属性值小于 m_ids 列表中最小的事务id，表明生成该版本的事务在生成
   ReadView 前已经提交，所以该版本可以被当前事务访问。

2. 如果被访问版本的 trx_id 属性值大于 m_ids 列表中最大的事务id，表明生成该版本的事务在生成
   ReadView 后才生成，所以该版本不可以被当前事务访问。

3. 如果被访问版本的 trx_id 属性值在 m_ids 列表中最大的事务id和最小事务id之间，那就需要判断
   一下 trx_id 属性值是不是在 m_ids 列表中，如果在，说明创建 ReadView 时生成该版本的事务还
   是活跃的，该版本不可以被访问；如果不在，说明创建 ReadView 时生成该版本的事务已经被提
   交，该版本可以被访问。

   在Mysql 中，Read Committed 和 Repeatable read 隔离级别的一个最大区别就是它们生成ReadView 的时机不同

   ##### 

   

##### Read Committed

每次读取数据前都会生成一个ReadView

比方说现在系统里有两个 id 分别为 100 、 200 的事务在执行：  

![image-20200413145029550](http://q8sats5bw.bkt.clouddn.com/image-20200413145029550.png)

![image-20200413145040055](http://q8sats5bw.bkt.clouddn.com/image-20200413145040055.png)

假设现在有一个Read Committed隔离级别的事务开始执行

```sql
# 使用ReadCommitted隔离级别的事务
Begin:
 # select1 : transaction 100 ,200 未提交
 select * from t where id =1; # 得到的列C的值为‘刘备’
```

![image-20200413145342189](http://q8sats5bw.bkt.clouddn.com/image-20200413145342189.png)

##### Repeatable read

在事务开始之后第一次读取数据时产生一个ReadView

![image-20200413145500654](http://q8sats5bw.bkt.clouddn.com/image-20200413145500654.png)

![image-20200413150058674](http://q8sats5bw.bkt.clouddn.com/image-20200413150058674.png)

![image-20200413150120341](http://q8sats5bw.bkt.clouddn.com/image-20200413150120341.png)

![image-20200413150154893](http://q8sats5bw.bkt.clouddn.com/image-20200413150154893.png)

![image-20200413150223919](http://q8sats5bw.bkt.clouddn.com/image-20200413150223919.png)

![image-20200413150259083](http://q8sats5bw.bkt.clouddn.com/image-20200413150259083.png)

Read committed 和 Repaatable read 这两个隔离级别最大的不同在于生成ReadView的时机不同，Read committed在每次进行查询操作前都会生成一个Readview，而Repeatable read只在第一次进行select 操作前生成一个readview，之后的每次查询操作都重复使用这个readview

##### 当前读和快照读

简单的select 操作，属于快照读，不加锁

SELECT * from table where ?;

当前读：特殊的读操作，查询/更新/删除操作，属于当前读，需要加锁。

select * from table where ? lock in share mode;

select * from table where ? for update;

insert into table values()

update table set ? where ?

delete from table where ?

update 操作的具体流程：

1.  当Update SQL被发给MySQL后，MySQL Server会根据where条件，读取第一条满足条件的记
   录，然后InnoDB引擎会将第一条记录返回，并加锁 (current read)。

2. 待MySQL Server收到这条加锁的记录之后，会再发起一个Update请求，更新这条记录。
3. 一条记录操作完成，再读取下一条记录，直至没有满足条件的记录为止。

##### 一致性非锁定读

一致性非锁定读指的是InnoDB存储引擎通过多版本并发控制(MVCC)读取当前数据库中行数据的方式，如果读取的行正在执行update 或者delete 操作，这个读取操作不会因此去等待行锁的释放，相反的，InnoDB会去读取行的一个快照。

#### 事务总结

事务的隔离性由多版本并发控制和锁来实现，而原子性，持久性和一致性主要通过redo log、undo log 和 force log at commit，redo log用于在崩溃时恢复数据，undo log用于对事务的影响进行撤销，也可以用于多版本控制。而Force Log at Commit机制保证事务提交后redo log日志都已经持久化  

![image-20200413164821872](http://q8sats5bw.bkt.clouddn.com/image-20200413164821872.png)

##### 间隙锁

对于RR隔离级别来说，当前读采用的是 GAP间隙锁来解决的

## MYSQL性能分析

### 性能分析的思路

1. 首先使用 【慢查询日志】功能，去获取所有查询时间比较长的SQL语句

2. 其次【查看执行计划】查看有问题的SQL的执行计划

3. 最后使用【show profile[s]】查看问题SQL的性能使用情况

   

### 慢查询日志开启

- 查看是否开启了慢查询功能

  ```shell
  mysql>show variables like '%slow_query%';
  
  mysql>show variable like 'long_query_time%';
  ```

- 暂时开启慢查询日志，重启MySQL恢复

```shell
set global slow_query_log = ON;
set long_query_time =1;
```

- 永久开启 vim /etc/my.cnf

  ```shell
  slow_query_log=1
  long_query_time=1
  slow_query_log_file=/var/lig/mysql/slow.log
  ```

  慢查询分析工具mysqldumpslow

#### profile 功能开启

```sql
# 查看是否开启了profile功能
mysql>select @@profiling;
mysql>show varibales like '%profil%';
```

开启profile功能

set profiling  = 1  --1 是开启，0是关闭

### MYSQL性能优化

##### 服务器层面的优化

innodb_buffer_pool_size 设置为总内大小的3/4 或者 4/5

##### 服务器硬件优化

centos 系统针对mysql 的参数优化

##### sql 层面的优化

- 优化索引1

  搜索字段，排序字段，select 查询列创建合适的索引

  创建组合索引

  尽量使用索引覆盖

  order by group by 尽量使用到索引

  索引长度尽量短

  索引不能频繁更新

- limit优化

  如果预先知道查询结果为1条，使用limit，可以停止全表扫描

- 其他查询优化

  小表驱动大表，建议使用left join，使用join的话，必然是全表扫描，第一张表必须全表扫描，以少关联多可以减少扫描的次数

  mysql 在使用不等于（!= 或者 <>） 的时候无法使用导致全表扫描。在查询的时候，如果对索引使用不等于操作，会破坏索引，导致索引失效，进行全表扫描

  尽量不要使用count（*） ，而使用count（主键）

  count(*) : 查询行数，会遍历所有的行，所有的列

  count(列): 会指定查询列不为null的行数

  count（伪列）：例如count（1）

- join 两张表的关联字段最好都建立索引，而且最好字段类型一致

- where 不要使用 1=1 not in 语句 （建议使用 not exists）

- 不用sql 的内置函数，因为内置函数不会建立查询缓存

- 合理利用慢查询日志，explain执行计划查询，show profile查看sql 执行时的资源使用情况

## MYSQL集群

主从复制

binlog reay-log（中继日志）：主从复制环境中产生的日志，主要作用是为了从机可以从中继日志中获取到从主机同步过来的SQL语句，然后在从机中执行

主从同步实际上是从服务器执行reay-log 日志中的SQL语句，将数据保存在从服务器上，实现数据同步备份。

主从同步延迟问题

## MYSQL分库分表

### 数据切分

- 垂直切分：按照业务模块来切，将不同模块的表切分到不同的数据库中去，
  - 垂直分库：按照业务分
  - 垂直分表：数据表的列为依据进行切分，大表拆小表的模式 （详情表的模式）
    - 优势：业务解耦，缓解数据库压力
    - 劣势：提升了开发复杂度，分布式事务管理难度增加

- 水平切分：将一张大表按照一定的切分规则切分，按照行切分成不同的表，或者切分到不同的库中去
  - 数据量大小的切分，分为库内分表，分库分表

切分规则

- 按照ID取模
- 按照日期
- 按照范围
- hash 取模

分库分表要解决对问题：

强一致性事务（同步）

最终一致性事务 （异步）

分布式主键生成问题：

redis incr 命令

数据库（生成主键）

UUID

snowflake算法

分库分表实现技术

sharding-jdbc

mycat

### sharding -jdbc

自增主键生成策略 UUID和雪花算法

## MYSQL缓存

### MySQL查询缓存的缓存机制

缓存主要包括关键字缓存（key cache）和 查询缓存（query cache）

mysql 查询缓存的缓存机制（query cache）简单来说就是缓存sql 语句和查询结果。如果运行相同的sql 语句，服务器直接从缓存中提取结果，而不是再去解析和执行sql。这些缓存会被会话共享，一旦某一个客户端建立了查询缓存，其他客户端再发送相同的sql语句的查询也可以使用到这些缓存。

### MYSQL查询缓存的工作原理

当mysql收到传入的sql语句时，它首先和先前解析过的sql语句进行比较，如果发现相同，则返回已缓存数据。

如下几种情况被认为是不同的：

1.sql 语句相同，大小写不同

2.如果一条sql语句是另外一条sql语句的子串，类似下面的情况，第02行的语句不会被缓存；如果sql语句是存储过程、触发器或者事件内部的一条语句，同样也不会被缓存。查询缓存也受到权限的影响，对于没有权限访问数据库中数据的用户，即使输入了同样的sql语句，缓存中的数据也会无权访问

01 SELECT 课程名 FROM KC where 学分 in(

02 SELECT 学分 FROM KC

03 );

 当传入的sql语句被认为是存在缓存的情况下，系统会修改mysql的一个状态变量Qcache_hits，并将其值增加1，可以运行语句来查看qcahce_hits的值，如下：

![](http://q8sats5bw.bkt.clouddn.com/image-20200408142611385.png)

### 查看MYSQL的缓存信息

![image-20200408143253604](D:\必看\MYSQL\MYSQL\mysql.assets\image-20200408143253604.png)

![](http://q8sats5bw.bkt.clouddn.com/image-20200408143343566.png)

### MYSQL查询缓存的配置和使用

1.配置查询缓存

vim /etc/my.cnf

```shell
[mysql]
port = 3306
socket = /tmp/mysql.sock
query_cache_type =1
query_cache_size=268435456
query_cache_limit=1048576
```

Query_cache_type可以是0,1,2，0代表不使用缓存，1代表使用缓存，2代表根据需要使用

```shell
// 查询缓存会生成碎片，可以通过下面命令来清理碎片
mysql> flush query cache;
```