# 廖雪峰的javascript教程

使用==比较会自动转换类型，所以坚持使用===

字符串不可变

遍历Array可以采用下标循环，遍历Map和Set就无法使用下标。为了统一集合类型，ES6标准引入了新的iterable类型，Array、Map和Set都属于iterable类型。

具有iterable类型的集合可以通过新的for ... of循环来遍历。