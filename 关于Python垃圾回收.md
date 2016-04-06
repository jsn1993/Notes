<br>Python垃圾回收

现在的主流语言基本都支持三种最基本的内存分配方式:

A）静态分配，如静态变量、全剧变量，通常这一部分内存不需要释放或者回收；

B）自动分配，通常由语言或者编译器控制，如栈中局部变量、参数传递；

C）动态分配，这部分通常在栈中分配，由用户申请。

垃圾收集机制就是用来处理程序动态分配内存时产生的垃圾。

————————

python垃圾回收以引用计数为主，标记-清除和分代收集为辅。

引用计数最大缺陷就是循环引用的问题。什么是循环引用？A和B相互引用而再没有外部引用A与B中的任何一个，它们的引用计数虽然都为1，但显然应该被回收，例子：

    
    a = { } # a 的引用为 1
    b = { } # b 的引用为 1
    a['b'] = b # b 的引用增 1，b的引用为2
    b['a'] = a # a 的引用增 1，a的引用为 2
    del a # a 的引用减 1，a的引用为 1
    del b # b 的引用减 1, b的引用为 1
在这个例子中,del语句减少了 a 和 b 的引用计数并删除了用于引用的变量名，可是由于两个对象各包含一个对方对象的引用，虽然最后两个对象都无法通过名字访问了，但引用计数并没有减少到零。因此这个对象不会被销毁，它会一直驻留在内存中，这就造成了内存泄漏。为了解决循环引用问题，Python引入了标记-清除和分代回收两种GC机制。

——标记清除——

标记——清除（Mark——Sweep）是一种基于追踪（Tracing）回收技术实现的垃圾回收算法，对象之间通过引用（指针）连在一起，构成一个有向图，对象构成这个有向图的节点，而引用关系构成这个有向图的边。从根对象（root object）出发，沿着有向边遍历对象：标记阶段，遍历所有的对象，如果是可达的（reachable），也就是还有对象引用它，那么就标记该对象为可达；清除阶段，再次遍历对象，如果发现某个对象没有标记为可达，则就将其回收。所谓根对象就是一些全局引用对象和函数栈中的引用，这些引用所引用的对象是不可被删除的。注意，在进行垃圾回收的阶段，会暂停整个应用程序，等待标记清除结束后才会恢复应用程序的运行。

gc.collect()返回此次垃圾回收的unreachable(不可达)对象个数。

    a=[]  
    b=[]  
    a.append(b)  
    b.append(a)  
    del a  
    del b  
    print gc.collect()  

寻找root object集合，root object多指全局引用和函数栈上的引用，如上面代码所示，a就是root object。

从root object出发，通过其每一个引用到达的所有对象都标记为reachable（垃圾检测）。

将所有非reachable的对象删除（垃圾回收）。

这里还需要提到垃圾回收中的->>可收集对象链表，Python将所有可能产生循环引用的对象用链表连接起来，所谓的可产生循环引用的对象也就是list,dict,class等的容器类，int,string不是，每次实例化该种对象时都将加入这个链表，我们将该链表称为可收集对象链表(ps该链表是双向的)。

如，a=[],b=[],c={},将会产生：head <----> a  <----> b <----> c 双向链表。

我们可以假想上述代码的垃圾回收过程：当调用gc.collect()时，将从root object开始垃圾回收，由于del a ,del b后，a,b都将成为unreachable对象，且循环引用将被拆除，此时a,b引用数都是0，a,b将被回收，所以collect将返回2。

    a=[]  
    b=[]  
    a.append(b)  
    b.append(a)  
    del b  
    print gc.collect()  
此次并没有垃圾回收，虽然del b了，但从a出发，找到了b的引用，所以b还是reachable对象，所以并不会被收集。

Python有了垃圾回收机制也可能会造成内存泄漏

    class A:
        def __del__(self):
        pass  
    class B:
        def __del__(self):
            pass  
    a=A()  
    b=B()  
    print hex(id(a))  
    print hex(id(a.__dict__))  
    a.b=b  
    b.a=a  
    del a  
    del b  
    print gc.collect()  
    print gc.garbage  
从输出中我们看到uncollectable字样，很明显这次垃圾回收搞不定了，造成了内存泄漏。因为del b时,会调用b的__del__方法，该方法中很可能使用了b.a，但如果在之前的del a时将a给回收掉，此时将造成异常。所以Python没办法，造成了uncollectable，也就产生了内存泄漏。所以__del__方法要慎用，如果用的话一定要保证没有循环引用。

——分代回收——

分代回收是一种以空间换时间的操作方式，Python将内存根据对象的存活时间划分为不同的集合，每个集合称为一个代，Python将内存分为了3“代”，分别为年轻代（第0代）、中年代（第1代）、老年代（第2代），他们对应的是3个链表，它们的垃圾收集频率与对象的存活时间的增大而减小。活的越久的对象，就越不是垃圾，回收的频率就应该越低。当Python发现进过几次垃圾回收该对象都是reachable，就将该对象移到二代中，以此类推。新创建的对象都会分配在年轻代，年轻代链表的总数达到上限时，Python垃圾收集机制就会被触发，把那些可以被回收的对象回收掉，而那些不会回收的对象就会被移到中年代去，依此类推，老年代中的对象是存活时间最久的对象，甚至是存活于整个系统的生命周期内。同时，分代回收是建立在标记清除技术基础之上，在执行标记清除算法时可以有效减小遍历的对象数，从而提高垃圾回收的速度。


——引用计数——

一切皆对象 id(obj)/type(obj)/value

关系运算符 "==" 比较的是两个对象的值是否相等，而 is 比较的是两个变量是否为同一个对象（或者说指向同一块内存）。对两个变量赋予相同的值，它们有可能是同一个对象（对不可变对象而言，可以节省内存），也可能是两个不同的对象，这可能取决于对象的类型（type）。

解释器跟踪对象的引用计数，而垃圾收集器负责释放内存。当一个对象的引用计数变为0，解释器会暂停，释放掉这个对象和仅有这个对象可访问的其他对象。

每个变量都和指向对象（object）的指针相关联，每一个object都有一个reference counter（引用计数器）记住有多少个变量和这个object绑定（bind）。每次bind，reference count都加1，每次删除bind关系，都减少1，只有reference counter变成0的时候才真正删除对象。

以下情况时，对象的引用计数增加

1.对象被创建；

2.另外的别名被创建；

3.作为参数传递给函数；

4.成为容器对象的一个元素；

以下情况时，对象的引用计数减少

1.一个本地引用离开其作用范围，比如函数结束时，所有局部非循环引用变量都被自动销毁；

2.用del语句显式删除一个变量（同时该变量从name space中删除）；

3.对象的一个别名被赋值给其他对象；

4.对象被移出一个容器对象时；

5.容器对象本身的引用计数变成0；

例1

    x = 3.14         　　　　　　　　# 创建的3.14 这个对象的引用计数为1
    y = x        　　　　　　　　　　# 创建对象别名，对象”3.14”的引用计数为2
    myList = [123, x, ‘xyz’]      # 成为容器的一个元素，对象”3.14”的引用计数为3
    y = 123        　　　　　　　　 # 对象别名bind到其它对象，对象”3.14”的引用计数为2
    del myList        　　　　　　  # 容器被删除，对象”3.14”的引用计数为1

例2

    >>> foo = "XYZ"                            // 此时引用计数是1
    >>> sys.getrefcount(foo)                   // 返回2，包括了getrefcount()的参数引用
    >>> bar = foo                              // 2
    >>> foo = 123                              // 1

字符串对象 “XYZ” 被创建并赋值给 foo 时，引用计数是 1，当增加了一个别名 bar 时，引用计数变成了 2，不过当 foo 被重新赋值给整数对像 123 时，”XYZ” 对象的引用计数自减 1 ，又变成 1 了。


数字、字符串、元组是不可变对象，列表和字典是可变对象。

对于不可改变类型来说，无法通过变量更新对象的值，例如：
    
    >>> x = 12
    >>> id(x)
    8483376
    >>> x = 45
    >>> id(x)
    8484576
这里，表面上看变量x的值被更新了，其实是12这个数字对象被销毁了，然后创建了一个新的数字对象45，变量x被bind到这个新对象上。

不可变对象作为左值时，必须是一个完整的对象。不可变对象的方法通常有返回值，而可变对象的方法通常返回None。这是因为不可变对象因为对象自身无法修改，因此其方法只能返回一个新对象；而可变对象直接原地修改原对象。



————浅拷贝深拷贝————

序列类型对象（比如list）的赋值操作只是一种简单的浅拷贝

    >>> a = [1,2,3]   
    >>> b = a
    >>> id(a)      
    139972192906488
    >>> id(b)
    139972192906488
    >>> b[2] = "hello"
    >>> a
    [1, 2, 'hello']
可见，列表a和列表b指向对一个对象，修改列表b之后，列表a也相应的受到影响了。

import copy    copy.deepcopy    深拷贝

字典使用dict.copy()深拷贝



————weakref 弱引用————

一个对象若只被弱引用所引用，则被认为是不可访问（或弱可访问）的，并因此可能在任何时刻被回收。主要作用就是减少循环引用，减少内存中不必要的对象存在的数量。

对比

    import weakref
    import gc
    class NewObj(object):
        def my_method(self): 
            print "called me " 

    obj = NewObj()
    r = weakref.ref(obj)
    gc.collect()
    print r() is obj

    obj = 1
    gc.collect() #
    print r() is None, r()

    print '*******************' 
    obj = NewObj()
    s = obj
    gc.collect()
    print s is obj

    obj = 1
    gc.collect()
    print s is None, s

对比结果：

    True True None ******************* True False <__main__.NewObj object at 0x024FB870>

弱引用计数器没有增加，所以当obj不在引用NewObj的时候，NewObj对象就被释放了，所以r的引用对象就没了。

例2

    import weakref
    class ExpensiveObject(object):
        def __del__(self):
            print '(Deleting %s)' % self
    obj = ExpensiveObject()
    r = weakref.ref(obj)
    print 'obj:', obj
    print 'ref:', r
    print 'r():', r()
    del obj
    print 'r():', r()

输出

    obj: <__main__.ExpensiveObject object at 0x10046d410>
    ref: <weakref at 0x100467838; to 'ExpensiveObject' at 0x10046d410>
    r(): <__main__.ExpensiveObject object at 0x10046d410>
    (Deleting <__main__.ExpensiveObject object at 0x10046d410>)
    r(): None

或者
    
    def callback(reference):
        """Invoked when referenced object is deleted"""
        print 'callback(', reference, ')'
    obj = ExpensiveObject()
    r = weakref.ref(obj, callback)

例3

    >>> import weakref, gc
    >>> class A:
    ...     def __init__(self, value):
    ...             self.value = value
    ...     def __repr__(self):
    ...             return str(self.value)
    ...
    >>> a = A(10)                   # create a reference
    >>> d = weakref.WeakValueDictionary()
    >>> d['primary'] = a            # does not create a reference
    >>> d['primary']                # fetch the object if it is still alive
    10
    >>> del a                       # remove the one reference
    >>> gc.collect()                # run garbage collection right away
    0
    >>> d['primary']                # entry was automatically removed
    KeyError: 'primary'

