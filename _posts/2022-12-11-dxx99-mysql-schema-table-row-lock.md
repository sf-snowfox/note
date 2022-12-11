[Toc]

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

### 临时表

### 复制表

## 行

## 列

## 锁

## 思考？
### 如何修改一个数据库名？
### mysql的视图与表的异同，能否重名，视图是否有对应的实体结构，对应的性能如何？


