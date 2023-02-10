# 幻读是什么，幻读有什么问题？
## 幻读是什么
### 下面语句序列，是怎么加锁的，加的锁又是什么时候释放的？
* 表结构及初始化

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```
* 执行语句

```sql
begin;
select * from t where d=5 for update;
commit;
```
这个语句会命中d=5这一行，对应的主键id=5，因此在select 语句执行完成后，id=5这一行会加一个写锁，而且由于两阶段协议，这个写锁会在执行commit语句的时候释放。

由于字段d上没有索引，因此这条查询语句会做全表扫描。

其他被扫描到，但是不满足条件的5行记录上，会不会加锁呢？

### 如果只在id=5这一行加锁，其他行不加锁的话，会怎么样？
* 假设场景如下：

| |Session A|Session B|Session C|
| ----- | ----- | ----- | ----- |
|T1|begin;<br>select * from t where d=5 for update;/Q1\*/*<br>result:(5,5,5)| | |
|T2| |update t set d=5 where id=0;| |
|T3|select * from t where d=5 for update;/Q2\*/*<br>result:(0,0,5),(5,5,5)| | |
|T4| | |insert into t values(1,1,5);|
|T5|select * from t where d=5 for update;/Q3\*/*<br>result::(0,0,5),(1,1,5)(5,5,5)| | |
|T6|commit;| | |
Q1、Q2、Q3分别会返回什么结果?

1. Q1只返回id=5这一行
2. 在T2时刻，session B 把id=0这一行的d值改成了5，因此T3时刻，Q2查出来的是id=0和id=5这两行
3. 在T4时刻，session C 又插入一行(1,1,5)，因此T5时刻Q3查出来的是 id=0、id=1和id=5这三行

### 幻读定义
其中，Q3读到id=1这一行的现象，被称为幻读。

也就是说，幻读是指一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。

* 说明

1. 在可重复读隔离级别下，普通的查询是快照读，是不会看到别的事务插入的数据的。因此，幻读在“当前读”下才会出现。
2. 幻读仅专指“新插入的行”。

## 幻读有什么问题
### 语义
Session A 在T1时刻，声明“我要把所有d=5的行锁住，不准别的事务进行读写操作”。实际上，这个语义被破坏了。

* 例子：

| |Session A|Session B|Session C|
| ----- | ----- | ----- | ----- |
|T1|begin;<br>select * from t where d=5 for update;/Q1\*/*<br>result:(5,5,5)| | |
|T2| |update t set d=5 where id=0;<br>update t set c=5 where id= 0;| |
|T3|select * from t where d=5 for update;/Q2\*/*<br>result:(0,0,5),(5,5,5)| | |
|T4| | |insert into t values(1,1,5);<br>update t set c=5 where id=1;|
|T5|select * from t where d=5 for update;/Q3\*/*<br>result::(0,0,5),(1,1,5)(5,5,5)| | |
|T6|commit;| | |

由于在T1时刻，Session A还只是给id=5这一行加了行锁，并没有给id=0这一行加锁。因此Session B在T2时刻，是可以执行这两条update语句的。这样，就破坏了Session A里Q1语句要锁住所有d=5的行的加锁声明。

Session C同理。

### 数据一致性
* 例子：

| |Session A|Session B|Session C|
| ----- | ----- | ----- | ----- |
|T1|begin;<br>select * from t where d=5 for update;/Q1\*/*<br>update t set d=100 where d=5;| | |
|T2| |update t set d=5 where id=0;<br>update t set c=5 where id= 0;| |
|T3|select * from t where d=5 for update;/Q2\*/*| | |
|T4| | |insert into t values(1,1,5);<br>update t set c=5 where id=1;|
|T5|select * from t where d=5 for update;/Q3\*/*| | |
|T6|commit;| | |
在T1时刻添加一个更新语句

执行完成后，数据库的结果：

1. 经过T1时刻，id=5这一行变成（5,5,100）,当然这个结果最终是在T6时刻正式提交的；
2. 经过T2时刻，id=0这一行变成(0,5,5)；
3. 经过T4时刻，表里多了一行（1,5,5）；
4. 其他行跟这个执行序列无关，保持不变。

这样看，数据也没啥问题，但是此时binlog里面的内容如下：

1. T2时刻，Session B事务提交，写入了两条语句；
2. T4时刻，Session C事务提交，写入了两条语句；
3. T6时刻，Session A事务提交，写入了 update t set d=100 where d=5这条语句。
4. 统一放到一起的话，就是这样的：

```sql
update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/

insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/
```
这个语句序列，不论是拿去备库执行，还是以后用binlog来克隆一个库，这三行的结果，都变成了(0,5,100),(1,5,100)和(5,5,100)。

也就是说 id=0 和 id=1 这两行，发生了数据不一致。

* 这个数据不一致是怎么引入的？

是我们假设“select \* from t where d=5 for update 这条语句只给d=5这一行，也就是id=5这一行加锁”导致的。

所以，上面的设定不合理。

* 假设我们改成，扫描过程中碰到的行，也都加上锁，再来看看执行效果。

| |Session A|Session B|Session C|
| ----- | ----- | ----- | ----- |
|T1|begin;<br>select * from t where d=5 for update;/Q1\*/*<br>update t set d=100 where d=5;| | |
|T2| |update t set d=5 where id=0;<br>(blocked)<br>update t set c=5 where id= 0;| |
|T3|select * from t where d=5 for update;/Q2\*/*| | |
|T4| | |insert into t values(1,1,5);<br>update t set c=5 where id=1;|
|T5|select * from t where d=5 for update;/Q3\*/*| | |
|T6|commit;| | |

由于Session A把所有的行都加了写锁，所以Session B在执行第一个update语句的时候就被锁住了。需要等到T6时刻，Session A提交以后，Session B才能继续执行。

这样对于id=0这一行，在数据库里的结果最终还是（0,5,5）。在binlog里面，执行序列是这样的。

```sql
insert into t values(1,1,5); /*(1,1,5)*/
update t set c=5 where id=1; /*(1,5,5)*/

update t set d=100 where d=5;/*所有d=5的行，d改成100*/

update t set d=5 where id=0; /*(0,0,5)*/
update t set c=5 where id=0; /*(0,5,5)*/
```
可以看到，按照日志顺序执行，id=0这一行的最终结果也是(0,5,5)。所以，id=0这一行的问题解决了。

但对于id=1这一行，在数据库里的结果是(1,5,5)，而跟进binlog的执行结果是(1,5,100)，也就是说幻读的问题还是没有解决。

也就是说，即使把所有的记录都加上锁，还是阻止不了新插入的记录。

## 如何解决幻读
### 间隙锁(Gap Lock)
产生幻读的原因是，行锁只能锁住行，但是新插入记录这个动作，要更新的是记录之间的“间隙”。因此，为了解决幻读问题，InnoDB只好引入新的锁，也就是间隙锁(Gap Lock)。

顾名思义，间隙锁，锁的就是两个值之间的间隙。

比如表t，初始化插入了6个记录，这就产生了7个间隙。

0          5        10           15           20              25

(-∞,0) | (0,5) | (5,10) | (10,15) | (15,20) |  (220,25)  |  (25,+∞)

这样，当执行 select \* from t where d=5 for update的时候，就不止是给数据库中已有的6个记录加上行锁，还同时加了7个间隙锁。这样就确保了无法再插入新的记录。

也就是说这时候，在一行行扫描的过程中，不仅给行加上了行锁，还给行两边的空隙，也加上了间隙锁。

数据行是可以加上锁的实体，数据行之间的间隙，也是可以加上锁的实体。

跟间隙锁存在冲突关系的，是“往这个间隙中插入一个记录”这个操作。间隙锁之间都不存在冲突关系。

* 例子：

|Session A|Session B|
| ----- | ----- |
|begin;<br>select \* from t where c=7 lock in share mode;| |
| |begin;<br>select \* form t where c=7 for update;|

这里Session B 并不会被堵住。因为表里并没有c=7这个记录，因此Session A加的是间隙锁(5,10)。而Session B也是在这个间隙加的间隙锁。它们有共同的目标，即：保护这个间隙，不允许插入值。但它们之间是不冲突的。

### next-key lock
间隙锁和行锁合称next-key lock，每个next-key lock是前开后闭区间。也就是说，我们的表t初始化以后，如果用select \* from t for update要把整个表所有记录锁起来，就形成了7个next-key lock，分别是 (-∞,0\]、(0,5\]、(5,10\]、(10,15\]、(15,20\]、(20, 25\]、(25, +supremum\]。

* supermum是啥？

因为+∞是开区间。实现上，InnoDB给每个索引加了一个不存在的最大值 supermum，这样才符合我们前面说的“都是前开后闭区间”。

## 总结
### 幻读是什么
幻读是指一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有看到的行。

幻读在“当前读”下才会出现。

幻读仅专指“新插入的行”。

### 幻读有什么问题
* 语义问题
* 数据一致性问题

### 怎么解决幻读
* 间隙锁(Gap lock)
* next-key lock

## 参考
[极客时间文章](https://time.geekbang.org/column/article/75173)

