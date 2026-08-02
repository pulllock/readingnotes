JDK 9开始HotSpot日志开始统一，所有日志功能都使用 `-Xlog`参数，格式如下： `-Xlog[:[selections][:[output][:[decorators][:output-options]]]]`，可以使用命令 `java -Xlog:help`来查看详细信息。

查看当前JVM使用的垃圾收集器：

- Java 8以及以前的版本： `java -XX:+PrintCommandLineFlags -version`
- Java 9以及以后的版本： `java -Xlog:gc -version`

打印收集器日志参数：

- Java 8以及以前的版本
	- `-XX:+PrintGC`： 打印基础GC概要信息，也可使用 `-verbose:gc`代替
	- `-XX:+PrintGCDetails`： 打印详细的GC回收前后内存大小及各分代的占用情况
	- `-XX:+PrintGCDateStamps`： 打印绝对的日期时间
	- `-XX:+PrintGCTimeStamps`： 打印相对于JVM启动的秒数
	- `-XX:+PrintGCCause`： 打印触发GC的原因
	- `-Xloggc:/var/log/jvm/gc-%t.log`： 指定日志输出的文件路径，`%t`会自动生成时间戳后缀，防止重启服务时覆盖旧日志
	- `-XX:+UseGCLogFileRotation`： 开启滚动日志
	- `-XX:NumberOfGCLogFiles=5`： 日志文件只保留的个数
	- `-XX:GCLogFileSize=20M`： 每个文件最大大小
- Java 9以及以后的版本，格式： `-Xlog[:[selections][:[output][:[decorators][:output-options]]]]`
	- `-Xlog:gc`： 打印基础GC概要信息，等价于 `-XX:+PrintGC`
	- `-Xlog:gc*`： 打印详细的GC日志，等价于 `-XX:+PrintGCDetails`
	- `:time`： 显示绝对日期
	- `:uptime`： 显示启动相对时间
	- `:file=/path/to/gc.log`： 指定日志输出到文件
	- `::filecount=5`： 日志文件个数
	- `,filesize=50M`： 每个文件大小

内存参数：

- `-X`：代表 E(X)tended（非标准扩展参数）
- `-Xms`：堆内存初始大小或者最小大小
	- 全称：E(X)tended Memory Starting size或Memory Size (Initial)
	- `-X`：代表E(X)tended（非标准扩展参数）
	- `m`：代表Memory（内存）
	- `s`：代表Starting（启动/初始）或Size
- `-Xmx`：堆内存最大值
	- 全称：E(X)tended Memory Ma(X)imum size
	- `-X`：代表E(X)tended（非标准扩展参数）
	- `m`：代表Memory（内存）
	- `x`：代表Ma(X)imum（最大值）
- `-Xmn`：新生代大小
	- 全称：E(X)tended Memory Nursery size或Memory New generation size
	- `-X`：代表E(X)tended（非标准扩展参数）
	- `m`：代表Memory（内存）
	- `n`：代表Nursery（苗圃/托儿所）或New（新一代）