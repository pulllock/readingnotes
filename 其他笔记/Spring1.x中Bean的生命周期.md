主要记录下学习源码的一些东西，这里主要是以Spring1.1.1源码为基础，分析下Bean的生命周期。分为两个部分：BeanFactory和ApplicationContext。

# BeanFactory中Bean的生命周期
1. 容器寻找Bean的定义信息，并将其实例化。
