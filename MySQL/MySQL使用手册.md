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