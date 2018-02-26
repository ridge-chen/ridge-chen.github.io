
python装饰器是一个函数，功能是在不修改已有代码（包括被装饰函数的本身代码，以及原有的调用被装饰函数的代码）的情况下，为其他函数增加额外的功能。

python装饰器通常用于面向切面（Aspect Oriented Programming）的场景上，比如日志插入，权限校验，事务处理。

## 没有参数的装饰器 ##

先来看一段测试代码：

    def decorator(func):
        def wrapper(*args, **kwargs):
            print "in wrapper()"
            return func(*args, **kwargs)
    
        print "in decorator()"
        print "func=%s" % func
        print "wrapper=%s" % wrapper
        return wrapper
    
    @decorator
    def foo():
        print "in foo()"
    
    print "in main()"
    print "foo=%s" % foo
    foo()

这段代码中，`decorator`函数是一个装饰器，当用它来装饰一个函数`foo`时，将修改原本函数的逻辑: 

1. 首先打印一条消息`print "in wrapper()"`。
2. 执行原本函数`foo`

**装饰的过程其实就是改变`foo`这个变量指向的函数对象**，从`<function foo at 0x7ff8cefd6668>`改为`<function wrapper at 0x7ff8cefd66e0>`。**修改后的函数对象为装饰器执行后返回值。**

另外为了了解装饰器本身的执行过程，装饰器本身在执行过程中也会打印`print "in decorator()"`消息，并打印被装饰的函数和装饰后的函数对象。我们可以发现，**装饰的过程对每个被装饰函数只出现一次，在被装饰函数声明（也是装饰）的时刻。**

执行结果：

    in decorator()
    func=<function foo at 0x7ff8cefd6668>
    wrapper=<function wrapper at 0x7ff8cefd66e0>
    in main()
    foo=<function wrapper at 0x7ff8cefd66e0>
    in wrapper()
    in foo()

我们也可以认为，被装饰函数的声明和装饰等效于以下代码：

    def foo():
        print "in foo()"
    
    foo = decorator(foo)


## 带参数的装饰器 ##

还是先来看一段测试代码：

    def add_log(level):
        def decorator(func):
            def wrapper(*args, **kwargs):
                print "%s: in wrapper()" % level
                return func(*args, **kwargs)
    
            print "in decorator()"
            return wrapper
    
        print "in add_log()"
        return decorator
    
    @add_log("DEBUG")
    def foo():
        print "in foo()"
    
    print "in main()"
    foo()


这里装饰器的作用来上段测试代码相同-在执行原本函数前打印一条消息。但是有一个不同，**这里装饰器`decorator`是通过一次函数调用`(add_log("DEBUG")`)返回**。

执行结果：

    in add_log()
    in decorator()
    in main()
    DEBUG: in wrapper()
    in foo()

我们也可以认为，被装饰函数的声明和装饰等效于以下代码：

    def foo():
        print "in foo()"
    
    foo = add_log("DEBUG")(foo)

**其实很容易理解带参数和不带参数的装饰器的这种差异。带参数的装饰器通过一个函数调用（@add_log("DEBUG")）获得一个函数对象。而不带参数的装饰器直接使用了一个函数对象（@decorator）。**

## 多层装饰器 ##

测试代码：

    def decorator1(func):
        def wrapper1(*args, **kwargs):
            print "in wrapper1()"
            return func(*args, **kwargs)
    
        print "in decorator1()"
        print "func=%s" % func
        print "wrapper1=%s" % wrapper1
        return wrapper1
    
    def decorator2(func):
        def wrapper2(*args, **kwargs):
            print "in wrapper2()"
            return func(*args, **kwargs)
    
        print "in decorator2()"
        print "func=%s" % func
        print "wrapper2=%s" % wrapper2
        return wrapper2
    
    @decorator2
    @decorator1
    def foo():
        print "in foo()"
    
    print "in main()"
    print "foo=%s" % foo
    foo()

执行结果：

    in decorator1()
    func=<function foo at 0x7f9f304bd6e0>
    wrapper1=<function wrapper1 at 0x7f9f304bd758>
    in decorator2()
    func=<function wrapper1 at 0x7f9f304bd758>
    wrapper2=<function wrapper2 at 0x7f9f304bd7d0>
    in main()
    foo=<function wrapper2 at 0x7f9f304bd7d0>
    in wrapper2()
    in wrapper1()
    in foo()

很直观，被装饰函数的声明和装饰等效于以下代码：

    def foo():
        print "in foo()"
    
    foo = decorator2(decorator1(foo))

## 装饰器的缺点 ##

由于函数对象被替换了，从而函数的元数据也会改变。比如函数的`__doc__`，`__name__`等。

解决方法： **可以使用functools.wraps来复制函数元数据。**

例如：

    import functools
    
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            print "in wrapper()"
            return func(*args, **kwargs)
    
        print "in decorator()"
        print "func=%s" % func
        print "wrapper=%s" % wrapper
        return wrapper
    
    @decorator
    def foo():
        """doc of foo"""
        print "in foo()"
    
    print "in main()"
    print "foo=%s" % foo
    foo()
    print foo.__doc__


执行结果：

    in decorator()
    func=<function foo at 0x7fc2aa69c848>
    wrapper=<function foo at 0x7fc2aa69c8c0>
    in main()
    foo=<function foo at 0x7fc2aa69c8c0>
    in wrapper()
    in foo()
    doc of foo



