---
title: 常用的http code
date: 2019-03-27 19:18:57
categories: 计算机网络
tags: 
- http
---

### 1XX 信息响应

- 100 Continue
  这个临时响应表明，迄今为止的所有内容都是可行的，客户端应该继续请求，如果已经完成，则忽略它。
- 101 Switching Protocol
  该代码是响应客户端的 Upgrade 标头发送的，并且指示服务器也正在切换的协议。

### 2XX 成功响应

- 200 OK
- 201 Created
- 202 Accepted
  响应状态码 202 Accepted 表示服务器端已经收到请求消息，但是尚未进行处理。但是对于请求的处理确实无保证的，即稍后无法通过 HTTP 协议给客户端发送一个异步请求来告知其请求的处理结果。这个状态码被设计用来将请求交由另外一个进程或者服务器来进行处理，或者是对请求进行批处理的情形。
- 204 No Content
  服务器成功处理了请求，但不需要返回任何实体内容，并且希望返回更新了的元信息。响应可能通过实体头部的形式，返回新的或更新后的元信息。204响应被禁止包含任何消息体

### 3XX 重定向

- 300 Multiple Choice
  是一个用来表示重定向的响应状态码，表示该请求拥有多种可能的响应。用户代理或者用户自身应该从中选择一个。由于没有如何进行选择的标准方法，这个状态码极少使用。

假如服务器可以提供一个优先选择，那么它应该生成一个  Location 首部。

- 301 Moved Permanently
- 304 Not Modified
  在进行条件请求时,客户端会提供给服务器一个If-Modified-Since请求头,其值为服务器上次返回的Last-Modified响应头中的日期值,还会提供一个If-None-Match请求头,值为服务器上次返回的ETag响应头的值
  服务器会读取到这两个请求头中的值,判断出客户端缓存的资源是否是最新的,如果是的话,服务器就会返回HTTP/304 Not Modified响应,但没有响应体.客户端收到304响应后,就会从缓存中读取对应的资源.
  另一种情况是,如果服务器认为客户端缓存的资源已经过期了,那么服务器就会返回HTTP/200 OK响应,响应体就是该资源当前最新的内容.客户端收到200响应后,就会用新的响应体覆盖掉旧的缓存资源.
  只有在客户端缓存了对应资源且该资源的响应头中包含了Last-Modified或ETag的情况下,才可能发送条件请求.如果这两个头都不存在,则必须无条件(unconditionally)请求该资源,服务器也就必须返回完整的资源数据.

### 4XX  客户端错误

- 400 Bad Request
  1、语义有误，当前请求无法被服务器理解。除非进行修改，否则客户端不应该重复提交这个请求。
  2、请求参数有误。
- 401 Unauthorized
  指的是由于缺乏目标资源要求的身份验证凭证，发送的请求未得到满足。
  这个状态码会与   WWW-Authenticate 首部一起发送，其中包含有如何进行验证的信息
- 403 Forbidden
  服务器已经理解请求，但是拒绝执行它。与 401 响应不同的是，身份验证并不能提供任何帮助，而且这个请求也不应该被重复提交。
- 404 Not Found
  请求失败，请求所希望得到的资源未被在服务器上发现。没有信息能够告诉用户这个状况到底是暂时的还是永久的。404这个状态码被广泛应用于当服务器不想揭示到底为何请求被拒绝或者没有其他适合的响应可用的情况下。
- 405 Method Not Allowed
  请求行中指定的请求方法不能被用于请求相应的资源。该响应必须返回一个Allow 头信息用以表示出当前资源能够接受的请求方法的列表。 
- 408 Request Timeout
  请求超时。客户端没有在服务器预备等待的时间内完成一个请求的发送。客户端可以随时再次提交这一请求而无需进行任何更改。

### 5XX 服务器错误

- 500 Internal Server Error
  服务器遇到了不知道如何处理的情况。 通用错误码
- 501 Not Implemented 
  表示请求的方法不被服务器支持 服务器必须支持的方法（即不会返回这个状态码的方法）只有 GET 和 HEAD。
- 502 Bad Gateway
  此错误响应表明服务器作为网关需要得到一个处理这个请求的响应，但是得到一个错误的响应。(比如nginx没有从tomcat接受到正确的返回)
- 503 Service Unavailable
  服务器没有准备好处理请求。 常见原因是服务器因维护或重载而停机
- 504 Gateway Timeout

