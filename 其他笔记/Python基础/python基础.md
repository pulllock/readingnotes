### 源程序编码
在文件的首行添加如下：

```
# _*_ coding: utf-8 _*_
```

### 注释

`# xxxxx`注释一行

### 数字和运算
1,2,3等的类型是int，整型，boolean是integer的子类型

1.0,2.1等的类型是float浮点数

`/`表示除法，永远返回一个浮点数

`//`除法，得到的结果是一个整数，去掉了小数部分

`%`取余

`**`计算幂乘方

**整数和浮点数的混合计算中，整数会被转换为浮点数**

long，长整型，具有无限精度

Decimal，有固定精度的浮点数

Fraction，分数，精确表示任何有理数

复数，使用后缀i或者J表示虚数部分

python3中已经将一般整形和长整型合二为一了！

### 列表
列表支持切片操作，返回的新列表是一个新的浅拷贝副本

`append()`方法可以在列表末尾添加新的元素

### 循环技巧

在字典中循环时，key和value可以用items()方法同时读出来：

```
adict = {'sam':'sammy', 'bob':'boby'}
for k, v in adict.items():
	print(k, v)
```

在序列中循环时，索引和值可以使用enumerate()得到：

```
for i, v in enumerate(['tic', 'tac', 'toe']:
	print(i, v))
```

同时循环两个或者更多的序列，可以使用zip()方法打包：

```
questions = ['name', 'quest', 'color']
answers = ['fk', 'shshhs', 'yellow']

for q, a in zip(questions, answwes):
	print(q, a)
```

逆向循环序列，reversed()：

```
for i in reversed(range(1,10)):
	print(i)
```

按照排序后的序列排序，使用sorted()：

```
basket = ['orange', 'pear', 'apple', 'banaba']
for f in sorted(basket):
	print(f)
```

### 格式化输出

`str.rjust()`把字符串输出到一列，并通过向左侧填充空格来使其右对齐

`str.ljust()`左对齐

`str.center()`中间对齐

`str.zfill()`向数值字符串的左侧填充0

`str.format()`可以把大括号替换成传入的参数


