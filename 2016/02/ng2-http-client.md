# HTTP 客户端

> 讲述一个用 Angular Http 客户端的远程服务

[Http](https://tools.ietf.org/html/rfc2616)是浏览器/服务器通信最主要的协议。

> [WebSocket](https://tools.ietf.org/html/rfc6455) 协议是另一个重要的通信技术；我们在这一章中将不会涉及到。

Modern browsers support two HTTP-based APIs: XMLHttpRequest (XHR) and JSONP. A few browsers also support Fetch.

The Angular HTTP client library simplifies application programming of the XHR and JSONP APIs as we'll learn in this chapter covering:

Http client sample overview
Fetch data with http.get
RxJS Observable of HTTP Responses
Enabling RxJS Operators
Extract JSON data with RxJS map
Error handling
Log results to console
Send data to the server
Add headers
Promises instead of observables
JSONP
Set query string parameters
Debounce search term input
Appendix: the in-memory web api service

We illustrate these topics with code that you can run live in a browser.

### Http 客户端示例
