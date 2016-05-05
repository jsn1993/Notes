<b>

restful是一种范式，不是一种协议。restful接口只是对http协议使用更加规范，回归了http协议本身之前的约定。原则上是对资源、集合、服务（URL），get、post、put、delete（操作）的合理使用。

get post delete put等各种Method的区别是语义上的。是为了让你的应用程序能够清晰简化的进行通信。对于HTTP本身来讲没有任何区别。当然安全性也没有任何区别。但有一些客观因素比如公司斥巨资买了一个防火墙，发现只能拦截过滤POST和GET请求，那就没办法了。

REST的目的是“建立十年内不会过时的软件系统架构"，所以它具备三个特点： 

1. 状态无关 —— 确保系统的横向拓展能力 

2. 超文本驱动，Fielding的原话是”hypertext-driven" —— 确保系统的演化能力 

3. 对 resource 相关的模型建立统一的原语，例如：uri、http的method定义等 —— 确保系统能够接纳多样而又标准的客户端 

优点：

REST 的应用可以充分地挖掘 HTTP 协议对缓存支持的能力。

从另外一个角度看，第一条保证服务端演化，第三条保证客户端演化，第二条保证应用本身的演化，这实在是一个极具抽象能力的方案。

关键原则：

一、资源: 为所有“事物”定义ID

在这里我使用了“事物”来代替更正式准确的术语“资源”，每个事物都应该是可标识的，都应该拥有一个明显的ID——在Web中，代表ID的统一概念是：URI。

二、超媒体被当作应用状态引擎: 将所有事物链接在一起

任何可能的情况下，使用链接指引可以被标识的事物（资源）。

三、使用标准方法

GET方法支持非常高效、成熟的缓存，所以在很多情况下，你甚至不需要向服务器发送请求。还可以肯定的是，GET方法具有幂等性[译注：指多个相同请求返回相同的结果]——如果你发送了一个GET请求没有得到结果，你可能不知道原因是请求未能到达目的地，还是响应在反馈的途中丢失了。幂等性保证了你可以简单地再发送一次请求解决问题。幂等性同样适用于PUT（基本的含义是“更新资源数据，如果资源不存在的话，则根据此URI创建一个新的资源”）和DELETE（你完全可以一遍又一遍地操作它，直到得出结论——删除不存在的东西没有任何问题）方法。POST方法，通常表示“创建一个新资源”，也能被用于调用任意过程，因而它既不安全也不具有幂等性。

    getUser
    addUser/{id}
    updateUser
    deleteUser

    /orders
        GET - list all orders
        PUT - unused
        POST - add a new order
        DELETE - unused
    /orders/{id}
        GET - get order details
        PUT - update order
        POST - add item
        DELETE - cancel order
    /customers
        GET - list all customers
        PUT - unused
        POST - add a new customer
        DELETE - unused
    /customers/{id}
        GET - get customer details
        PUT - update customer
        POST - unused
        DELETE - cancel customer
    /customers/{id}/orders
        GET - get all orders for customer
        PUT - unused
        POST - unused
        DELETE - cancel all customer orders

标识一个顾客的URI上的GET方法正好相当于getCustomerDetails操作。

四、资源多重表述

如果客户程序知道如何处理一种特定的数据格式，那就可以与所有提供这种表述格式的资源交互，包括XML、JSON、HTML等。即服务器端需要向外部提供多种格式的资源表述，供不同的客户端使用。比如移动应用可以使用XML或JSON和服务器端通信，而浏览器则能够理解HTML。

五、无状态通信

例如我订阅了一个人的博客，想要获取他发表的所有文章（这里『他发表的所有文章』就是一个资源Resource）。于是我就向他的服务发出请求，说『我要获取你发表的所有文章，最好是atom格式的』，这时候服务器向你返回了atom格式的文章列表第一页（这里『atom格式的文章列表』就是表征Representation）。你看到了第一页的页尾，想要看第二页，这时候有趣的事情就来了。如果服务器记录了应用的状态（stateful），那么你只要向服务询问『我要看下一页』，那么服务器自然就会返回第二页。类似的，如果你当前在第二页，想服务器请求『我要看下一页』，那就会得到第三页。但是REST的服务器恰恰是无状态的（stateless），服务器并没有保持你当前处于第几页，也就无法响应『下一页』这种具有状态性质的请求。因此客户端需要去维护当前应用的状态（application state），也就是『如何获取下一页资源』。当然，『下一页资源』的业务逻辑必然是由服务端来提供。服务器在文章列表的atom表征中加入一个URI超链接（hyper link），指向下一页文章列表对应的资源。客户端就可以使用统一接口（Uniform Interface）的方式，从这个URI中获取到他想要的下一页文章列表资源。上面的『能够进入下一页』就是应用的状态（State）。服务器把『能够进入下一页』这个状态以atom表征形式传输（Transfer）给客户端就是表征状态传输（REpresentational State Transfer）这个概念。

举个具体API的例子：

请求：
    GET /posts HTTP/1.1
    Accept: application/atom+xml

响应：
    HTTP/1.1 200 OK
    Content-Type: application/atom+xml

    <?xml version="1.0" encoding="utf-8"?>
    …
    <link href="http://example.org/posts" rel="self" />
    <link href="http://example.org/posts?pn=2" rel="next" />
    …
        <link href="http://example.org/post-xxx" />
    …
    </feed>

注意上面atom格式中的多个<link>元素，它们分别定义了当前状态下合法的状态转移。

例如，这是一个指向自己的链接，其中rel属性指定了状态转移的关系为自身。

    <link href="http://example.org/posts" rel="self" />

这是下一页的链接，

    <link href="http://example.org/posts?pn=2" rel="next" />

如果当前不是第一页的话，就会有类似如下的链接来表示上一页，

    <link href="http://example.org/posts?pn=2" rel="prev" />

而这个是某一篇文章的链接，

    <link href="http://example.org/post-xxx" />

总结一下，就是：

1、服务器生成包含状态转移的表征数据，用来响应客户端对于一个资源的请求；

2、客户端借助这份表征数据，记录了当前的应用状态以及对应可转移状态的方式。

当然，为了要实现这一系列的功能，一个不可或缺的东西就是超文本（hypertext）或者说超媒体类型（hypermedia type）。这绝对不是一个简简单单的媒体类型（例如，JSON属性列表）可以做到的。

普通的使用GET、POST返回JSON的这种API属于其第二层HTTP Verbs，RESTful的API属于第三层Hypermedia Controls。相比第二层，第三层的Web服务具有一个很明显的优势，客户端与服务端交互解耦。服务端可以仅仅提供单一的入口，客户端只要依次“遍历”超链接，就可以完成对应的合法业务逻辑。当资源的转换规则发送变化时（如某一页由于历史文章被删除了而没有下一页，又或者某篇文章转储在了其他网站上），客户端不需要作额外的更新升级，只需要升级服务端返回的超链接即可。

    {
      "links": {
        "self": "http://example.com/articles",
        "next": "http://example.com/articles?page[offset]=2",
        "last": "http://example.com/articles?page[offset]=10"
      },
      "data": [{
        "type": "articles",
        "id": "1",
        "attributes": {
          "title": "JSON API paints my bikeshed!"
        },
        "relationships": {
          "author": {
            "links": {
              "self": "http://example.com/articles/1/relationships/author",
              "related": "http://example.com/articles/1/author"
            },
            "data": { "type": "people", "id": "9" }
          },
          "comments": {
            "links": {
              "self": "http://example.com/articles/1/relationships/comments",
              "related": "http://example.com/articles/1/comments"
            },
            "data": [
              { "type": "comments", "id": "5" },
              { "type": "comments", "id": "12" }
            ]
          }
        },
        "links": {
          "self": "http://example.com/articles/1"
        }
      }],
      "included": [{
        "type": "people",
        "id": "9",
        "attributes": {
          "first-name": "Dan",
          "last-name": "Gebhardt",
          "twitter": "dgeb"
        },
        "links": {
          "self": "http://example.com/people/9"
        }
      }, {
        "type": "comments",
        "id": "5",
        "attributes": {
          "body": "First!"
        },
        "relationships": {
          "author": {
            "data": { "type": "people", "id": "2" }
          }
        },
        "links": {
          "self": "http://example.com/comments/5"
        }
      }, {
        "type": "comments",
        "id": "12",
        "attributes": {
          "body": "I like XML better"
        },
        "relationships": {
          "author": {
            "data": { "type": "people", "id": "9" }
          }
        },
        "links": {
          "self": "http://example.com/comments/12"
        }
      }],
      "msg" : "done", // 请求状态描述，调试用
      "code": 1001, // 业务自定义状态码
      "extra" : { // 全局附加数据，字段、内容不定
            "type": 1,
            "desc": "签到成功！"
        }
    }

自定义状态码

    // 授权相关
    1001: 无权限访问
    1002: access_token过期
    1003: unique_token无效
    // 用户相关
    2001: 未登录
    2002: 用户信息错误
    2003: 用户不存在
    // 业务1
    3001: 业务1XXX
    3002: 业务1XXX
