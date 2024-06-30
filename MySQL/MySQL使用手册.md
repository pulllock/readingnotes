# 数据库基本操作命令

- 登录数据库： `mysql -h 主机IP -u 用户名 -p 密码`
- 查看所有数据库： `show databases;`
- 创建数据库： `create database 数据库名;`
- 删除数据库： `drop database 数据库名;`
- 选择数据库： `use 数据库名;`
- 查看支持的存储引擎： `show engines[\G];`
- 查看数据库的默认编码： `show variables like 'character_set_database';`
- 查看警告信息： `show warnings;`
- 查看线程： `show processlist;`、 `show full processlist;`

# 表的基本操作命令

- 创建表：

```sql
create table 表名
(
    字段名1 数据类型 [列级别约束条件] [默认值],
    字段名2 数据类型 [列级别约束条件] [默认值],
    ...,
    [表级别约束条件]
);
```

- 查看表基本结构： `describe 表名;` 或者： `desc 表名;`

- 查看表基本结构： `show create table 表名[\G];`

- 修改表名： `alter table 旧表名 rename [to] 新表名;`

- 修改字段类型： `alter table 表名 modify 字段名 数据类型;`

- 修改字段名： `alter table 表名 change 旧字段名  新字段名 新数据类型;`

- 添加字段： `alter table 表名 add 新字段名 数据类型 [约束条件] [first|after 已存在字段名];`

- 删除字段： `alter table 表名 drop 字段名;`

- 修改字段排列位置： `alter table 表名 modify 字段1 数据类型 first|after 字段2;`

- 更改表的存储引擎： `alter table 表名 engine=新存储引擎名;`

- 删除表的外键约束： `alter table 表名 drop foreign key 外键约束名;`

- 删除没有被关联的表： `drop table [if exists] 表1, 表2, ..., 表n;`

- 删除被其他表关联的主表： 需要先解除关联子表的外键约束，再删除主表

- 查看表的默认编码，可使用查看表基本结构的命令： `show create table 表名[\G];`

- 创建表的时候创建索引： `create table 表名 [字段名 数据类型] [unique | fulltext | spatial] [index | key] [索引名] (列名 [索引长度])； `

- 在已经存在的表上创建索引：`alter table 表名 add [unique | fulltext | spatial] [index | key] [索引名] (列名 [索引长度], ...) [asc | desc]；`

- 查看指定表中创建的索引： `show index from 表名\G;`，结果中字段如下：
  
  - Table表示创建索引的表
  
  - Non_unique表示索引非唯一，1是非唯一索引，0是唯一索引
  
  - Key_name索引名称
  
  - Seq_in_index该字段在索引中的位置
  
  - Column_name定义索引的列字段
  
  - Sub_part索引长度
  
  - Null表示该字段是否能为空值
  
  - Index_type表示索引类型

- `create index`创建索引： `create [unique | fulltext | spatial] index 索引名 on 表名 (列名 [索引长度], ...) [asc | desc]；`

- 删除索引： `alter table 表名 drop index 索引名；`

- 删除索引： `drop index 索引名 on 表名;`

# 查询语句

select查询语句基本格式：

```sql
select
[distinct] <* | 列或列表达式> [,<列或列表达式>] ...
from <表名或视图名> [,<表名或视图名>...] | (<select 语句>) [as] <别名>
[where <条件表达式>]
[group by <列名> 【 [having <条件表达式>]]
[order by <列名> [, 列名] [asc | desc]]
[limit <offset>, <row count>]
;
```

- 连接查询
  
  - 内连接查询： `inner join`
  
  - 外连接查询：
    
    - 左（外）连接： `left [outer] join`
    
    - 右（外）连接： `right [outer] join`

- 子查询
  
  - any
  
  - some
  
  - all
  
  - in
  
  - not in
  
  - exists
  
  - not exists

- 合并查询结果：`union [all]`，基本格式如下：

```sql
select 列, ... from 表1
union [all]
select 列, ... from 表2
```

- 别名
  
  - 表： `表名 [as] 表别名`
  
  - 字段： `列名 [as] 列别名`

# 数据类型

## 整数类型

| 类型           | 说明      | 存储大小 |
| ------------ | ------- | ---- |
| tinyint      | 很小的整数   | 1字节  |
| smallint     | 小的整数    | 2字节  |
| mediumint    | 中等大小的整数 | 3字节  |
| int(integer) | 普通大小的整数 | 4字节  |
| bigint       | 大整数     | 8字节  |

## 浮点数类型和定点数类型

| 类型                | 说明     | 存储大小  |
| ----------------- | ------ | ----- |
| float             | 单精度浮点数 | 4字节   |
| double            | 双精度浮点数 | 8字节   |
| decimal(M,D), dec | 定点数    | M+2字节 |

## 日期与时间类型

| 类型        | 格式                  | 存储大小 |
| --------- | ------------------- | ---- |
| year      | YYYY                | 1字节  |
| time      | HH:MM:SS            | 3字节  |
| date      | YYYY-MM-DD          | 3字节  |
| datetime  | YYYY-MM-DD HH:MM:SS | 8字节  |
| timestamp | YYYY-MM-DD HH:MM:SS | 4字节  |

## 文本字符串类型

| 类型         | 说明       | 存储大小                    |
| ---------- | -------- | ----------------------- |
| char(M)    | 固定长度字符串  | M字节，M范围：[1, 255]        |
| varchar(M) | 变长字符串    | L+1字节，L<=M，M范围：[1, 255] |
| tinytext   | 非常小的字符串  | L+1字节，L<2^8             |
| text       | 小的字符串    | L+2字节，L<2^16            |
| mediumtext | 中等大小的字符串 | L+3字节，L<2^24            |
| longtext   | 大的字符串    | L+4字节，L<2^32            |
| enum       | 枚举类型     | 1或2字节                   |
| set        | 集合       | 1、2、3、4或8字节             |

## 二进制字符串类型

| 类型            | 说明         | 存储大小         |
| ------------- | ---------- | ------------ |
| bit(M)        | 位字段类型      | 大约(M+7)/8字节  |
| binary(M)     | 固定长度二进制字符串 | M字节          |
| varbinary(M)  | 可变长度二进制字符串 | M+1字节        |
| tinyblob(M)   | 非常小的BLOB   | L+1字节，L<2^8  |
| blob(M)       | 小BLOB      | L+2字节，L<2^16 |
| mediumbolb(M) | 中等大小的BLOB  | L+3字节，L<2^24 |
| longblob(M)   | 非常大的BLOB   | L+4字节，L<2^32 |

# 运算符

- 算术运算符：
  - 加（`+`）
  - 减（`-`）
  - 乘（`*`）
  - 除（`/`）
  - 求余（模运算`%`）
- 比较运算符：
  - 大于（`>`）
  - 小于（`<`）
  - 等于（`=`）
  - 安全等于（`<=>`）
  - 大于等于（`>=`）
  - 小于等于（`<=`）
  - 不等于（`!=`或`<>`）
  - in
  - not in
  - between and
  - is null或isnull
  - is not null
  - greatest
  - least
  - like：
    - `%`：匹配任何数目字符
    - `_`：匹配一个字符
  - regexp，正则：
    - `^`：以字符开头
    - `$`：以字符结尾
    - `.`：匹配任何一个单字符
    - `[...]`：匹配在方括号内的任何字符
    - `*`：匹配0或多个在它前面的字符
- 逻辑运算符：
  - 逻辑非（not或`!`）
  - 逻辑与（and或`&&`）
  - 逻辑或（or或`||`）
  - 逻辑异或（xor）
- 位运算符：
  - 位与（`&`）
  - 位或（`|`）
  - 取反（`~`）
  - 位异或（`^`）
  - 左移（`<<`）
  - 右移（`>>`）

# 函数

## 数学函数

- 绝对值： `abs()`

- 圆周率： `pi()`

- 平方根： `sqrt(x)`

- 求余： `mod(x, y)`

- 返回不小于x的最小整数： `ceil(x)` 、 `ceiling(x)`

- 返回不大于x的最大整数： `floor(x)`

- 随机数： `rand()` 、 `rand(x)`

- 返回最接近x的整数，对x进行四舍五入： `round(x)`

- 返回最接近x的整数，其值保留到小数点后y位： `round(x, y)`

- 截取，舍去至小数点后y位的数字x： `truncate(x, y)`

- 返回参数的符号： `sign(x)`

- 幂运算，乘方： `pow(x, y)` 、 `power(x, y)`

- 幂运算，e的x乘方： `exp(x)`

- 对数运算： `log(x)` 、 `log10(x)`

- 角度转化为弧度： `radians(x)`

- 弧度转化为角度： `degrees(x)`

- 正弦： `sin(x)`

- 反正弦： `asin(x)`

- 余弦： `cos(x)`

- 反余弦： `acos(x)`

- 正切： `tan(x)`

- 反正切： `atan(x)`

- 余切： `cot(x)`

## 字符串函数

- 字符个数： `char_length(str)`

- 字符串字节长度： `length(str)`

- 合并字符串： `concat(s1, s2, ...)` ，如果有一个参数为null则返回值为null

- 合并字符串，带有分隔符： `concat_ws(分隔符, s1, s2, ...)` ， concat_ws表示concat with separator

- 替换字符串： `insert(s1, start, len, s2)`

- 转小写： `lower(str)`  、 `lcase(str)`

- 转大写： `upper(str)` 、 `ucase(str)`

- 获取指定长度字符串，返回字符串s的最左边n个字符： `left(s, n)`

- 获取指定长度字符串，返回字符串s的最右边n个字符： `right(s, n)`

- 填充字符串： `lpad(s1, len, s2)` 返回字符串s1，其左边由字符串s2填充到len字符长度

- 填充字符串： `rpad(s1, len, s2)` 返回字符串s1，其右边由字符串s2填充到len字符长度

- 删除左侧空格： `ltrim(s)`

- 删除右侧空格： `rtrim(s)`

- 删除两侧空格： `trim(s)`

- 删除指定字符串： `trim(s1 from s)` 删除字符串s中两端所有的字符串s1

- 重复生成字符串： `repeat(s, n)` 返回一个由重复的字符串s组成的字符串，字符串s的数目等于n

- 返回由n个空格组成的字符串： `space(n)`

- 替换字符串： `replace(s, s1, s2)` 使用字符串s2替换字符串s中所有的字符串s1

- 比较字符串大小： `strcmp(s1, s2`

- 获取子串： `substring(s, n, len)` 、 `mid(s, n, len)`  从字符串s的n位置起，返回一个长度为len的子串

- 匹配字串开始位置： `locate(str1, str)` 、 `position(str1 in str)` 、 `instr(str, str1)` 返回子字符串str1在字符串str中的开始位置

- 字符串逆序： `reverse(s)`

- 返回指定位置的字符串： `elt(n, 字符串1, 字符串2, ...)` n表示要返回第几个字符串

- 返回指定字符串位置： `field(s, s1, s2, ..., sn)` 返回字符串s在列表s1...sn中第一次出现的位置

- 返回子串位置： `find_in_set(s1, s2)` 返回字符串s1在字符串列表s2中出现的位置，s2是由多个逗号分开的字符串

- 选取字符串： `make_set(x, s1, s2, ..., sn)` 按照x的二进制数从s1, s2, ..., sn中选取字符串

## 日期和时间函数

- 系统当前日期和时间： `now()` 、 `current_timestamp()`、 `localtime()` 、 `sysdate()`

- 系统当前日期： `current_date()` 、 `curdate()`

- 系统当前时间： `current_time()` 、 `curtime()`

- unix时间戳： `unix_timestamp(date)`

- unix时间戳转换为普通格式： `from_unixtime(date)`

- UTC日期： `utc_date()`

- UTC时间： `utc_time()`

- 获取月份： `month(date)`

- 获取月份的英文全名： `monthname(date)`

- 获取date对应的工作日英文名： `dayname(date)`

- 获取date对应的一周中的索引： `dayofweek(date)` 1表示周日、2表示周一、...、7表示周六

- 获取date对应工作日索引：` weekday(date)` 0表示周一、1表示周二、...、6表示周日

- 获取日期date是一年中的第几周： `week(date)`

- 计算某天位于一年中的第几周： `weekofyear(date)`

- 获取date是一年中的第几天： `dayofyear(date)`

- 获取date是一个月中的第几天： `dayofmonth(date)`

- 获取年份： `year(date)`

- 获取季度： `quarter(date)`

- 获取分钟： `minute(time)`

- 获取秒： `second(time)`

- 获取日期指定值： `extract(type from date)`
  
  - type取值： `year`、 `year_month` 、 `day_minute`

- 时间转为秒： `time_to_sec(time)`

- 秒转为时间： `sec_to_time(seconds)`

### 计算日期和时间的函数

- `date_add(date, interval expr type)`

- `adddate(date, interval expr type)`

- `date_sub(date, interval expr type)`

- `subdate(date, interval expr type)`

- `addtime(date, expr)`

- `subtime(date, expr)`

- `datediff(date1, date2)` 计算两个日期之间的间隔天数

上面函数的参数如下：

- `date`： 是一个datetime或date值，用来指定起始时间

- `expr`： 是一个表达式，用来指定从起始日期添加或减去的时间间隔值

- `type`： 是关键词，用来解释表达式expr，下面表格是type和expr的关系

| type取值               | expr格式                         |
| -------------------- | ------------------------------ |
| `microsecond`        | `microsenconds`                |
| `second`             | `seconds`                      |
| `minute`             | `minutes`                      |
| `hour`               | `hours`                        |
| `day`                | `days`                         |
| `week`               | `weeks`                        |
| `month`              | `months`                       |
| `quarter`            | `quarters`                     |
| `year`               | `years`                        |
| `second_microsecond` | `'seconds.microseconds'`       |
| `minute_microsecond` | `'minutes.microseconds'`       |
| `minute_second`      | `'minutes:seconds'`            |
| `hour_microsecond`   | `'hours.microseconds'`         |
| `hour_second`        | `'hours:minutes:seconds'`      |
| `hour_minute`        | `'hours:minutes'`              |
| `day_microsecond`    | `'days.microseconds'`          |
| `day_second`         | `'days hours:minutes:seconds'` |
| `day_minute`         | `'days hours:minutes'`         |
| `day_hour`           | `'days hours'`                 |
| `year_month`         | `'years-months'`               |

### 日期时间格式化函数

- `date_format(date, format)`

- `time_format(time, format)`

其中参数format格式如下表：

| format格式  | 说明                               |
| --------- | -------------------------------- |
| `%a`      | 工作日缩写（Sun...Sat）                 |
| `%b`      | 月份缩写（Jan...Dec）                  |
| `%c`      | 月份（0...12）                       |
| `%D`      | 英文后缀表示的月中的几号（1st, 2nd...）        |
| `%d`      | 该月日期（00...31）                    |
| `%e`      | 该月日期（0...31）                     |
| `%f`      | 微秒（000000...999999）              |
| `%H`      | 2位数表示的24小时（00...23）              |
| `%h`、`%I` | 2位数表示12小时（01...12）               |
| `%i`      | 分钟（00...59）                      |
| `%j`      | 一年中的天数（001...366）                |
| `%k`      | 以24小时表示时间（0...23）                |
| `%l`      | 以12小时表示时间（1...12）                |
| `%M`      | 月份名称（January...December）         |
| `%m`      | 月份（00...12）                      |
| `%p`      | 上午（AM）、下午（PM）                    |
| `%r`      | 时间，12小时制（`hh:mm:ss AM\|PM`）      |
| `%S`、`%s` | 2位数形式表示秒（00...59）                |
| `%T`      | 24小时制时间（`hh:mm:ss`）              |
| `%U`      | 周（00...53），周日为每周第一天              |
| `%u`      | 周（00...53），周一为每周第一天              |
| `%V`      | 周（01...53），周日为每周第一天，和`%X`同时使用    |
| `%v`      | 周（01...53），周一为每周第一天，和`%x`同时使用    |
| `%W`      | 工作日名称（周日...周六）                   |
| `%w`      | 一周中的每日（0=周日...6=周六）              |
| `%X`      | 该周的年份，周日为每周第一天；4位数数字形式；和`%V`同时使用 |
| `%x`      | 该周的年份，周一为每周第一天；4位数数字形式；和`%v`同时使用 |
| `%Y`      | 年份，4位数形式                         |
| `%y`      | 年份，2位数形式                         |
| `%%`      | 标识符`%`                           |

- `get_format(val_type, format_type)` 返回日期时间字符串的显示格式：
  
  - `val_type`表示日期数据类型：`date`、`datetime`、`time`
  
  - `format_type`表示格式化显示类型

`get_format(val_type, format_type)`参数对应关系和返回的格式字符串如下表：

| val_type | format_type | 显示格式字符串             |
| -------- | ----------- | ------------------- |
| date     | EUR         | `%d.%m.%Y`          |
| date     | INTERVAL    | `%Y%m%d`            |
| date     | ISO         | `%Y-%m-%d`          |
| date     | JIS         | `%Y-%m-%d`          |
| date     | USA         | `%m.%d.%Y`          |
| time     | EUR         | `%H.%i.%s`          |
| time     | INTERVAL    | `%H%i%s`            |
| time     | ISO         | `%H:%i:%s`          |
| time     | JIS         | `%H:%i:%s`          |
| time     | USA         | `%h:%i:%s %p`       |
| datetime | EUR         | `%Y-%m-%d %H.%i.%s` |
| datetime | INTERVAL    | `%Y%m%d%H%i%s`      |
| datetime | ISO         | `%Y-%m-%d %H:%i:%s` |
| datetime | JIS         | `%Y-%m-%d %H:%i:%s` |
| datetime | USA         | `%Y-%m-%d %H.%i.%s` |

## 条件判断函数（控制流程函数）

- `if(expr, v1, v2)`： 如果表达式expr为true（`expr <>0 and expr <> NULL`），则返回v1；否则返回v2

- `ifnull(v1, v2)`： 如果v1部位NULL则返回v1；否则返回v2

- `case expr when v1 then r1 [when v2 then r2] ... [else rn+1] end`： 如果expr值等于某个vn,则返回对应位置then后面的结果；如果所有值都不相等，则回返else后面的rn+1

## 系统信息函数

- 获取MySQL服务器版本号： `version()`

- 服务器当前连接的次数： `connection_id()`

- 获取数据库名： `database()`、 `schema()`

- 获取用户名： `user()` 、`current_user()`、 `system_user()`、 `session_user()`

- 获取字符串的字符集： `charset(str)`

- 返回字符排列方式： `collation(str)`

- 获取最后一个自增ID值： `last_insert_id()`

## 加密函数

- `MD5(str)` 为字符串计算出一个MD5 128比特校验和，32位十六进制数字的二进制字符串形式

- `SHA(str)`

- `SHA2(str, hash_length)` 使用hash_length作为长度加密str，hash_Length支持的值有224、256、384、512、0，其中0等同于256

## 其他函数

- 格式化： `format(x, n)` 将数字x格式化，并以四舍五入的方式保留小数点后n位

- 不同进制数间的转换： `conv(N, from_base, to_base)`，其中N为要转换的数字，从`from_base`进制转换为`to_base`进制

- 网络地址的点地址表示转整数数值形式： `inet_aton(expr)`

- 整数数值形式转网络地址的点地址形式： `inet_ntoa(expr)`

- 加锁： `get_lock(str, timeout)` 使用字符串str作为名字加锁，超时时间为timeout。成功加锁返回1，操作超时返回0，发生错误返回null

- 解锁： `release_lock(str)` 解开被`get_lock()`获取的锁。如果锁被解开则返回1，如果该线程未创建锁则返回0，如果锁不存在则返回null

- 查询锁是否可用： `is_free_lock(str)` 如果锁可以使用则返回1，如果锁正在被使用则返回0，如果出现错误则返回null

- 查询锁是否正在被使用： `is_used_lock(str)` 如果被使用则返回使用该锁的客户端的连接标识符（connection ID），否则返回null

- 重复执行指定操作： `benchmark(count, expr)` 重复执行表达式expr，执行count次

- 改变字符集： `convert(str using 字符集)`

- 改变数据类型： `cast(x, as type)` 、 `convert(x, type)`  将x转换为type类型，其中可转换的type有：binary、char(n)、date、time、datetime、decimal、signed、unsigned

- 数字转换为二进制： `bin()`

- `group_concat(列)` 将分组中各字段的值显示出来

## 集合函数

- 返回某列的最大值： `max()`

- 返回某列的最小值： `min()`

- 返回某列的行数： `count(*)`、 `count(字段名)`

- 返回某列值的和： `sum()`

- 返回某列的平均值： `avg()`

## 窗口函数

窗口函数

# explain语句

explain输出结果字段解释如下：

- id 表示编号，标识select所属行

- select_type表示所使用的select查询类型，取值有如下：
  
  - simple表示简单查询，不包括子查询和UNION
  
  - primary如果查询有任何复杂的子部分，则最外层部分标记为primary
  
  - union在union中的第二个和随后的select被标记为union
  
  - union result用来从union的匿名临时表检索结果的select
  
  - subquery包含在select列表中的子查询中的select
  
  - derived用来表示包含在from子句的子查询中的select

- table表示对应行正在访问哪个表

- type，取值有如下（这些取值从上到下是最优到最差：
  
  - system
  
  - const
  
  - eq_ref索引查找，最多只返回一条符合条件的记录
  
  - ref索引访问（索引查找）
  
  - range范围扫描
  
  - index跟全表扫描一样，只是扫描表时按照索引次序进行，而不是按照行进行扫描。主要优点是避免了排序。
  
  - ALL全表扫描

- possible_keys显示了查询可以使用哪些索引

- key显示了MySQL决定采用哪个索引来优化对该表的访问

- key_len显示了MySQL在索引里使用的字节数

- ref显示了之前的表在key列记录的索引中查找值所用的列或常量

- rows是估计为了找到所需行而要读取的行数

- filtered显示的是针对表里符合某个条件的记录数的百分比所做的一个悲观估算

- Extra显示额外信息，常见值如下：
  
  - Using index表示将使用覆盖索引，避免访问表
  
  - Using where表示将在存储引擎检索行后在进行过滤
  
  - Using temporary对查询结果排序时会使用一个临时表
  
  - Using filesort对结果使用一个外部索引排序，而不是按索引次序从表里读取行
  
  - Range checked for each record (index map: N)表示没有好用的索引，新的索引将在联接的每一行上重新估算。

# 存储过程和函数

## 创建存储过程

创建存储过程基本语法：

```sql
create procedure 存储过程名称
([存储过程参数列表])
[存储过程特性]
存储过程的SQL代码体
```

各部分说明如下：

- 存储过程参数列表，形式如下：`[IN | OUT | INOUT] 参数名 参数类型`
  
  - IN表示输入参数
  
  - OUT表示输出参数
  
  - INOUT表示既可以输入也可以输出
  
  - 参数类型可以是MySQL数据库中的任意类型

- 存储过程特性，取值如下：
  
  - LANGUAGE SQL说明存储过程代码体是由SQL语句组成的
  
  - [NOT] DETERMINISTIC指明存储过程执行的结果是否正确
    
    - DETERMINISTIC表示结果是确定的
    
    - NOT DETERMINISTIC表示结果是不确定的，该值是默认值
  
  - {CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA}指明子程序使用SQL语句的限制
    
    - CONTAINS SQL表名子程序包含SQL语句，但不包含读写数据的语句，该值是默认值
    
    - NO SQL表名子程序不包含SQL语句
    
    - READS SQL DATA说明子程序包含读数据的语句
    
    - MODIFIES SQL DATA表名子程序包含写数据的语句
  
  - SQL SECURITY {DEFINER | INVOKER}指明谁有权限来执行
    
    - DEFINER表示只有定义者才能执行，该值为默认值
    
    - INVOKER表示拥有权限的调用者可以执行
  
  - COMMENT 'string'注释信息

- 存储过程的SQL代码体，是SQL代码内容，可以用BEGIN...END来表示SQL代码的开始和结束

## 创建存储函数

创建存储函数基本格式如下：

```sql
create function 函数名
（[存储函数参数列表]）
returns 返回的数据类型
[存储函数特性]
存储函数
```

各部分说明如下：

- 存储函数参数列表，形式如下：`[IN | OUT | INOUT] 参数名 参数类型`
  
  - IN表示输入参数
  
  - OUT表示输出参数
  
  - INOUT表示既可以输入也可以输出
  
  - 参数类型可以是MySQL数据库中的任意类型

- 返回的数据类型表示函数返回数据的类型

- 存储函数特性，取值如下：
  
  - LANGUAGE SQL说明存储函数代码体是由SQL语句组成的
  
  - [NOT] DETERMINISTIC指明存储函数执行的结果是否正确
    
    - DETERMINISTIC表示结果是确定的
    
    - NOT DETERMINISTIC表示结果是不确定的，该值是默认值
  
  - {CONTAINS SQL | NO SQL | READS SQL DATA | MODIFIES SQL DATA}指明子程序使用SQL语句的限制
    
    - CONTAINS SQL表名子程序包含SQL语句，但不包含读写数据的语句，该值是默认值
    
    - NO SQL表名子程序不包含SQL语句
    
    - READS SQL DATA说明子程序包含读数据的语句
    
    - MODIFIES SQL DATA表名子程序包含写数据的语句
  
  - SQL SECURITY {DEFINER | INVOKER}指明谁有权限来执行
    
    - DEFINER表示只有定义者才能执行，该值为默认值
    
    - INVOKER表示拥有权限的调用者可以执行
  
  - COMMENT 'string'注释信息

- 存储过程的SQL代码体



- 定义变量： `declare 变量名 [, 变量名] ... 变量类型 [default 默认值];`

- 为变量赋值： `set 变量名 = 表达式 [, 变量名 = 表达式] ...;`

- 为变量赋值： `select 列名 [, ...] into 变量名 [, ...] 查询条件表达式`

- 定义条件： `DECLARE 条件名 CONDITION FOR SQLSTATE [VALUE] sqlstate_value`或 `DECLARE 条件名 CONDITION FOR mysql_error_code}`
  
  - sqlstate_value表示长度为5的字符串类型错误代码
  
  - mysql_error_codeb表示为数值类型的错误代码

- 定义处理程序： `DECLARE handler_type HANDLER FOR condition_value [,...] sp_statement`
  
  - handler_type为错误处理方式，取值如下：
    
    - CONTINUE表示遇到错误不处理
    
    - EXIT表示遇到错误马上突出
    
    - UNDO表示遇到错误后撤回之前的操作
  
  - condition_value表示错误类型，取值如下：
    
    - SQLSTATE [VALUE] sqlstate_value包含5个字符的字符串错误值
    
    - condition_name表示使用DECALRE CONDITION定义的错误条件名称
    
    - SQLWARNING匹配所有以01开头的SQLSTATE错误代码
    
    - NOT FOUND匹配所有以02开头的SQLSTATE错误代码
    
    - SQLEXCEPTION匹配所有其他的SQLSTATE错误代码
    
    - MySQL_error_code匹配数值类型错误代码
  
  - sp_statement是程序语句段，表示在遇到定义的错误时需要执行的存储过程或函数

- 调用存储过程： `call 存储过程名字([参数[, ...])`

- 调用存储函数：和MySQL内部函数使用方法一样

- 声明光标：`DECLARE 光标名称 CURSOR FOR select语句内容`

- 打开光标：`OPEN 光标名称`

- 使用光标： `FETCH 光标名称 INTO 变量名 [, 变量名] ...`

- 关闭光标： `CLOSE 光标名称`

- 查看存储过程和函数的状态： `SHOW {PROCEDURE | FUNCTION} STATUS [LIKE 'pattern'] [\G]` LIKE语句表示匹配存储过程或函数的名称

- 查看存储过程和函数的定义： `SHOW CREATE {PROCEDURE | FUNCTION} 存储过程或函数名称 [\G]`

- 查看存储过程和函数的定义，在information_schema数据库的routines表中

- 修改存储过程和函数：`ALTER {PROCEDURE | FUNCTION} 存储过程或函数名称 [存储函数的特性]`，其中存储函数特性可能的取值有：
  
  - CONSTAINS SQL表示子程序包含SQL语句但不包含读写数据的语句
  
  - NO SQL表示子程序中不包含SQL语句
  
  - READS SQL DATA表示子程序中包含读数据的语句
  
  - MODIFIES SQL DATA表示子程序中包含写数据的语句
  
  - SQL SECURITY {DEFINER | INVOKER}指明谁有全线来执行
    
    - DEFINER表示只有定义着自己能够执行
    
    - INVOKER表示调用者可执行
  
  - COMMENT 'string'注释信息

- 删除存储过程和函数： `DROP {PROCEDURE | FUNCTION} [IF EXISTS] 存储过程或函数名称`

## 流程控制

- IF语句：

```sql
IF 条件表达式 
    THEN 执行语句
    [ELSEIF 条件表达式 THEN 执行语句] ...
    [ELSE 执行语句]
END IF
```

- CASE语句：

```sql
CASE 条件表达式
    WHEN 表达式可能的值 THEN 执行语句
    [WHEN 表达式可能的值 THEN 执行语句] ...
    [ELSE 执行语句]
END CASE
```

```sql
CASE
    WHEN 条件表达式 THEN 执行语句
    [WHEN 条件表达式 THEN 执行语句] ...
    [ELSE 执行语句]
END CASE
```

- LOOP语句：

```sql
loop语句名称: LOOP
    需要循环执行的语句
END LOOP loop语句名称
```

- LEAVE语句： `LEAVE 循环标志` 用来退出被标注的流程控制

- ITERATE语句： `ITERATE 循环标志` 只能出现在LOOP、REPEAT、WHILE语句内，表示再次循环

- REPEAT语句：

```sql
repeat语句名称: REPEAT
    执行的语句
UNTIL 条件表达式
END REPEAT repeat语句名称
```

- WHILE语句：

```sql
while语句名称: WHILE 条件表达式 DO
    执行的语句
END WHILE while语句名称
```

# 全局变量

- 设置全局变量： `SET GLOBAL 全局变量名称=值`

- 设置持久化的全局变量： `SET PERSIST 全局变量名称=值`

# 视图

- 创建视图：

```sql
CREATE [OR REPLACE] [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
VIEW 视图名 [(属性列)]
AS select语句
[WITH [CASCADED | LOCAL] CHECK OPTION]
```

- 创建视图各参数说明如下：
  
  - ALGORITHM表示视图选择算法，有三个值：
    
    - UNDEFINED表示MySQL将自动选择算法
    
    - MERGE表示将使用的视图语句与视图定义合并起来，使得视图定义的某一部分取代语句相应的部分
    
    - TEMPTABLE表示将视图的结果存入临时表，用临时表来执行语句
  
  - `WITH [CASCADED | LOCAL] CHECK OPTION`表示视图在更新时保证在视图的权限范围之内
    
    - CASCADED是默认值，表示更新视图时要满足所有相关表和表的条件
    
    - LOCAL表示更新视图时满足该视图本身定义的条件即可

- 查看视图基本信息： `DESCRIBE 视图名;` 或 `SHOW TABLE STATUS LIKE '视图名';`

- 查看视图详细信息： `SHOW CREATE VIEW 视图名;`

- 查看视图详细信息，在information_schema数据库的views表中

- 修改视图：

```sql
CREATE OR REPLACE [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
VIEW 视图名 [(属性列)]
AS select语句
[WITH [CASCADED | LOCAL] CHECK OPTION]
```

- 修改视图：

```sql
ALTER [ALGORITHM = {UNDEFINED | MERGE | TEMPTABLE}]
VIEW 视图名 [(属性列)]
AS select语句
[WITH [CASCADED | LOCAL] CHECK OPTION]
```

- 删除视图： `DROP VIEW [IF EXISTS] 视图名 [, 视图名] ... [RESTRICT | CASCADE]`

# 触发器

- 创建触发器： `CREATE TRIGGER 触发器名称 触发时机 触发事件 ON 表名 FOR EACH ROW 触发器执行语句`
  
  - 触发时机有：before、after
  
  - 触发事件有：INSERT、UPDATE、DELETE

- 创建有多个执行语句的触发器：

```sql
CREATE TRIGGER 触发器名称 触发时机 触发事件 ON 表名 FOR EACH ROW 
BEGIN
    触发器执行语句列表
END
```

- 查看触发器： `SHOW TRIGGERS \G;`

- 查看触发器，在information_schema数据库的triggers表中

- 删除触发器： `DROP TRIGGER 触发器名称;`

# 权限和安全

- 权限表
  
  - user表记录允许连接到服务器的账号信息
  
  - db表存储了用户对某个数据库的操作权限，决定用户能从哪个主机存取哪个数据库
  
  - table_priv表用来对表设置操作权限
  
  - columns_priv表用来对表的某一列设置权限
  
  - procs_priv表可以对存储过程和存储函数设置操作权限

- 登录mysql： `mysql -h 主机名 -u 用户名 -p 密码 -P 端口号 数据库名 -e 要执行的SQL语句或命令`

- 新建普通用户： `CREATE USER '用户名'@'主机名' IDENTIFIED BY [PASSWORD] '密码'` 或 `CREATE USER '用户名'@'主机名' IDENTITIED WITH 插件名称 [AS '插件参数']`

- 删除普通用户： `DROP USER '用户名'@'主机名' [, '用户名'@'主机名', ...]`

- 修改root用户密码： `update mysql.user set authentication_string = MD5("密码") where user='root' and host='localhost';` 修改后执行`flush privileges`重新加载用户权限

- 修改普通用户密码： `SET PASSWORD FOR '用户名'@'主机名'='密码';`

- 授权的层级：
  
  - 全局层级，适用于服务器中的所有数据库。`GRANT ALL ON *.*`授予全局权限；`REVOKE ALL ON *.*`撤销全局权限
  
  - 数据库层级，适用于指定数据库。`GRANT ALL ON 数据库名.*`授予数据库权限；`REVOKE ALL ON 数据库名.*`撤销数据库权限
  
  - 表层级，适用于指定表中的所有列。`GRANT ALL ON 数据库名.表名`授予表权限；`REVOKE ALL ON 数据库名.表名`撤销表权限
  
  - 列层级，适用于指定表中的单一列。
  
  - 子程序层级

- GRANT语法：`GRANT 权限类型 [(具体列)] [, 权限类型 [(具体列)] ... ON [TABLE | FUNCTION | PROCEDURE] 表1, 表2, ..., 表n TO '用户名'@'主机名' IDENTIFIED BY '密码' [WITH GRANT OPTION]`
  
  - WITH后面的GRANT OPTION表示被授权的用户可以将这些权限赋予别的用户

- 收回所有用户的所有权限：`REVOKE ALL PRIVILEGES, GRANT OPTION FROM '用户名'@'主机名' [, '用户名'@'主机名', ...]`

- 收回指定权限： `REVOKE 权限类型 [(具体列)] [, 权限类型 [(具体列)] ... ON 表1, 表2, ..., 表n FROM '用户名'@'主机名' [, '用户名'@'主机名', ...]`

- 查看权限： `SHOW GTANTS FOR '用户名'@'主机名';`

- 创建角色： `CREATE ROLE 角色名`

- 给角色授予权限： `GRANT SELECT ON db.* TO '角色名'`

- 删除角色权限：`REVOKE SELECT ON db.* FROM '角色名'`

- 给用户赋予角色： `GRANT '角色名' TO '用户名'@'主机名'`

- 删除角色： `DROP ROLE 角色名`

# 备份和恢复

- mysqldump命令备份： `mysqldump -u 用户名 -h 主机名 -p密码 数据库名 [表名, [表名, ...]] > 备份文件名.sql`

- mysqldump备份多个数据库： `mysqldump -u 用户名 -h 主机名 -p密码 --databases [数据库名 [, 数据库名, ...]] > 备份文件名.sql`

- 恢复： `mysql -u 用户名 -h 主机名 -p密码 [数据库名] < 备份文件名.sql`

- 已登录服务器时可以使用source命令导入sql文件： `source 文件名`

- 迁移： `mysqldump -u 用户名 -h 源主机名 -p密码 数据库名 | mysql -u 用户名 -h 目标主机名 -p密码 `

- 在服务器上导出文本文件： `SELECT 列 FROM 表 WHERE 条件 INTO OUTFILE '文件名' [选项]`，其中选项有如下取值：
  
  - FIELDS TERMINATED BY '分隔符'：设置字段间的分隔符，默认情况下为制表符`\t`
  
  - FIELDS [OPTIONALLY] ENCLOSED BY '包围字符'：设置字段的包围字符
  
  - FIELDS ESCAPED BY '转义字符'：设置转义字符，默认值为`\`
  
  - LINES STARTING BY '每行数据开头字符'：设置每行数据开头的字符
  
  - LINES TERMINATED BY '每行数据结尾字符'：设置每行数据结尾的字符，默认为`\n`

- 在客户主机导出文件： `mysql -u 用户名 -p密码 -e "select ..." 数据库名 > 文件名` 或`mysql -u 用户名 -p密码 --execute= "select ..." 数据库名 > 文件名`，其他参数：
  
  - `--vertical` 显示结果
  
  - `--html` 将查询结果导出到html文件中： `mysql -u 用户名 -p密码 --html --execute= "select ..." 数据库名 > 文件名.html`
  
  - `--xml` 将查询结果导出到xml文件中： `mysql -u 用户名 -p密码 --xml --execute= "select ..." 数据库名 > 文件名.xml`

- mysqldump导出文本文件： `mysqldump -T 导出数据的目录 -u 用户名 -p密码 数据库名 [表] [选项]`，其中选项取值如下：
  
  - `--fields-terminated-by=分隔符`：设置字段间的分隔符，默认情况下为制表符`\t`
  
  - `--fields-enclosed-by=包围字符`：设置字段的包围字符
  
  - `--fields-optionally-enclosed-by=包围字符`：设置字段的包围字符，只能为单个字符，只能包括CHAR和VARCHAR等字符数据字段
  
  - `--fields-escaped-by=转移字符`：设置转义字符，默认值为`\`
  
  - `--lines-terminated-by=每行数据结尾字符`：设置每行数据结尾的字符，默认为`\n`

- 在服务器导入文本文件： `LOAD DATA INFILE '文件名' INTO TABLE 表名 [选项] [IGNORE number LINES]`
  
  - 选项取值有：
    
    - FIELDS TERMINATED BY '分隔符'：设置字段间的分隔符，默认情况下为制表符`\t`
    
    - FIELDS [OPTIONALLY] ENCLOSED BY '包围字符'：设置字段的包围字符
    
    - FIELDS ESCAPED BY '转义字符'：设置转义字符，默认值为`\`
    
    - LINES STARTING BY '每行数据开头字符'：设置每行数据开头的字符
    
    - LINES TERMINATED BY '每行数据结尾字符'：设置每行数据结尾的字符，默认为`\n`
  
  - `[IGNORE number LINES]`表示忽略文件开始处的行数

- 在客户主机导入文本文件： `mysqlimport -u 用户名 -p密码 数据库名 文件名 [选项]`，选项取值如下：
  
  - `--fields-terminated-by=分隔符`：设置字段间的分隔符，默认情况下为制表符`\t`
  
  - `--fields-enclosed-by=包围字符`：设置字段的包围字符
  
  - `--fields-optionally-enclosed-by=包围字符`：设置字段的包围字符，只能为单个字符，只能包括CHAR和VARCHAR等字符数据字段
  
  - `--fields-escaped-by=转移字符`：设置转义字符，默认值为`\`
  
  - `--lines-terminated-by=每行数据结尾字符`：设置每行数据结尾的字符，默认为`\n`
  
  - `--ignore-lines=n`：忽略数据文件的前n行

- mysqlimport其他选项
  
  - `--columns=column_list` 或 `-c column_list`：逗号分割，指定列名
  
  - `--compress`或`-C`：压缩在客户端和服务器短之间发送的信息
  
  - `--delete`或`-d`：导入文本文件前清空表
  
  - `--force`或`-f`：忽略错误
  
  - `--host=主机名`或`-h 主机名`：将数据导入给定的主机
  
  - `--ignore`或`-i`
  
  - `--replace`或`-r`： 该选项和`--ignore`选项控制复制唯一键值已存在的情况，如果指定了`--replace`则新的数据会替换已有的；如果指定`--ignore`则新的会被跳过；如果两个选项都不指定，则报错
  
  - `--ignore-lines=n`：忽略数据文件前n行
  
  - `--local`或`-L`：从本地客户端读取文件
  
  - `--lock-tables`或`-l`：处理文本文件前锁定所有表
  
  - `--password[=密码]`或`-p[密码]`：连接服务器的密码
  
  - `--port=端口号`或`-P 端口号`：连接的端口号
  
  - `--protocol={TCP | SOCKET | PIPE | MEMROY}`：使用的连接协议
  
  - `--slient`或`-s`：沉默模式，只有出现错误时才会输出信息
  
  - `--user=用户名`或`-u 用户名`：连接服务器的用户名
  
  - `--verbose`或`-v`：打印详细信息
  
  - `--version`或`-V`：版本信息