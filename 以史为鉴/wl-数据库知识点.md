#### 一、MySQL逻辑架构

MySQL逻辑架构：

![MySQL内部逻辑结构图](https://user-gold-cdn.xitu.io/2018/7/5/1646976a5ac24dbb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

MySQL查询过程：

![](https://user-gold-cdn.xitu.io/2018/7/5/1646976a5ab7387c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

1. 在实际执行查询之前，如果查询缓存是打开的，先看此条查询语句是否能命中查询缓存，如果能在查询缓存中找到结构，检查一下用户权限就返回，不执行解析、优化、查询等动作。
2. 缓存表（类似hashtable的结构），只有一模一样的查询语句才可能命中缓存。
3. 缓存失效：当缓存相关的表结构发生改变的时候。缓存失效需要系统开销，这造成对数据库写操作性能的损失。另外，由于每次查询都要检查缓存，所有会对读操作的性能也造成影响。
4. 可以通过`SQL_CACHE` 和`SQL_NO_CACHE`  来控制某个查询语句是否需要进行缓存

##### 总结MySQL查询执行步骤
1. 客户端向MySQL服务器发送一条查询请求
2. 服务器首先检查查询缓存，如果命中缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段
3. 服务器进行SQL解析、预处理、再由优化器生成对应的执行计划
4. MySQL根据执行计划，调用存储引擎的API来执行查询
5. 将结果返回给客户端，同时缓存查询结果



#### 二、索引
一般索引的数据结构：B树/B+树

B+树是平衡的多路搜索树。

![](https://user-gold-cdn.xitu.io/2018/7/5/1646976acf9594da?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

索引也要消耗空间，一般索引是以索引文件的形式存放在磁盘上，所以使用索引是需要IO开销的。

MySQL巧妙利用了磁盘预读原理，将一个节点(B+树的节点)的大小设为等于一个页（操作系统内容，磁盘页），这样每个节点只需要一次I/O就可以完全载入。

##### 聚簇索引和非聚簇索引

1. 定义
    - 聚簇索引：数据行的物理顺序与列值（一般是主键的那一列）的逻辑顺序相同，一个表中只能拥有一个聚集索引。MySQL InnoDB 的主键索引即为聚簇索引。
    - 非聚簇索引：该索引中索引的逻辑顺序与磁盘上行的物理存储顺序不同，一个表中可以拥有多个非聚集索引。
2. 聚簇索引的特点
    - 索引的叶子节点就是对应的数据节点
    - 易于进行范围查询
    - 方便对聚簇索引的列进行全盘扫描

##### MySQL索引失效的场景

1. 查询条件中，在索引列上进行运算（+ - * /）或者使用 MySQL 内部函数（rount等）
2. 小表查询
3. 隐式转换导致索引失效：误传不同类型的字段给索引列
4. 如果MySQL估计使用索引比全表扫描更慢，则不适用索引，比如对均匀索引值分布的范围查询
5. memory 存储引擎使用hash索引，只有在 = in 的查询条件中才会使用索引
6. 用 or 分隔开的条件，如果 or 前条件中的列有索引，而后面的列中没有索引，那么涉及的索引都不会被用到。
7. 对于联合索引，如果要使用的索引列不是联合索引的第一部分，则索引对加快查询无效
8. 如果 where 子句的查询条件里使用了 LIKE、REGEXP，MySQL只在搜索模板的第一个字符不是通配符（%）的情况下才会使用索引。
9. order by 操作中，MySQL只有在排序条件不是一个查询表达式的情况下才使用索引
10. not in 和 <> 操作都不会使用索引而将进行全表扫描
11. 索引不会包含有 null 值的列

[MySQL索引使用注意事项](https://www.cnblogs.com/duanxz/p/5244703.html)

##### 索引分析方法
```
show status like 'Handler_read%';
+-----------------------+--------+
| Variable_name         | Value  |
+-----------------------+--------+
| Handler_read_first    | 9      |
| Handler_read_key      | 16     |  // 这个值高表示索引正在工作
| Handler_read_last     | 0      |
| Handler_read_next     | 680908 |
| Handler_read_prev     | 0      |
| Handler_read_rnd      | 0      |
| Handler_read_rnd_next | 935519 |  // 这个值高表示查询低效，需要建索引
+-----------------------+--------+
7 rows in set (0.00 sec)
```



#### 三、InnoDB和MyISAM存储引擎的区别
MySQL 5.1之前，MyISAM是默认存储引擎，后换成InnoDB。
> 两种类型最主要的差别就是InnoDB支持事务处理与外键和行级锁。而MyISAM不支持。所以MyISAM往往就容易被人认为只适合在小项目中使用。

- 随着CPU核心数的增加，InnoDB的吞吐量显著上升， MyISAM 的吞吐量则没有明显变化，因为后者的表级锁限制了读和写的吞吐量。
- MyISAM是一种非事务性的引擎，使得MyISAM引擎的MySQL可以提供高速存储和检索，以及全文搜索能力，适合数据仓库等查询频繁的应用。
- 每个MyISM表在磁盘上存成三个文件：.frm(framework) .myd(my data) .myi(my index)
- InnoDB 存储引擎在磁盘上存储数据和日志文件。
- InnoDB行锁是通过给索引项加锁来实现的，即只有通过索引条件检索数据，InnoDB才使用行级锁，否则将使用表锁。
- 当InnoDB不能确定要扫描的范围，也会使用表级锁，比如字符串的模糊匹配。
- 清空整个表时，InnoDB是一行一行的删除，效率非常慢。MyISAM则会重建表。
- 对于AUTO_INCREMENT类型的字段，InnoDB中必须包含只有该字段的索引，但是在MyISAM表中，可以和其他字段一起建立联合索引

##### 使用选择
> **MyISAM适合：** </br>
> 1）做很多count 的计算； </br>
> 2）插入不频繁，查询非常频繁，如果执行大量的SELECT，MyISAM是更好的选择； </br>
> 3）没有事务。</br>
> **InnoDB适合：** </br>
> 1）可靠性要求比较高，或者要求事务； </br>
> 2）表更新和查询都相当的频繁； </br>
> 3）如果你的数据执行大量的INSERT或UPDATE，出于性能方面的考虑，应该使用InnoDB表。



#### 四、MySQL数据类型

##### MySQL数据类型的三大类

- 数值
- 日期/时间
- 字符串

[MySQL数据类型及其占用空间大小](http://www.runoob.com/mysql/mysql-data-types.html)

文本存储：
- char：定长，最大255个字符，char(m)定义的列的长度为固定的，当保存CHAR值时，在它们的右边填充空格以达到指定的长度。在存储和检索过程中不进行大小写转换。CHAR字段上索引效率较高。
- varchar：变长，最大65535个字符。varchar(m)定义的列为可变长字符串，保存值时只保存需要的字符数，另加一个或两个字节来记录长度。从空间上考虑，varchar合适。从效率上考虑，char合适。
- text：变长，有字符集的大对象，对大小写不敏感。
- blob：一个能保存可变数量的数据的二进制大对象。BLOB可以存储图片，而text不行，text只能存储纯文本文件。BLOB对大小写敏感。

##### 行存储方式

记录是以行的形式进行保存的，Mysql5.1以后，行的保存格式默认为Compact格式。行记录Compact格式为：

```
| 变长字段，记录记录长度 | null标志位 | 记录头信息 | 列数据1 | 列数据2 | …… |
```

 第一个变长字段是记录这行的总字段长度，如果行记录的字段总长小于255字节，变长字段就占**一个字节**（一个字节有8位，8位的二进制最多能表示到255）。当大于255时，变长字段的长度就是**两个字节**。

 变长字段的长度最大为2个字节。

##### varchar 不同字符集

理论上来讲 varchar 的最大长度是 65535（2^16）字节。

- Utf8中一个字符相当于3个字节
- Gbk中一个字符相当于2个字节
- latin1一个字符相当于一个字节

##### 字段length的含义

- int(2) int(4) int(6) 所能表示的数据范围是一样的，都是4个字节的整数
- length 只在显示时起作用，比如 int(4) 存了123，当你设置zerofill时，会显示 0123



#### 五、数据库最左匹配原则
MySQL 联合索引的最左匹配原则：
> mysql创建联合索引的规则是首先会对联合合索引的最左边的，也就是第一个字段col1的数据进行排序，在第一个字段的排序基础上，然后再对后面第二个字段col2进行排序。其实就相当于实现了类似 order by col1 col2这样一种排序规则。

最左匹配的使用注意点：写sql的时候要把联合索引左边列的筛选写在前面。

联合索引的优点：
1. 减少开销：建一个联合索引(col1,col2,col3)，实际相当于建了(col1),(col1,col2),(col1,col2,col3)三个索引。减少了分别建索引的开销。
2. 覆盖索引：对联合索引(col1,col2,col3)，如果有如下的sql: select col1,col2,col3 from test where col1=1 and col2=2。那么MySQL可以直接通过遍历索引取得数据，而无需回表，这减少了很多的随机io操作。
3. 效率高：索引列越多，通过索引筛选出的数据越少。单值索引要进行多次筛选的，联合索引只需要一次筛选。



#### 六、数据库操作
##### 数据库 join 和 join on 的区别？
- join on 是有条件连接，直接 join 是笛卡儿积
- inner join = join
- full join / full outer join 结果是笛卡儿积

##### 数据库删除一个表的方法？
Drop、Delete、Truncate
- Drop：用于删除表，同时将表的结构、属性、索引也删除
- Delete：可以删除单行，也可以在保持表结构的情况下删除所有行
- Truncate：仅删除表内数据，不删除表定义，相当于delete删除所有行的情形
- delete属于数据操作语言（DML），不能自动提交事务，需commit提交
- Truncate不会激活触发器

使用场合：
- 当你不再需要该表时， 用 drop;
- 当你仍要保留该表，但要删除所有记录时， 用 truncate;
- 当你要删除部分记录时（always with a where clause), 用 delete.

##### distinct(A)和distinct(A,B,C)的区别？
- 作用于单列，该列属性值不一样就算distinct的
- 作用于多列，所有列全部相同才一样
- distinct 和 count常一起使用，不能count多个字段的distinct结果
- distinct语句中select显示的字段只能是distinct指定的字段

##### 常用函数
count、max、min、distinct、stdev、sum、字符串操作函数等

##### count()函数：SQL Server中的count(1),count(*),count(column_name) 

> In terms of behavior, COUNT(1) gets converted into COUNT(*) by SQL Server, so there is no difference between these. The 1 is a literal, so a COUNT('whatever') is treated as equivalent.

>  COUNT(column_name) behaves differently. If the column_name definition is NOT NULL, this gets converted to COUNT(*). If the column_name definition allows NULLs, then SQL Server needs to access the specific column to count the **non-null values** on the column.

`count(*)`会被SQL优化。如果表中有not null的列建立了非聚簇索引， `count(*)` 会直接对这个索引项进行扫描来统计总行数。

##### select XXX for update

会对选中的行加排他锁。其他事务只能读这些行，不能写这些行。直到 update 的事务commit。

for update仅适用于InnoDB，且必须在事务块(BEGIN/COMMIT)中才能生效。

##### 连接关闭时数据库回滚

当使用事务时，如果有空事务长期存在，会因为事务锁造成大量请求阻塞。
- 数据库端应该监控有事务的连接，当长时间不活动时主动断连回滚；
- 数据库遭遇客户端异常断开的时候，会有自动回滚机制。



#### 七、大表分页查询优化

分页查询操作的执行流程：先查询出所有满足条件的数据，再根据limit取分页数据。

当数据量较大时，越往后的分页速度越慢。

[MySQL 分页查询优化思路](https://claude-ray.github.io/2019/03/11/mysql-pagination/)

##### 优化方法
1. 使用子查询优化，子查询尽可能地使用索引覆盖，而不是查询所有的列，然后根据需要做一个关联操作返回所需的列。
    - 举例子：
    ```
    // 原本的查询语句
    select film_id, description from sakila.film order by title limit 500, 5;
    // 改写后的语句
    select film.film_id, film.description
    -> from sakia.film inner join
    ->  (select film_id from sakila.film order by title limit 50, 5) as lim using(film_id);
    
    // 另一种子查询优化
    // 先定位偏移位置的id，然后往后查询，适用于 id 递增的情况
    select * from orders_history where type=8 and 
    id>=(select id from orders_history where type=8 limit 100000,1) 
    limit 100;
    
    select * from orders_history where type=8 limit 100000,100;
    ```
    - 这样先通过覆盖索引将要查询的id范围缩小，然后通过关联语句回原表查询需要的列信息，避免从原表中取出大量无效的记录

2. 将limit查询转换为已知位置的查询，让 MySQL 通过范围扫描获得对应的结果。例如，如果在一个位置列上有索引，并且预先计算出了边界值，上面的查询就可以改写为：
    ```
    select film_id, description from sakila.film where position between 50 and 54 order by position;
    ```
3. 记录上一次的偏移量，下一次直接从偏移量往后取。比如传参数的时候不传  pageSize 和 pageIndex，而是传 pageSize 和 上一页最后一条记录的id，这样可以用如下语句进行分页查询：
    ```
    select * from table_name where id > last_id limit pageSize;
    ```

#### 另一个点，大表如何快速做行数统计
1. MyISAM存储引擎会将表的总行数存在索引里，所以count(*) 可以很快得到查询结果，但 InnoDB 存储引擎需要做全表/索引扫描才能得到结果。
2. 通过查询 mysql 的 information_schema 数据库中 INNODB_SYS_TABLESTATS 表,它记录了 innodb 类型每个表大致的数据行数。
```
use information_schema;
SELECT NUM_ROWS FROM INNODB_SYS_TABLESTATS WHERE `NAME` = 'my_test/user';
```
3. 在内存里/单独的数据库表里维护一个计数器，记录某个表的总行数。每次插入/删除时更新，需要时直接拿出来。（这种方案在有分库分表操作的情况下，扩展性会更好）


#### 八、MySQL InnoDB MVCC 实现原理
##### 1. 数据库通过什么方式保证事务的隔离性？
加锁。

![](https://github.com/May-Wang-666/picture-hut/blob/pictures/self/MySQL%20%E9%94%81%E5%88%86%E7%B1%BB.png?raw=true)

- 行锁和表锁
    - 行锁都是作用在索引上面的，只有 SQL 语句命中索引时，才会使用行锁，否则使用表锁
    - todo：行锁是用什么实现的？标志位吗？
- 意向锁
    - 为何存在：当一个事务要获取表级锁时，必须先检查当前的每个行锁中是否有写锁，方式是进行索引遍历。如果有连续多个事务都需要加表级锁，每个事务都要去进行索引遍历，是很低效的
    - 所以MySQL自己维护一个表级的意向锁，当有事务想要加表锁时，需要先获取到对应的意向锁才可以。意向锁之间互不冲突，但表级锁和意向锁会有冲突，详见下文。
    - 什么时候加意向锁：所有锁操作（不管是行级锁还是表级锁）之前都要先加对应的意向锁（意向共享/意向排他）

意向锁的兼容互斥性：

| ---      | 意向共享（IS） | 意向排他（IX） |
| -------- | -------------- | -------------- |
| 意向共享 | 兼容           | 兼容           |
| 意向排他 | 兼容           | 兼容           |
| ---        | 意向共享（IS） | 意向排他 |
| ---------- | -------------- | -------- |
| 表级共享锁 | 兼容           | 互斥     |
| 表级排他锁 | 互斥           | 互斥     |


意向锁不会与行级的共享 / 排他锁互斥！！！

参考：[详解 MySql InnoDB 中意向锁的作用](https://juejin.im/post/5b85124f5188253010326360)

##### 2. MVCC 解决什么问题？
频繁加解锁操作会对数据库性能造成影响。使用 MVCC 减少加锁操作（写操作加锁，读时不加锁），通过多版本控制使读写操作可以并发进行。

##### 3. Innodb MVCC实现的核心知识点
1. 事务的版本号
    - 每次事务开启前都会从数据库获得一个自增长的事务ID，可以从事务ID判断事务的执行先后顺序
2. 表格的隐藏列
    - DB_TRX_ID: 记录操作该数据的事务ID
    - DB_ROLL_PTR：指向上一个版本数据在 undo log 里的位置指针
    - DB_ROW_ID: 隐藏ID ，当创建表没有合适的索引作为聚集索引时，会用该隐藏ID创建聚集索引
3. undo log
    - 用于记录数据被修改之前的日志，在表信息修改之前先会把数据拷贝到undo log 里，当事务进行回滚时可以通过undo log 里的日志进行数据还原
    - 作用：事务 rollback 操作和 MVCC 快照读
4. read view
    - InnoDB 中每个事务执行前都会获取一个 read view，里面保存了当前数据库系统种正处于活跃状态的事务id。（即：还没有提交的事务 ID）简单地说，这个 view 里面保存了不能被当前事务看到的事务 id 列表。
    - 作用：针对有共享数据两个事务，保证低版本未提交的修改不会被高版本看到。（避免读未提交导致的脏读和不可重复读）
    - Read view 的几个重要属性：
        - trx_ids: 当前系统活跃(未提交)事务版本号集合
        - low_limit_id: 创建当前read view 时当前系统活跃的事务最大版本号
        - up_limit_id: 创建当前read view 时系统活跃的事务最小版本号
        - creator_trx_id: 创建当前read view的事务版本号
    - Read view 查询匹配规则：
        1. 数据事务 ID < up_limit_id，直接返回
        2. 数据事务 ID > low_limit_id，不返回，查找 undo log
        3. up_limit_id < 数据事务 ID < low_limit_id，用事务数据 ID 与 trx_ids 和 creator_trx_id 进行比较，如果不在活跃事务列表（已提交）或是当前创建 read view 的事务，就返回，否则不返回。
    - 读取数据时，利用read view 判断当前数据是否应该被返回。当数据的事务 ID 不满足 read view 条件时候，从 undo log 里面获取数据的历史版本，比较历史版本的事务 ID 直到满足 read view 条件为止。
    - 不同数据库隔离级别中 read view 的工作：
        1. read uncommit：不获取 read view
        2. read commit：每次查询都获取一个 read view，可能导致幻读
        3. repeatable read：只在事务开始时获取一次 read view

隐藏列 和 undo log：

![](https://pic1.zhimg.com/80/v2-1daaeab59495ff3378dae24ea21dc158_hd.jpg)

MVCC 快照读流程：

![](https://pic3.zhimg.com/80/v2-a086ef515e7d0a023ca3cfcc5759c7f6_hd.jpg)

MVCC 实现原理参考：https://zhuanlan.zhihu.com/p/52977862



#### 九、MySQL 主从同步
![image](https://upload-images.jianshu.io/upload_images/17609428-e34b898e3c0f1adc.png?imageMogr2/auto-orient/strip|imageView2/2/w/494/format/webp)

基本原理：
- master 开启 bin log，对 master 库的所有写操作都会以事件的形式记录在 bin log 中；
- slave 数据库运行两个线程
    - io 线程与 master 进行连接通信，检测到 master 的日志文件有新增内容，则将其复制到 salve 的 relay log（中继日志） 中；
    - sql 线程读取 relay log 中新增内容，在 slave 上执行，实现主从同步

细节要点：
1. 主服务器负责记录从服务器已经拷贝的最新日志位置。
2. 当一个从服务器连接主服务器时，它通知主服务器从服务器在日志中读取的最后一次成功更新的位置。从服务器接收从那时起发生的任何更新，然后封锁并等待主服务器通知的更新。
3. slave 服务器会在一定时间间隔内对 master 二进制日志进行探测其是否发生改变，如果发生改变，则开始一个  I/O Thread 请求 master 二进制事件，同时主节点为每个I/O线程启动一个dump线程，用于向其发送二进制事件。（不是长连接，而是周期性地询问）
4. 中继日志通常会位于OS的缓存中，所以中继日志的开销很小。

配置要点：
1. master 和 slave 都配置唯一 server id；
2. master 开启 bin log；
3. slave 设置 master 为自己的 MASTER，即同步源；
4. 运行master，运行slave。

##### MySQL支持的复制类型
1. 基于语句的复制：同步 SQL 语句，主服务器上执行的语句，在从服务器上再执行一遍；
2. 基于行的复制：把改变的内容复制过去，而不是把命令在从服务器上执行一遍。从mysql 5.0开始支持；
3. 混合类型的复制：默认采用基于语句的复制，一旦发现基于语句的复制无法精确复制时，采用基于行的复制。

binlog日志格式：

![image](https://img-blog.csdnimg.cn/20190326103256132.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ppU2h1aVNhblFpYW5MaQ==,size_16,color_FFFFFF,t_70)

##### MySQL 主从复制的优点
- 读写分离，slave 负责查询工作，减轻 master 的压力；
- 主从形成备份关系，当主服务器出现问题时，可以快速切换到从服务器，保证高可用性。

##### MySQL binlog
MySQL的二进制日志binlog可以说是MySQL最重要的日志，它记录了所有的DDL和DML语句（除了数据查询语句select）。

二进制的，binlog中记录了：
- 执行的SQL语句；
- 执行者，执行时间，log 日志执行完后的 position；
- 通过 MySQL 自带的 mysqlbinlog 工具才能查看。



#### 十、MySQL 事务日志
![image](https://img-blog.csdnimg.cn/20190701201126826.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0ppU2h1aVNhblFpYW5MaQ==,size_16,color_FFFFFF,t_70)

- redo log：事务中操作的任何数据，将最新的数据备份到一个地方。
- undo log：事务开始之前，在操作任何数据之前,首先将需操作的数据备份到一个地方。

##### redo log
- 事务一边执行一边写 redo log。
- 是为了实现事务的持久性而出现的日志。
- 一旦事务成功提交且数据持久化落盘之后，此时Redo log中的对应事务数据记录就失去了意义，所以老的日志可以被安全覆盖。

##### Undo log 
- log 中的数据可作为数据旧版本快照供其他并发事务进行快照读。
- 是为了实现事务的原子性而出现的产物，在 Mysql innodb 存储引擎中用来实现多版本并发控制。

##### 一个小例子
 假设有A、B两个数据，值分别为1,2，开始一个事务，事务的操作内容为：把1修改为3，2修改为4，那么实际的记录如下（简化）：

```
  A.事务开始.
  B.记录A=1到undo log.
  C.修改A=3.
  D.记录A=3到redo log.
  E.记录B=2到undo log.
  F.修改B=4.
  G.记录B=4到redo log.
  H.将redo log写入磁盘。
  I.事务提交
```