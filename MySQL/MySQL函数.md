# MySQL函数

- 数学函数
- 字符串函数
- 日期时间函数
- 条件判断函数
- 系统信息函数
- 加密函数

# 数学函数

| 函数           | 说明                                 |
| -------------- | ------------------------------------ |
| abs(x)         | 绝对值                               |
| pi()           | 圆周率                               |
| sqrt(x)        | 平方根                               |
| mod(x, y)      | 求余函数                             |
| ceil(x)        | 返回不小于x的最小整数值              |
| ceiling(x)     | 返回不小于x的最小整数值              |
| floor(x)       | 返回不大于x的最小整数值              |
| rand()         | 获取随机数                           |
| rand(x)        | 获取随机数                           |
| round(x)       | 返回最接近x的整数，对x进行四舍五入   |
| round(x, y)    | 返回最接近x的数，值保留到小数点后y位 |
| truncate(x, y) | 返回被舍去至小数点后y位的数字x       |
| sign(x)        | 返回参数的符号                       |
| pow(x, y)      | 返回x的y次方的结果                   |
| power(x, y)    | 返回x的y次方的结果                   |
| exp(x)         | 返回e的x次方的结果                   |
| log(x)         | 返回x的自然对数                      |
| log10(x)       | 返回x的基数为10的对数                |
| radians(x)     | 将参数x由角度转化为弧度              |
| degrees(x)     | 将参数x由弧度转化为角度              |
| sin(x)         | 返回x的正弦                          |
| asin(x)        | 返回x的反正弦                        |
| cos(x)         | 返回x的余弦                          |
| acos(x)        | 返回x的反余弦                        |
| tan(x)         | 返回x的正切                          |
| atan(x)        | 返回x的反正切                        |
| cot(x)         | 返回x的余切                          |

# 字符串函数

| 函数                      | 说明                                                      |
| ------------------------- | --------------------------------------------------------- |
| char_length(str)          | 返回str包含的字符个数                                     |
| length(str)               | 返回字符串的字节长度                                      |
| concat(s1, s2, ...)       | 连接字符串                                                |
| concat_ws(x, s1, s2, ...) | 使用指定的分隔符x丽娜届字符串                             |
| insert(s1, x, len, s2)    | 替换字符串                                                |
| lower(str)                | 转小写                                                    |
| lcase(str)                | 转小写                                                    |
| upper(str)                | 转大写                                                    |
| ucase(str)                | 转大写                                                    |
| left(s, n)                | 返回字符串s开始的最左边n个字符                            |
| right(s, n)               | 返回字符串s最右边的n个字符                                |
| lpad(s1, len, s2)         | 返回字符串s1,其左边由字符串s2填充到len字符长度            |
| rpad(s1, len, s2)         | 返回字符串s1,其右边由字符串s2填充到len字符长度            |
| ltrim(s)                  | 删除左侧空格                                              |
| rtrim(s)                  | 删除右侧空格                                              |
| trim(s)                   | 删除两侧空格                                              |
| trim(s1 from s)           | 删除字符串s两端所有的子字符s1                             |
| repeat(s, n)              | 返回一个由重复的字符串s组成的字符串，字符串s的数目等于n   |
| space(n)                  | 返回一个由n个空格组成的字符串                             |
| replace(s, s1, s2)        | 使用字符串s2代替字符串s中所有的字符串s1                   |
| strcmp(s1, s2)            | 比较字符串大小                                            |
| substring(s, n, len)      | 获取子串，从字符串s返回一个长度为len的字符串，起始于位置n |
| mid(s, n, len)            | 获取子串，从字符串s返回一个长度为len的字符串，起始于位置n |
| locate(s1, s)             | 返回子字符传s1在字符串s中的开始位置                       |
| position(s1 in s)         | 返回子字符传s1在字符串s中的开始位置                       |
| instr(s, s1)              | 返回子字符传s1在字符串s中的开始位置                       |
| reverse(s)                | 将字符串s反转                                             |
| elt(n, s1, s2, s3, ...)   | 返回指定位置的字符串                                      |
| field(s, s1, s2, ...)     | 返回指定字符串位置的函数                                  |
| find_in_set(s1, s2)       | 返回子串位置的函数，返回s1在字符串列表s2中出现的位置      |
| make_set(x, s1, s2, ...)  | 按x的二进制数从s1, s2, ...中选取字符串                    |

# 日期时间函数

| 函数                               | 说明                                                 |
| ---------------------------------- | ---------------------------------------------------- |
| curdate()                          | 获取当前日期                                         |
| current_date()                     | 获取当前日期                                         |
| curtime()                          | 获取当前时间                                         |
| current_time()                     | 获取当前时间                                         |
| current_timestamp()                | 获取当前日期和时间                                   |
| localtime()                        | 获取当前日期和时间                                   |
| now()                              | 获取当前日期和时间                                   |
| sysdate()                          | 获取当前日期和时间                                   |
| unix_timestamp(date)               | 返回unix时间戳                                       |
| from_unittime(date)                | 把unix时间戳转换为普通格式时间                       |
| utc_date()                         | 返回当前UTC日期                                      |
| utc_time()                         | 返回当前UTC时间值                                    |
| month(date)                        | 返回date对应的月份                                   |
| monthname(date)                    | 返回date对应的月份的英文全名                         |
| dayname(d)                         | 返回d对应的工作日的英文名                            |
| dayofweek(d)                       | 返回d对应的一周中的索引                              |
| weekday(d)                         | 返回d对应的工作日的索引                              |
| week(d)                            | 计算日期d是一年中的第几周                            |
| weekofyear(d)                      | 计算某天位于一年中的第几周                           |
| dayofyear(d)                       | 返回d是一年中的第几天                                |
| dayofmonth(d)                      | 返回d是一个月中的第几天                              |
| year(date)                         | 返回date对应的年份                                   |
| quarter(date)                      | 返回date对应一年中的季度值                           |
| minute(time)                       | 返回time对应的分钟数                                 |
| second(time)                       | 返回time对应的秒数                                   |
| extract(type from date)            | 获取日期指定值，type可为year，year_month，day_minute |
| time_to_sec(time)                  | 将time转化为秒                                       |
| sec_to_time(seconds)               | 将秒转化为time                                       |
| date_add(date, interval expr type) | 日期加运算                                           |
| adddate(date, interval expr type)  | 日期加运算                                           |
| date_sub(date, interval expr type) | 日期减运算                                           |
| subdate(date, interval expr type)  | 日期减运算                                           |
| addtime(date, expr)                | 时间加运算                                           |
| subtime(date, expr)                | 时间减运算                                           |
| date_diff(date1, date2)            | 返回起始时间date1和结束时间date2之间的天数           |
| date_format(date, format)          | 根据format指定的格式显示date值                       |
| time_format(time, format)          | 根据format指定的格式显示time值                       |
| get_format(val_type, format_type)  | 返回日期时间字符串的显示格式                         |

# 条件判断函数

| 函数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| if(expr, v1, v2)                                             | 如果expr为true，则返回v1，否则返回v2                         |
| ifnull(v1, v2)                                               | 如果v1不为null则返回v1，否则返回v2                           |
| case expr when v1 then r1 [when v2 then r2] ... [else rn+1] end | 如果expr等于某个v，则返回对应的then后面的结果；如果所有值都不相等，则返回else后的值 |

# 系统信息函数

| 函数             | 说明                           |
| ---------------- | ------------------------------ |
| version()        | 返回MySQL服务器的版本          |
| connection_id()  | 返回当前连接的次数             |
| database()       | 返回当前使用的数据库           |
| schema()         | 返回当前使用的数据库           |
| user()           | 返回用户名和主机名             |
| current_user     | 返回用户名和主机名             |
| current_user()   | 返回用户名和主机名             |
| system_user()    | 返回用户名和主机名             |
| session_user()   | 返回用户名和主机名             |
| charset(str)     | 返回str的字符集                |
| collation(str)   | 返回str的字符排序方式          |
| last_insert_id() | 返回最后生成的auto_increment值 |

# 加密函数

| 函数     | 说明    |
| -------- | ------- |
| md5(str) | md5加密 |
| sha(str) | sha加密 |

# 集合函数

| 函数    | 说明                           |
| ------- | ------------------------------ |
| max()   | 返回指定列中最大值             |
| min()   | 返回指定列中最小值             |
| count() | 统计数据表中包含的记录行的总数 |
| sum()   | 求和                           |
| avg()   | 求平均值                       |



# 其他函数

| 函数                        | 说明                                                      |
| --------------------------- | --------------------------------------------------------- |
| format(x, n)                | 将数字x格式化，并以四舍五入的方式保留小数点后n位          |
| conv(n, from_base, to_base) | 进行不同进制数间的转换，将n从from_base进制转为to_base进制 |
| inet_aton(expr)             | 给出一个作为字符串的网络地址的点地址表示                  |
| inet_ntoa                   | 给定一个数字网络地址返回作为字符串的该地址的点地址表示    |
| get_lock(str, timeout)      | 使用str给定的名字得到一个锁，超时时间为timeout秒          |
| release_lock(str)           | 释放锁                                                    |
| is_free_lock(str)           | 检查锁是否可以使用                                        |
| is_used_lock(str)           | 检查锁是否正在被使用                                      |
| benchmark(count, expr)      | 重复count次执行表达式expr                                 |
| convert(... using ...)      | 改变字符集                                                |
| cast(x, as type)            | 改变数据类型                                              |
| convert(x, type)            | 改变数据类型                                              |
| window                      | 窗口函数                                                  |
| group_concat(字段)          | 将分组中字段值显示出来                                    |

