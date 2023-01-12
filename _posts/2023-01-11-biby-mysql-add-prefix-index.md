# 怎么给字符串字段加索引？
## 给字符串加索引
* email字段用来存储用户邮箱，可做登陆验证，如何给其加索引
  * 整个邮箱字符串加索引
  * 邮箱字符串的前n个字节加索引

### 前缀索引
给字符串前n个字节加索引的方式为前缀索引。

MySQL支持前缀索引，可以定义字符串的一部分作为索引。如果创建索引的语句不指定前缀长度，索引就会包含整个字符串。

#### 全字符串索引与前缀索引执行流程区别
* index1索引，包含每个记录的整个字符串

```bash
alter table SUser add index index1(email);
```
* index2索引，每个记录都是只取前6个字节

```bash
alter table SUser add index index2(email(6))
```
email(6)这个索引结构中每个邮箱只取前6个字节，占用的空间会比较小，这是使用前缀索引的优势。但这可能会增加额外的记录扫描次数。
##### 如下语句在两个索引定义下分别是怎么执行的
```bash
select id,name,email from Suser where email=“zhangbibin@gmail.com”
```
* 使用index1(email整个字符串的索引结构)

1. 从index1索引树找到满足索引值是”[zhangbibin@gmail.com](mailto:zhangbibin@gmail.com)”的这条记录，取主键ID的值。
2. 到主键上查到主键值是ID的行，判断email的值是正确的，将这行记录加入结果集。
3. 取index1索引树上刚刚查到的位置的下一条记录，发现不满足email=“[zhangbibin@gmail.com](mailto:zhangbibin@gmail.com)”的条件，循环结束。

这个过程中，只需要回主键索引取一次数据，所以系统认为只扫描了一行。

* 使用index2(email(6)索引结构)

1. 从index2索引树找到满足索引值是“zhangb”的记录，找到第一个ID1。
2. 到主键上查到主键值是ID1的行，判断出email的值不是”[zhangbibin@gmail.com](mailto:zhangbibin@gmail.com)”，这行记录丢弃。
3. 取index2上刚刚查到的位置的下一条，发现仍然是"zhangb"，取出ID2，再到ID索引上取行然后判断，这次值对了，将这行记录加入结果集。
4. 重复上一步，直到在index2上取到的值不是“zhangb”时，循环结束。

在这个过程汇总，要回主键索引取4次数据，也就是扫描了了4行。(这里假设满足条件的索引是4行，即有四个索引是"zhangb")

* 区别与结论

通过对比可知，使用前缀索引后，可能会导致查询语句读数据的次数变多。

但是对于这个查询语句来说，如果定义的index2不是email(6)而是email(7),那么满足“zhangbi”的记录只有一个，也能够直接查到ID2，只扫描一行就结束了。

因此，使用前缀索引，定义好长度，可以做到既节省空间，又不用额外增加太多的查询成本。

#### 创建前缀索引时，如何确定应该使用多长的前缀
我们在建立索引时关注的是区分度，区分度越高越好。区分度越高，重复的键值越少。

我们可以通过统计索引上有多少个不同的值来判断要使用多长的前缀。

* 计算前缀索引长度

算出列上有多少个不同的值

```bash
select count(distinct email) as L from Suser;
```
接着选取不同长度的前缀来看这个值，比如看4-7个字节的前缀索引

```bash
select 
count(distinct left(email,4)) as L4,
count(distinct left(email,5)) as L5,
count(distinct left(email,6)) as L6,
count(distinct left(email,7)) as L7
from Suser;
```
使用前缀索引会损失区分度，需要预先设定一个可以接受的损失比例，比如5%。然后在L4-L7中，找出不小于L\*95%的值。可以选择满足条件的最小值来做为前缀索引的长度。

* 实例

![image](images/wY7Eu4znKjWU3SlMQhvO-UzsdL5gZ8OfyYslLof6czY.png)

![image](images/WELzoRwWJwvHLhZAb4UR8r4TK6Gxuw60roSfzzALgy8.png)

mobile列有33个不同的值，使用4、5、6、7前缀索引列上不同的值(区分度)分别是26，26，27，28

假设可以接受5%的损失，即可接受的值是 L\**0.95=33\*0.*95=24.7

那么满足条件的最小值就是长度为4的前缀索引

#### 前缀索引对覆盖索引的影响
1. 前缀索引可能会增加扫描行数，会影响到性能
2. 使用前缀索引就用不上覆盖索引对性能的优化

* eg:

```bash
select id.email from SUser where email=“zhangbibin@gmail.com”;
```
如果使用index1（email整个字符串的索引结构），可以利用覆盖索引，从index1查到结果后直接就返回了，不需要回到ID索引再去查一次。

如果用index2（email(6)）索引结构，不得不回到ID索引再去判断email字段的值。

即使将index2修改为包含所有信息的前缀索引，InnoDB还是要回到id索引再查一下，因为系统并不确定前缀索引的定义是否截断了完整信息。

#### 其他提高区分度的方式
##### 使用倒序存储
存储身份证号的时候倒过来存，每次查询可以这么写

```bash
select filed_list from t where id_card = reverse(‘input_id_card_string’);
```
##### 使用hash字段
可以在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引

```bash
aleter table t add id_card_crc int unsigned, add index(id_card_crc);
```
每次插入新数据时，同时用crc32()这个函数得到校验码填到这个新字段。

校验码可能存在冲突（哈希冲突），因此查询语句where部分要判断 id\_card 的值是否精确相同。

```bash
select filed_list from t where id_card_crc=crc32(‘input_id_card_string’) and id_card=‘input_id_card_string’
```
这样索引长度变成了4个字节。

##### 使用倒序存储和使用hash字段这两种方法的异同点
* 相同点

都不支持范围查询，只支持等值查询

* 区别

1. 占用额外空间角度。倒序存储在主键索引上，不会消耗额外的存储空间，hash字段方法需要增加一个字段。
2.  CPU消耗角度。倒序方式每次写和读，都需要额外调用一次 reverse 函数，hash字段方式需要额外调用一次 crc32 函数。从函数计算复杂度来看，reverse 函数额外消耗的CPU资源小一些。
3. 查询效率角度。使用hash字段方式的查询性能相对稳定一些。hash冲突的概率很低，可以认为每次查询平均的扫描行数接近1。倒序索引存储的区分度因数据类型而已，区分度一般没有hash方式高，扫描行数可以认为比hash方式多。

## 涉及重点概念
* 前缀索引

定义字段的前n个字节来构建索引

* 好处（优势）

占用空间小

* 缺点
  * 损失区分度
  * 不能利用覆盖索引，前缀索引符合条件后还需要(回表)回主键索引去判断，区分度不够会导致多次回表

## 总结
给字符串加索引，可以使用全字节索引或前缀索引。

前缀索引可能因为区分度不够，导致多次回表，增加查询扫描次数。

选择合适的前缀索引长度可以既省空间又不用多次回表(提高区分度)。

前缀索引无法利用覆盖索引对查询的优化。

倒序存储与hash方法可以提高索引的区分度，同时减少存储空间。

## 解决问题
给字符串加索引，怎么在不损失太多查询性能的情况下节约存储空间。\[节约存储空间在相同的数据页可以存放更多的索引值，降低磁盘IO\]