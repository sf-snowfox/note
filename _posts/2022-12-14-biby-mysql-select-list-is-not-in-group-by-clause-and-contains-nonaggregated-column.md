# \*\_\*  MySQL GROUP BY SELECT 未聚合列

## 报错示例
* 需求场景
  
  搜索统计表serach_record,结构如下,要按word聚合，并按word出现的次数排序。
  
  由于word可能出现在不同的搜索类型中（比如type，1：课程，2：商品），需要随机返回一个type。
  
  这时候如果按word跟type做group by条件，则会出现多条数据。因此需要group by word并且查询word跟type。

```bash
CREATE TABLE `serach_record` (
  `id` int unsigned NOT NULL AUTO_INCREMENT COMMENT '自增ID',
  `user_id` int unsigned NOT NULL COMMENT '用户ID',
  `type` tinyint NOT NULL COMMENT '搜索类型',
  `word` varchar(50) NOT NULL COMMENT '搜索关键字',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted_at` timestamp NULL DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_user_id` (`user_id`),
  KEY `idx_word` (`word`)
) ENGINE=InnoDB AUTO_INCREMENT=459 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='搜索记录';
```

* 数据库版本
```bash
Server version: 8.0.29 MySQL Community Server - GPL
```

* 根据需求,执行如下sql
```bash
select count(word) as total, word, type from `serach_record` where `user_id` = 7 group by `word` order by `total` desc limit 10
```
执行后得到如下错误
```bash
Syntax error or access violation: 1055 Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'education.serach_record.type' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by (SQL: select count(word) as total, word, type from `serach_record` where `user_id` = 7 group by `word` order by `total` desc limit 10)
```

* 解决方式
```bash
SET GLOBAL sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```

## 原因
SQL标准中不允许SELECT列表、HAVING条件语句、ORDER BY 语句中出现 GROUP BY 未列出的可聚合列。

MySQL中的 ONLY_FULL_GROUP_BY 标识用来判断是否遵循这一标准，默认为开启状态。

* 如下示例不被允许
```bash
SELECT a,b FROM table GROUP BY a
```
需要修改如下，但就满足不了前面的需求场景
```bash
SELECT a FROM table GROUP BY a
SELECT a,b FROM table GROUP BY a,b
```

## 解决方案

### 关闭 ONLY_FULL_GROUP_BY

#### 通过设置 sql_mode 关闭
* 查看 sql_mode 值
```bash
mysql> SELECT @@sql_mode;
+----------------------------------------------------------------------------------------------------+
| @@sql_mode                                                                                         |
+----------------------------------------------------------------------------------------------------+
| ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION |
+----------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)
```

* 关闭设置

设置后再次查询@@sql_mode,返回中没有 ONLY_FULL_GROUP_BY 模式
```bash
//当前session
SET SESSION sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY,',''));
//全局
SET GLOBAL sql_mode=(SELECT REPLACE(@@sql_mode,'ONLY_FULL_GROUP_BY',''));
```

#### 修改配置文件
...待补全

### ANY_VALUE()
...待补全

### 添加列间依赖
...待补全

## 参考
[stackoverflow](https://stackoverflow.com/questions/41887460/select-list-is-not-in-group-by-clause-and-contains-nonaggregated-column-inc)

[mysql_group_by_issue](https://www.cnblogs.com/Wayou/p/mysql_group_by_issue.html)