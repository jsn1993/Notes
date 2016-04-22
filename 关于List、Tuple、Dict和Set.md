<b>

List是有序的集合，可以随时删除和添加其中的元素

tuple和list非常类似，但是tuple一旦初始化就不能修改

dict全称dictionary，在其他语言中也称为map，使用键-值（key-value）存储，具有极快的查找速度

set和dict类似，也是一组key的无序集合，但不存储value。由于key不能重复，所以，在set中，没有重复的key

和list比较，dict有以下几个特点：

    1.    查找和插入的速度极快，不会随着key的增加而增加；
    2.    需要占用大量的内存，内存浪费多。
而list相反：

    1.    查找和插入的时间随着元素的增加而增加；
    2.    占用空间小，浪费内存很少。

——set——

支持add添加、remove删除

支持 x in set, len(set),和 for x in set

支持求交集a & b， 求差集a - b，求并集a|b

经常用在数据的去重处理和一些数据的中转处理

    ~$ python -m timeit -n 1000 "[x for x in range(1000) if x in range(500, 1500)]"
        1000 loops, best of 3: 28.2 msec per loop
    ~$ python -m timeit -n 1000 "set(range(1000)).intersection(range(500, 1500))"
        1000 loops, best of 3: 120 usec per loop
List 大概用了Set的225倍的时间。

List转Set基本用不了什么时间，所以如果有需要求（集合，列表等）的并集和交集的时候，最好使用Set。

转化 l = list(s)  s = set(l)

————

————

————

redis数据处理相关：存入dict或list取出变成string的问题

——dict与string的转化——

    import ast
    ast.literal_eval("{'x':1, 'y':2}")
    => {'y': 2, 'x': 1}
——list与string转化——

    import ast
    ast.literal_eval("{'x':[1,2]")['x']
    => [1,2]

所以，存入dict即可。

