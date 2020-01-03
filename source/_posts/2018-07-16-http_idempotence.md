---
layout: post
title: "HTTP幂等性"
tag: "http"
date: "2018-07-16 00:00:00 +0800"
categories: "技术"
---

<a href="https://www.cnblogs.com/weidagang2046/archive/2011/06/04/idempotence.html" target="_blank">原文</a>

HTTP协议是一种分布式的面向资源的网络应用层协议，Web API如此流行很大程度上应归功于简单有效的HTTP协议。然而，正如简单的Java语言并不意味着高质量的Java程序，
简单的HTTP协议也不意味着高质量的Web API。要想设计出高质量的Web API，还需要深入理解分布式系统及HTTP协议的特性。

<!--more-->

### http幂等性定义

http1.1规范中幂等性的定义是：

> Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of N > 0 identical 
> requests is the same as for a single request.  

HTTP方法的幂等性是指**一次和多次请求某一个资源应该具有同样的副作用**。幂等性属于语义范畴，正如编译器只能帮助检查语法错误一样，HTTP规范也没有办法通过消息格式等语法手段来定
义它，这可能是它不太受到重视的原因之一。但实际上，**幂等性是分布式系统设计中十分重要的概念**。

### 分布式事务 vs 幂等性

为什么需要幂等性？从一个账户充钱的远程API（可以是HTTP的也可以不是HTTP的）：

```
bool withdraw(account_id, amount)
```

withdraw是从账号account_id中扣除amount数额的钱，扣除成功则返回true，否则返回false账户余额不变。一种可能的情况是，服务器已经成功处理请求并返回，但是由于网络等原因可能
导致客户端并没有收到响应结果。如果是在网页上，一些不恰当的设计可能让用户认为上次操作失败然后刷新网页，这就会导致withdraw被调用两次，账户再次被扣钱。如图所示：
![](https://olef5l6y5.qnssl.com/20180716220334.png)

这个问题的解决方案一是采用分布式事务，通过引入支持分布式事务的中间件来保证withdraw功能的事务性。分布式事务的优点是对于调用者很简单，复杂性都交给了中间件来管理。缺点则是
一方面架构太重量级，容易被绑在特定的中间件上，不利于异构系统的集成；另一方面分布式事务虽然能保证事务的ACID性质，而但却无法提供性能和可用性的保证。

另一种更轻量级的解决方案是幂等设计：
```
int create_ticket() 
bool idempotent_withdraw(ticket_id, account_id, amount)
```

create_ticket的语义是获取一个服务器端生成的唯一的处理号ticket_id，它将用于标识后续的操作。idempotent_withdraw和withdraw的区别在于关联了一个ticket_id，一个ticket_id
表示的操作至多只会被处理一次，每次调用都将返回第一次调用时的处理结果。这样，idempotent_withdraw就符合幂等性了，客户端就可以放心地多次调用。

基于幂等性的解决方案中一个完整的取钱流程被分解成了两个步骤：1.调用create_ticket()获取ticket_id；2.调用idempotent_withdraw(ticket_id, account_id, amount)。虽然
create_ticket不是幂等的，但在这种设计下，它对系统状态的影响可以忽略，加上idempotent_withdraw是幂等的，所以任何一步由于网络等原因失败或超时，客户端都可以重试，直到获得
结果。
![](https://olef5l6y5.qnssl.com/20180717020852.png)

和分布式事务相比，**幂等设计的优势在于它的轻量级，容易适应异构环境，以及性能和可用性方面**。在某些性能要求比较高的应用，幂等设计往往是唯一的选择。

### HTTP的幂等性

HTTP GET方法用于获取资源，不应有副作用，所以是幂等的。

HTTP DELETE方法用于删除资源，有副作用，但它应该满足幂等性。比如：DELETE http://www.forum.com/article/4231 ，调用一次和N次对系统产生的副作用是相同的，即删掉id为4231
的帖子；因此，调用者可以多次调用或刷新页面而不必担心引入错误。

比较容易混淆的是HTTP POST和PUT。POST和PUT的区别容易被简单地误认为"POST表示创建资源，PUT表示更新资源"；而实际上，二者均可用于创建资源，更为本质的差别是在幂等性方面。

>The POST method is used to request that the origin server accept the entity enclosed in the request as a new subordinate of the resource identified 
>by the Request-URI in the Request-Line. …… If a resource has been created on the origin server, the response SHOULD be 201 (Created) and contain an 
>entity which describes the status of the request and refers to the new resource, and a Location header.
 
>The PUT method requests that the enclosed entity be stored under the supplied Request-URI. If the Request-URI refers to an already existing 
>resource, the enclosed entity SHOULD be considered as a modified version of the one residing on the origin server. If the Request-URI does not point 
>to an existing resource, and that URI is capable of being defined as a new resource by the requesting user agent, the origin server can create the 
>resource with that URI.

POST所对应的URI并非创建的资源本身，而是资源的接收者。比如：POST http://www.forum.com/articles的语义是在 http://www.forum.com/articles 下创建一篇帖子，
HTTP响应中应包含帖子的创建状态以及帖子的URI。两次相同的POST请求会在服务器端创建两份资源，它们具有不同的URI；所以，POST方法不具备幂等性。

而PUT所对应的URI是要创建或更新的资源本身。比如：PUT http://www.forum/articles/4231 的语义是创建或更新ID为4231的帖子。对同一URI进行多次PUT的副作用和一次PUT是相同的；
因此，PUT方法具有幂等性。


