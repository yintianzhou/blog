# CORS
> CORS（Cross-Origin Resource Sharing，跨域资源共》享）是W3C的一个工作草案，用于定义在必须进行跨域资源访问时，浏览器与服务器该如何沟通。  

这是《JavaScript高级程序设计》里对CORS的介绍，说白了，CORS就是一种规定，用来规定在跨域请求时服务器和浏览器如何沟通。既然是W3C的规定，那就很可能会有各大浏览器厂商的不同实现方式（前端摔！）。

事实上也确实如此，不过我们先来看一下CORS的具体工作原理，再看一下各大浏览器的具体实现。

- - - -

## CORS原理

> CORS背后的基本思想，就是使用自定义的HTTP头部让浏览器和服务器进行沟通，从而决定请求或者响应是应该成功还是失败。

浏览器将CORS请求分成两类：简单请求和非简单请求。
满足以下两大条件的请求就是简单请求，否则就是非简单请求。

> 1. 请求方法是以下三种方法之一：  
> - HEAD  
> - GET  
> - POST  
> 2. HTTP的头信息不超出以下几种字段：  
> - Accept  
> - Accept-Language  
> - Content-Language  
> - Last-Event-ID  
> - Content-Type：只限于三个值application_x-www-form-urlencoded、multipart_form-data、text/plain  

### 简单请求
对于简单请求，浏览器会自动在HTTP请求的头部添加一个_Origin_字段，该字段的值就是请求的源信息（协议、域名、端口）。
服务器接受到请求以后检测HTTP请求头是否携带Origin字段。
如果携带是否在可许范围内，如果认为该请求可以接受则在响应头部添加_Access-Control-Allow-Origin_字段，字段与请求的_Origin_字段相同（也可以回发*号，表示任意来源）。
浏览器接收到响应以后去解析这个头部字段，如果没有这个头部或者源信息不匹配，浏览器就会驳回请求。

### 非简单请求
对于非简单请求，浏览器会发送预检请求（Preflighted Requests）向服务器进行询问：允许的域名、允许的请求方法、允许的请求头。

Preflight请求发送下列头部
* Origin：源信息
* Access-Control-Request-Method：请求自身使用的方法
* Access-Control-Request-Headers：（可选）自定义请求头，多个头部以逗号分隔
栗子：
```
OPTIONS /cors HTTP/1.1
Origin: http://www.nczonline.net
Access-Control-Request-Method: POST
Access-Control-Request-Headers: X-Custom-Header
```

服务器接收到请求以后解析请求头，通过在响应头中发送以下头部信息与浏览器沟通：
* Access-Control-Allow-Origin：允许的请求源
* Access-Control-Allow-Methods：允许的方法，多个以逗号分隔
* Access-Control-Allow-Headers：允许的自定义头部字段，多个以逗号分隔
* Access-Control-Max-Age：预检请求结果缓存（以秒表示），在指定的时间内不用重复发送预检请求

### 带凭据的请求
默认情况下，跨域请求不提供凭据（cookie、HTTP认证及客户端SSL证明等）。
如果要发送带凭据的跨域请求，需要满足以下两个条件：
1. 在AJAX请求中将withCredentials属性设置为true：
```
var xhr = new XMLHttpRequest();
xhr.withCredentials = true;
```

2. 同时服务端也需要在响应头部加上_Access-Control-Allow-Credentials_字段：
`Access-Control-Allow-Credentials: true`

以上两个条件如果有一个不满足，带凭据的请求就不会发送成功。

ps：
> 如果要发送Cookie，Access-Control-Allow-Origin就不能设为星号，必须指定明确的、与请求网页一致的域名。同时，Cookie依然遵循同源政策，只有用服务器域名设置的Cookie才会上传，其他域名的Cookie并不会上传，且（跨源）原网页代码中的document.cookie也无法读取服务器域名下的Cookie。  

- - - -

## 浏览器对CORS的实现

### IE对CORS的实现
在IE8、IE9版本中，CORS通过XDR（XDomainRequest）对象实现。这个对象与XHR（XMLHttpRequest）类似。以下是两者的区别：
* cookie不会随请求发送，也不会随响应返回
* 只能设置请求头部信息中的Content-Type字段
* 不能访问响应头中的信息。
* 只支持GET和POST请求
Emmmmm…
从感觉上来说，这就是一个删减版的XHR，但是用它确实能进行跨域资源访问，并且也缓解了CSRF（Cross-Site Request Forgery，跨站点请求伪造）和XSS（Cross-Site Scripting，跨站点脚本）的问题。
但是使用起来确实不爽，有以下注意点：
```
var xdr = new XDomainRequest();
xdr.open("get", "http://www.somewhere.com/page");
xdr.send(null);
```

1. 请求返回后可以通过load事件获取响应数据，但是只能访问响应的原始文本，不能确定响应的状态码
```
xdr.onload = function () {
	console.log(xdr.responseText);
}
```

2. 在请求失败时会触发error事件，但是没有具体的错误信息
```
xdr.onerror = function () {
	console.log("Error!");
}
```

3. 只能创建异步请求

4. 可以设置请求超时（timeout）和超时处理方法（ontimeout）
```
xdr.timeout = 3000;
xdr.ontimeout = function () {
	console.log("Request took too long");
}
```

5. 只能通过contentType影响头部信息
```
xdr.contentType = "application/x-www-form-urlencodeed"
```

不过好在在IE10版本中被包含CORS的XHR取代了~~~

### 其他浏览器对CORS的实现
其他桌面浏览器和移动端平台浏览器都通过XMLHttpRequest对象实现CORS，并且在调用方法上和扑通XHR没有什么不同，在使用过程中基本感知不到。

- - - -

## 参考
《JavaScript高级程序设计》

[跨域资源共享 CORS 详解 - 阮一峰的网络日志](http://www.ruanyifeng.com/blog/2016/04/cors.html)