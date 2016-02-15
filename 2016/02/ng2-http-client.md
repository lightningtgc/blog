# HTTP 客户端

> 本文讲述一个用 Angular Http 客户端的远程服务

[Http](https://tools.ietf.org/html/rfc2616)是浏览器/服务器通信最主要的协议。

> [WebSocket](https://tools.ietf.org/html/rfc6455) 协议是另一个重要的通信技术；但我们在这一章中不会涉及到。

现代浏览器支持两种基于HTTP的API：[XMLHttpRequest (XHR)](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest) 和 [JSONP](https://en.wikipedia.org/wiki/JSONP)。少数浏览器也支持[Fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)。

Angular HTTP 客户端库简化了我们接下来将学到的`XHR`和`JSONP` API们的程序编码：
* Http 客户端概述
* 使用 http.get 获取数据
* HTTP 响应中得 RxJS Observable
* 使用 RxJS map 提取JSON数据
* 错误处理
* 打印结果日志
* 发送数据到服务器
* 添加 Headers
* 使用 Promises 代替 observables
* JSONP
* 设置查询字符串参数
* 搜索项输入防抖
* 附录：内存 web api 服务


Appendix: the in-memory web api service

我们通过代码来阐述这些主题，你可以[在浏览器中运行这些例子](https://angular.io/resources/live-examples/server-communication/ts/plnkr.html)。

### Http 客户端示例
