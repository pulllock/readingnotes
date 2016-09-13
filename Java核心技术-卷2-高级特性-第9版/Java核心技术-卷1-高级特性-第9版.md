# 流与文件

## 流
InputStream和OutputStream

Reader和Writer

### 读写字节
InputStream 中抽象方法：

```
abstract int read();
```
读入一个字节，返回读入的字节，遇到结尾时返回-1。

OutputStream中：

```
abstract void write();
```
向某个位置写出一个字节。

read和write方法执行时都将阻塞，直至字节确实被读入或写出。

available方法可以检查当前可读入的字节数量。

### 组合流过滤器
FIleInputStream和FIleOutputStream提供文件的输入流和输出流，需要提供给文件名和文件路径。

## 文本输入和输出
OutputStreamWriter类将使用选定的字符编码方式，把Unicode字符流转换为字节流。

InputStreamReader将包含字节的输入流转换为可以产生Unicode码的读入器。

# 网络
Socket


