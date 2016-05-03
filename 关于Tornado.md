<b>

Tornado4.3源码（待读）

###【主要模块】
    
ioloop - tornado是以IOLoop为核心的，IOLoop 是对epoll, kqueue, select的一层包装，是异步的核心，相当于 python3.4 引入的 DefaultSelector。
    
iostream - 对非阻塞式的 socket 的简单封装，以方便常用读写操作，也会传到HTTPConnection中。

web - 基础 Web 框架，包含了 Tornado 的大多数重要的功能，其中Application 是整个web应用的配置中心，配置一些东西比如 secret_cookie 等，其中还包括了 URL正则和对应的Handler 列表。

httpserver - 一个非常简单的 HTTP 服务器的实现，WSGI 负责解析HTTP请求。

httpclient - 一个非阻塞式 HTTP 客户端，它被设计用来和 web 及 httpserver 协同工作。
    
###【其他模块】

options - 命令行和配置文件解析工具，针对服务器环境做了优化。

log - 

escape - XHTML, JSON, URL 的编码/解码方法。

template - 基于 Python 的 web 模板系统。

database - 对 MySQLdb 的简单封装。

auth - 第三方认证的实现（包括 Google OpenID/OAuth、Facebook Platform、Yahoo BBAuth、FriendFeed OpenID/OAuth、Twitter OAuth）。

locale - 针对本地化和翻译的支持。

locks - 

util -

netutil -

httputil -

testing - 

gen - 

tcpserver/tcpclient - 

websocket - 

wsgi - 

curl_httpclient - 

simple_httpclient -

http1connection - 

stack_context - 

queues - 

process - 

concurrent - 

autoreload - 


###【从服务器socket开始】

实例化

    application = web.Application([
        (r"/", MainPageHandler),
    ])
    http_server = httpserver.HTTPServer(application)
    http_server.listen(8080)
    ioloop.IOLoop.instance().start()

首先实例化Application，调用isten方法就会调用start，start做了一件最重要的事情：把自己加入IOLoop。这样子，每当有请求来，epoll就会把 当前进程拉起来，然后开始执行。

在ioloop.PollIOLoop（继承IOLoop）可以看到start方法中的while True事件循环，并且可以找到event_pairs = self._impl.poll(poll_timeout)这句使用了poll，然后就在 self._handlers 里找到对应的handler去处理。


被 tornado.web.asynchronous 装饰之后，命中这个方法的 HTTP 请求就成为长连接，直到你调用 self.finish 发送响应之前，连接都在等待状态。如果直到某个事件出发（比如有信息更新）才调用 self.finish 则这个连接会一直等待到该事件触发。


————————

###【转载】

————————

让你的请求异步非阻塞

前言

也许有同学很迷惑:tornado不是标榜异步非阻塞解决10K问题的嘛?但是我却发现不是torando不好，而是你用错了.比如最近发现一个事情:某网站打开页面很慢,服务器cpu/内存都正常.网络状态也良好. 后来发现，打开页面会有很多请求后端数据库的访问，有一个mongodb的数据库业务api的rest服务.但是它的tornado却用错了,一步步的来研究问题:

说明

以下的例子都有2个url,一个是耗时的请求，一个是可以或者说需要立刻返回的请求,我想就算一个对技术不熟，从道理上来说的用户， 他希望的是他访问的请求不会影响也不会被其他人的请求影响

    #!/bin/env python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.options
    import tornado.web
    import tornado.httpclient

    import time

    from tornado.options import define, options
    define("port", default=8000, help="run on the given port", type=int)

    class SleepHandler(tornado.web.RequestHandler):
        def get(self):
            time.sleep(5)
            self.write("when i sleep 5s")

    class JustNowHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("i hope just now see you")

    if __name__ == "__main__":
        tornado.options.parse_command_line()
        app = tornado.web.Application(handlers=[
                (r"/sleep", SleepHandler), (r"/justnow", JustNowHandler)])
        http_server = tornado.httpserver.HTTPServer(app)
        http_server.listen(options.port)
        tornado.ioloop.IOLoop.instance().start()

假如你使用页面请求或者使用哪个httpie,curl等工具先访问http://localhost:8000/sleep,再访问http://localhost:8000/justnow.你会发现本来可以立刻返回的/jsutnow的请求会一直阻塞到/sleep请求完才返回.

这是为啥?为啥我的请求被/sleep请求阻塞了？如果平时我们的web请求足够快我们可能不会意识到这个问题，但是事实上经常会有一些耗时的进程，意味着应用程序被有效的锁定直至处理结束.

这是时候你有没有想起@tornado.web.asynchronous这个装饰器？但是使用这个装饰器有个前提就是你要耗时的执行需要执行异步,比如上面的time.sleep,你只是加装饰器是没有作用的，而且需要注意的是 Tornado默认在函数处理返回时关闭客户端的连接,但是当你使用@tornado.web.asynchonous装饰器时，Tornado永远不会自己关闭连接，需要显式的self.finish()关闭

我们大部分的函数都是阻塞的, 比如上面的time.sleep其实tornado有个异步的实现:

    #!/bin/env python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.options
    import tornado.web
    import tornado.gen
    import tornado.httpclient
    import tornado.concurrent
    import tornado.ioloop

    import time

    from tornado.options import define, options
    define("port", default=8000, help="run on the given port", type=int)

    class SleepHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        @tornado.gen.coroutine
        def get(self):
            yield tornado.gen.Task(tornado.ioloop.IOLoop.instance().add_timeout, time.time() + 5)
            self.write("when i sleep 5s")


    class JustNowHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("i hope just now see you")

    if __name__ == "__main__":
        tornado.options.parse_command_line()
        app = tornado.web.Application(handlers=[
                (r"/sleep", SleepHandler), (r"/justnow", JustNowHandler)])
        http_server = tornado.httpserver.HTTPServer(app)
        http_server.listen(options.port)
        tornado.ioloop.IOLoop.instance().start()

这里有个新的tornado.gen.coroutine装饰器, coroutine是3.0之后新增的装饰器.以前的办法是用回调，还是看我这个例子:

    class SleepHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            tornado.ioloop.IOLoop.instance().add_timeout(time.time() + 5, callback=self.on_response)
        def on_response(self):
            self.write("when i sleep 5s")
            self.finish()

使用了callback, 但是新的装饰器让我们通过yield实现同样的效果:你在打开/sleep之后再点击/justnow， justnow的请求都是立刻返回不受影响.但是用了asynchronous的装饰器你的耗时的函数也需要执行异步

刚才说的都是没有意义的例子，下面写个有点用的:读取mongodb数据库数据，然后再前端按行write出来

    #!/bin/env python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.options
    import tornado.web
    import tornado.gen
    import tornado.httpclient
    import tornado.concurrent
    import tornado.ioloop

    import time
    # 一个mongodb出品的支持异步的数据库的python驱动
    import motor
    from tornado.options import define, options
    define("port", default=8000, help="run on the given port", type=int)
    # db其实就是test数据库的游标
    db = motor.MotorClient().open_sync().test

    class SleepHandler(BaseHandler):
        @tornado.web.asynchronous
        @tornado.gen.coroutine
        def get(self):
            # 这一行执行还是阻塞需要时间的，我的tt集合有一些数据并且没有索引
            cursor = db.tt.find().sort([('a', -1)])
            # 这部分会异步非阻塞的执行二不影响其他页面请求
            while (yield cursor.fetch_next):
                message = cursor.next_object()
                self.write('<li>%s</li>' % message['a'])
            self.write('</ul>')
            self.finish()

        def _on_response(self, message, error):
            if error:
                raise tornado.web.HTTPError(500, error)
            elif message:
                for i in message:
                    self.write('<li>%s</li>' % i['a'])
            else:
                self.write('</ul>')
                self.finish()


    class JustNowHandler(BaseHandler):
        def get(self):
            self.write("i hope just now see you")

    if __name__ == "__main__":
        tornado.options.parse_command_line()
        app = tornado.web.Application(handlers=[
                (r"/sleep", SleepHandler), (r"/justnow", JustNowHandler)])
        http_server = tornado.httpserver.HTTPServer(app)
        http_server.listen(options.port)
        tornado.ioloop.IOLoop.instance().start()

一个同事提示为什么这个耗时的东西不能异步的丢给某工具去执行而不阻塞我的请求呢?好吧，我也想到了:celery，正好github有这个东西:tornado-celery

执行下面的程序首先你要安装rabbitmq和celery:

    #!/bin/env python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.options
    import tornado.web
    import tornado.gen
    import tornado.httpclient
    import tcelery, tasks

    import time

    from tornado.options import define, options
    define("port", default=8000, help="run on the given port", type=int)

    tcelery.setup_nonblocking_producer()

    class SleepHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        @tornado.gen.coroutine
        def get(self):
            # tornado.gen.Task的参数是:要执行的函数, 参数
            yield tornado.gen.Task(tasks.sleep.apply_async, args=[5])
            self.write("when i sleep 5s")
            self.finish()

    class JustNowHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("i hope just now see you")

    if __name__ == "__main__":
        tornado.options.parse_command_line()
        app = tornado.web.Application(handlers=[
                (r"/sleep", SleepHandler), (r"/justnow", JustNowHandler)])
        http_server = tornado.httpserver.HTTPServer(app)
        http_server.listen(options.port)
        tornado.ioloop.IOLoop.instance().start()

task是celery的任务定义的文件，包含我们说的time.sleep的函数

    import time
    from celery import Celery

    celery = Celery("tasks", broker="amqp://guest:guest@localhost:5672")
    celery.conf.CELERY_RESULT_BACKEND = "amqp"

    @celery.task
    def sleep(seconds):
        time.sleep(float(seconds))
        return seconds

    if __name__ == "__main__":
        celery.start()

然后启动celelry worker(要不然你的任务怎么执行呢?肯定需要一个消费者取走):

    celery -A tasks worker --loglevel=info

但是这里的问题也可能很严重:我们的异步非阻塞依赖于celery,还是这个队列的长度,假如任务很多那么就需要等待,效率很低.有没有一种办法把我的同步阻塞函数变为异步(或者说被tornado的装饰器理解和识别)呢?

    #!/bin/env python

    import tornado.httpserver
    import tornado.ioloop
    import tornado.options
    import tornado.web
    import tornado.httpclient
    import tornado.gen
    from tornado.concurrent import run_on_executor
    # 这个并发库在python3自带在python2需要安装sudo pip install futures
    from concurrent.futures import ThreadPoolExecutor

    import time

    from tornado.options import define, options
    define("port", default=8000, help="run on the given port", type=int)

    class SleepHandler(tornado.web.RequestHandler):
        executor = ThreadPoolExecutor(2)
        @tornado.web.asynchronous
        @tornado.gen.coroutine
        def get(self):
            # 假如你执行的异步会返回值被继续调用可以这样(只是为了演示),否则直接yield就行
            res = yield self.sleep()
            self.write("when i sleep")
            self.finish()

        @run_on_executor
        def sleep(self):
            time.sleep(5)
            return 5

    class JustNowHandler(tornado.web.RequestHandler):
        def get(self):
            self.write("i hope just now see you")

    if __name__ == "__main__":
        tornado.options.parse_command_line()
        app = tornado.web.Application(handlers=[
                (r"/sleep", SleepHandler), (r"/justnow", JustNowHandler)])
        http_server = tornado.httpserver.HTTPServer(app)
        http_server.listen(options.port)
        tornado.ioloop.IOLoop.instance().start()

但是有朋友留言和我说为什么在浏览器打开多个url请求还是会阻塞一个个的响应呢?

这个事浏览器自身实现的可能是缓存把,当请求的资源相同就会出现这个问题,可以使用多浏览器(多人)或者命令行下的curl登都不会有这个问题,还有个比较恶的解决方法:

给你的请求添加一些无用参数，比如: http://localhost:8000/sleep/?a=1 也可以是个时间戳。
