# mysql-schema-table-row-lock(todo)

## 库
### 命名规则
- 可以由字母、数字、下划线、＠、＃、＄
- 区分大小写，必须唯一
- 不能使用关键字，如create、select
- 不能单独数字，最长128位

### 基本操作
#### 创建数据库
```sql
create database db_name default charset utf8;
``` 
#### 删除数据库
```sql
drop database db_name;
```
#### 选择数据库
```sql
use db_name;
```

## 表

### 基本操作
#### 创建表
```sql
CREATE TABLE `t9` (
  `id` int unsigned NOT NULL AUTO_INCREMENT COMMENT '主键',
  `name` varchar(100) NOT NULL DEFAULT '' COMMENT '姓名',
  PRIMARY KEY (`id`),
  KEY `index_name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
#### 删表
```sql
drop table t9;
```
- drop, truncate, delete的区别
  - drop 所有空间全部释放，包括表结构所占空间
  - truncate 清空数据，删除的数据不会记录log, 不能恢复
  - delete 删除指定条件的数据，会有事务日志记录

#### 修改表
- 添加字段
```sql
ALTER TABLE `t9` ADD COLUMN `create_time` TIMESTAMP not null DEFAULT CURRENT_TIMESTAMP AFTER `name`;
```
- 修改字段类型
```sql
ALTER TABLE `t9` MODIFY COLUMN `name` VARCHAR(200);
```
- 修改默认值
```sql
alter table `t9` alter column `name` set default "?";
```
- 删除默认值
```sql
alter table `t9` alter column `name` drop default ;
```
- 删除索引
```sql
alter table `t9` drop INDEX `index_name`;
```
```sql
drop index `index_name` on `t9`;
```
- 添加索引
```sql
alter table `t9` ADD INDEX `index_name`(`name`);
```

### 临时表 & 内存表
#### 特点
- 内存表，数据保存在内存中，表结构保存在硬盘上，重启数据被清空
- 临时表只在当前链接可见，链接关闭会自动释放所有空间
- 临时表可以与普通表同名，临时表优先级高于普通表，同名后增删改查访问的是临时表数据
  - 普通表的table_def_key是有"库名+表名", 会冲突
  - 临时表是由"库名+表名", 在 加上了"server_id + thread_id"
  - 每个线程会维护自己的临时表链表，遍历链表，先检查临时表，再检查普通表

#### 临时表应用
- 分库分表的中间层，proxy可以用临时表
- 用于查询的排序 `using temporary`

#### 操作
##### 创建临时表
```sql
CREATE TEMPORARY TABLE `t9` (
	id int not null auto_increment,
	name varchar(100) not null DEFAULT "",
	age int not null DEFAULT 0,
	PRIMARY key (id)
) charset = utf8;
```
```sql
create temporary table `t9_tmp` (
    select * from `t9` limit 0, 1000
);
```

### 复制表
#### 操作
##### 如何将一个表的数据复制到另外一个表
```sql
insert into target_table_name select * from source_table_name;
```
```sql
create table new_table select  * from  old_table; # 创建表复制数据
create table new_table select  * from  old_table where 0; # 创建表不复制数据
```

## 锁
### 全局锁
- `flush tables with read lock`让整个数据库处于只读状态
- 使用场景，全库备份，会阻塞系统的写操作
- 客户端断开之后会自动释放锁
#### 备份其他方式
- 使用mysqldump使用参数 `-single-transaction` 会开启事务，保证数据一致性，不阻塞数据的更新
- 需要引擎支撑mvcc, 可重复读隔离级别

### 表级锁
#### 表锁
- `lock tables table_name read/write`
  - write是排他锁，其他线程不能读不能写，本线程能读写
  - read是共享锁，其他线程只能读不能写，本线程也不能写
- `unlock tables` 主动释放，或者客户端断开

#### MDL锁(metadata lock)
- 访问表的时候自动加上，保证在访问的时候，读写的正确性，防止数据结构被修改
- 增删改查的时候，加MDL读锁
- 修改表结构时，加MDL写锁

### 行锁
- 由存储引擎自己实现，并不是所有的引擎都支持

#### 二阶段锁(InnoDB)
- 行锁在需要的时候才加上，并不是不需要了就释放，而是等到事务结束时才释放
- 事务由多个行锁，应将最有可能影响并发度的锁尽量往后放

#### 死锁
- 多个事务循环依赖资源，都在等待其他线程释放资源，导致几个线程都进入无线等待状态，称为死锁
- 死锁检测
  1. 等待，直到超时， `innodb_lock_wait_timeout` 默认 50s
  2. 主动检测，发现死锁后，主动回滚死锁链条的某个事务，让其他事务继续执行， `innodb_deadlock_detect` 设置on
     1. 当一个事务被锁，看它所依赖的线程有没有被人锁住，如此循环，最后判断是否出现循环等待
     2. 需要消耗大量的cpu资源

## 思考？
### 1. 如何修改一个数据库名？
### 2. mysql的视图与表的异同，能否重名，视图是否有对应的实体结构，对应的性能如何？
### 3. 临时表是针对线程操作，在主从复制系统中，是否会创建给从库，主库线程退出回收临时表，从库如何处理?
- 备库的应用日志线程是共用的，多个线程执行，会被分配到从库的同一个worker中执行
- 多个线程同步到一个共用线程，是否会有命名冲突？不会，因为table_def_key的定义"库名+表名+M的serverId+M的sessionId"
### 4. 为什么删了表，表文件的大小还是没变？
- 数据删除以后，在innodb某个页会被标记为可复用
- delete命名把整个表数据删除，所有页都被标记已复用，但磁盘上文件大小不会变小
- 经过大量的增删改的表，都是可能存在空洞，如果把空洞的数据去掉，就能达到收缩表空间的目的
- 重建表，就可以达到这样的目的 `alter table A engine=InnoDB`



  

