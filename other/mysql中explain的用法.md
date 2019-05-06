（未总结）
## 使用
直接使用explain加上要分析的sql语句就可以了，比如：

```
explain select * from t_user where userId = 1001;
```
执行完之后，结果就会以表的形式展现出来。

## 结果列说明

### id
按照查询由外而内的顺序排序，最外层的为1，最内层的子查询依次加1。
### select_type
select类型，有以下几种：

- simple，简单查询
- primary，最外层的select
- derived，中间表，比如from子句的子查询
- union，union后面的select语句
- dependent union，union后面的select，但是当前union是在一个子查询中，取决于外面的查询
- union result，union的结果
- subquery，子查询中的第一个select
- dependent subquery，子查询中的第一个select，取决于外面的查询
### table
当前行的数据是由哪个表产生的，derived1，derived2...表示的是衍生表，中间表。
### type
表示使用哪种类别的连接，以及使用索引的情况。结果值从好到坏依次是：
`system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery > index_subquery > range > index > ALL`

- const，表最多只有一个匹配行，在查询开始时被读取。因为仅有一行，可认为是常数，只读取一次，const很快。
- system，const连接类型的一个特例，表仅有一行满足条件。
- ref，对于每个来自于前面的表的行组合，所有有匹配索引值的行将从这张表中读取，如果联接只使用键的最左边的前缀，或如果键不是UNIQUE或PRIMARY KEY（换句话说，如果联接不能基于关键字选择单个行的话），则使用ref。如果使用的键仅仅匹配少量行，该联接类型是不错的。ref可以用于使用=或<=>操作符的带索引的列。
- eq_ref，对于每个来自于前面的表的行组合，从该表中读取一行。这可能是最好的联接类型，除了const类型。它用在一个索引的所有部分被联接使用并且索引是UNIQUE或PRIMARY KEY。
- ref_or_null，该联接类型如同ref，但是添加了MySQL可以专门搜索包含NULL值的行。在解决子查询中经常使用该联接类型的优化。
- index_merge，该联接类型表示使用了索引合并优化方法。在这种情况下，key列包含了使用的索引的清单，key_len包含了使用的索引的最长的关键元素。eq_ref可以用于使用= 操作符比较的带索引的列。比较值可以为常量或一个使用在该表前面所读取的表的列的表达式。
- unique_subquery
- index_subquery
- range，只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引。key_len包含所使用索引的最长关键元素。在该类型中ref列为NULL。当使用=、<>、>、>=、<、<=、IS NULL、<=>、BETWEEN或者IN操作符，用常量比较关键字列时，可以使用range
- index，该联接类型与ALL相同，除了只有索引树被扫描。这通常比ALL快，因为索引文件通常比数据文件小。当查询只使用作为单索引一部分的列时，MySQL可以使用该联接类型。
- all，对于每个来自于先前的表的行组合，进行完整的表扫描。如果表是第一个没标记const的表，这通常不好，并且通常在它情况下很差。通常可以增加更多的索引而不要使用ALL，使得行能基于前面的表中的常数值或列值被检索出。
### possible_keys
MySQL能使用哪个索引在该表中找到行，该列完全独立于EXPLAIN输出所示的表的次序。这意味着在possible_keys中的某些键实际上不能按生成的表次序使用。
### key
显示MySQL实际决定使用的键（索引）。如果没有选择索引，键是NULL。要想强制MySQL使用或忽视possible_keys列中的索引，在查询中使用FORCE INDEX、USE INDEX或者IGNORE INDEX。
### key_len
显示MySQL决定使用的键长度。如果键是NULL，则长度为NULL。
使用的索引的长度。在不损失精确性的情况下，长度越短越好 
### ref
显示使用哪个列或常数与key一起从表中选择行。
### rows
显示MySQL认为它执行查询时必须检查的行数。
### Extra

该列包含MySQL解决查询的详细信息。

#### Distinct 
一旦MYSQL找到了与行相联合匹配的行，就不再搜索了 

#### Not exists 
MYSQL优化了LEFT JOIN，一旦它找到了匹配LEFT JOIN标准的行，就不再搜索了 

#### Range checked for each Record（index map:#） 
没有找到理想的索引，因此对于从前面表中来的每一个行组合，MYSQL检查使用哪个索引，并用它来从表中返回行。这是使用索引的最慢的连接之一 

#### Using filesort 
看到这个的时候，查询就需要优化了。MYSQL需要进行额外的步骤来发现如何对返回的行排序。它根据连接类型以及存储排序键值和匹配条件的全部行的行指针来排序全部行 

#### Using index 
列数据是从仅仅使用了索引中的信息而没有读取实际的行动的表返回的，这发生在对表的全部的请求列都是同一个索引的部分的时候 

#### Using temporary 
看到这个的时候，查询需要优化了。这里，MYSQL需要创建一个临时表来存储结果，这通常发生在对不同的列集进行ORDER BY上，而不是GROUP BY上 

#### Using where
使用了WHERE从句来限制哪些行将与下一张表匹配或者是返回给用户。如果不想返回表中的全部行，并且连接类型ALL或index，这就会发生，或者是查询有问题。