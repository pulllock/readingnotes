# 基本概念
## 数据类型
### object类型

Object类型是所有它的实例的基础，Object类型所具有的任何属性和方法也同样存在于更具体的对象中。Object的每个实例都具有下列属性和方法：

- Constructor，保存着用于创建当前对象的函数。
- hasOwnProperty(propertyName)，用于检查给定的属性在当前对象实例中是否存在。
- isPrototypeOf(object)，用于检查传入的对象是否是另一个对象的原型。
- propertyIsEnumerable(propertyName)，用于检查给定的属性是否能够使用for-in语句来枚举。
- toLocaleString，返回对象的字符串表示，该字符串与执行环境的地区对应。
- toString，返回对象的字符串表示。
- valueOf，返回对象的字符串、数值或布尔值表示，通常与toString方法的返回值相同。