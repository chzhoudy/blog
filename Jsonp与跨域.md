## Jsonp与跨域 ##

**JSONP和JSON**<br/>JSON(JavaScript Object Notation) 是当前最流行的数据交换格式。花括号，键值对具有非常高的可读性，与XML相比减少了复杂性。
```
{
    "people":[
        {"firstName":"Brett","lastName":"McLaughlin","email":"aaaa"},
        {"firstName":"Jason","lastName":"Hunter","email":"bbbb"},
        {"firstName":"Elliotte","lastName":"Harold","email":"cccc"}
    ]
}
```

我们这里不讨论JSON，提及它无非和JSONP一字之差。<br/>

JSONP(JSON with Padding)是JSON的一种“使用模式”。因为HTML中的```<script>```元素是一个例外，利用```<script>```可以直接请求到跨域的资源，而jsonp的本质就是用带有callback函数名的URL去请求非同源资源，返回值包上callback函数名，回调callback函数去处理。

例如：
     
    1.有一个非同源资源地址:www.a.com/get.do，用户期望返回JSON数据为["a":"aa","b":"bb]。

    2.我们在script标签里用带有jsonp参数的URL去请求。www.a.com/get.do?callback=fn

    3.经过服务端处理返回Tag:fn(["a":"aa","b":"bb])
   
    4.然后在页面上实现function fn(result){}接受处理result

> 服务端返回的不是JSON，而是外层包有回调函数名字的值，如果直接返回json，会弹出Uncaught SyntaxError: Unexpected token : 解决的方法很简单，服务端在拼装返回值的时候加上request.getParameter("callback")和"()"。

服务端处理如下（Java):
```
public void deal(HttpServletRequest request, HttpServletResponse response) {  
    String callback = (String)request.getParameter("callback");  
    String jsonData = "{\"a\":\"aa\", \"b\":"bb"}";  
    String retStr = callback + "(" + jsonData + ")";  
    response.getWriter().print(retStr);  
}  
```

常用框架同样支持JSONP，我们来看一下Zepto.js关于ajaxJSONP的代码：

```
$.ajaxJSONP = function(options){
    var callbackName = 'jsonp' + (++jsonpID),
      script = document.createElement('script'),
      abort = function(){
        $(script).remove()
        if (callbackName in window) window[callbackName] = empty
        ajaxComplete('abort', xhr, options)
      }
      ...
      window[callbackName] = function(data){
        clearTimeout(abortTimeout)
        $(script).remove()
        delete window[callbackName]
        ajaxSuccess(data, xhr, options)
      }
     serializeData(options)
     script.src = options.url.replace(/=\?/, '=' + callbackName)
     $('head').append(script)
     ...
}
```
可以发现，类似于Zepto,Jquery的jsonp均是创建一个script的标签，然后将src设置为要请求的url，其实现形式正是上面提到的JSONP本质。

当然，Zepto不推荐使用$.ajaxJSONP,仍然集成到了$.ajax方法中，作了分流:

```
if (dataType == 'jsonp' || hasPlaceholder) {
      if (!hasPlaceholder) settings.url = appendQuery(settings.url, 'callback=?')
      return $.ajaxJSONP(settings)
}
```
在$.ajax中启用jsonp，必须设置如下2个参数：

- jsonp (默认：“callback”): JSONP回调查询参数的名称
- jsonpCallback (默认： “jsonp{N}”): 全局JSONP回调函数的 字符串（或返回的一个函数）名。设置该项能启用浏览器的缓存

简单的说，前面一个参数名，后面一个参数值，如果jsonp:callback,jsonpCallback:fn,则URL为
?callback=fn

如果你想方便一点，直接返回已经预处理的数据，建议使用[jQuery-JSONP](https://github.com/jaubourg/jquery-jsonp)

```
$.jsonp({
  url: "http://service.org/path?param1=xx&param2=xx",
  callbackParameter: "callback",
  dataFilter:function(json){  
     return json;  
  },  
});
```

**跨域**<br/>JavaScript出于安全方面的考虑，不允许跨域调用其他页面的对象（同源策略）。跨域的域仅仅是通过URL的头部来识别，即window.location.protocol 加上 window.location.host。下表为常见的情况：

<table>
	<tr>
		<td>
			URL
		</td>
		<td>
			说明
		</td>
        <td>
			是否允许访问
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com/a.js<br/>http://www.a.com/b.js
		</td>
		<td>
			同一域名下
		</td>
        <td>
			允许
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com/lab/a.js<br/>http://www.a.com/script/b.js
		</td>
		<td>
			同一域名下不同文件夹
		</td>
        <td>
			允许
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com:8000/a.js<br/>http://www.a.com/b.js
		</td>
		<td>
			同一域名，不同端口
		</td>
        <td>
			不允许
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com/a.js<br/>https://www.a.com/a.js
		</td>
		<td>
			同一域名，不同协议
		</td>
        <td>
			不允许
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com/a.js<br/>http://172.30.91.75/b.js
		</td>
		<td>
			域名和对应的IP
		</td>
        <td>
			不允许
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com/a.js<br/>http://script.a.com/b.js
		</td>
		<td>
			主域相同，子域不同
		</td>
        <td>
			不允许
		</td>
	</tr>
<tr>
		<td>
			http://www.cnblogs.com/a.js<br/>http://www.a.com/b.js
		</td>
		<td>
			不同域名
		</td>
        <td>
			不允许
		</td>
	</tr>
    <tr>
		<td>
			http://www.a.com/a.js<br/>http://a.com/b.js
		</td>
		<td>
			同一域名，不同二级域名
		</td>
        <td>
			不允许
		</td>
	</tr>
</table>

**解决跨域的其他方法**

- CORS(Cross-Origin Resource Sharing)

1.配置在Java程序中: response.addHeader("Access-Control-Allow-Origin","*");

2.配置在nginx Server端 

```
location / {
     if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
        add_header 'Access-Control-Max-Age' 1728000;
        add_header 'Content-Type' 'text/plain charset=UTF-8';
        add_header 'Content-Length' 0;
        return 204;
     }
     if ($request_method = 'POST') {
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
     }
     if ($request_method = 'GET') {
        add_header 'Access-Control-Allow-Origin' '*';
        add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';
        add_header 'Access-Control-Allow-Headers' 'DNT,X-CustomHeader,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type';
     }
}
```

Tip: HTTP request_method 共有8种,参考[RFC2616](https://datatracker.ietf.org/doc/rfc2616/?include_text=1)：

OPTIONS:发送请求，根据请求的资源类型，确定服务器支持的功能 

GET: 根据请求url请求资源

HEAD：根据请求url获取header内容

POST: 提交body内容给url接受资源者，非幂等

PUT:与Post相似，都可以创建和更新资源，但与Post的区别为幂等

HTTP定义的幂等：

> Methods can also have the property of "idempotence" in that (aside from error or expiration issues) the side-effects of N > 0 identical requests is the same as for a single request.

所以PUT请求一次和多次结果一样，但是POST不一样。POST是操作集合资源（/resources），而PUT操作的是具体资源（/resources/R1）。如果URL可以在客户端确定，那么就使用PUT，如果是在服务端确定，那么就使用POST。
 
DELETE：提交url删除资源

TRACE：允许客户端查看从发起到接收整一条链上面的信息，服务器上启动Trace method，他人可以利用返回的详细信息，进行跨站脚本攻击（XSS）

CONNECT：根据url连接资源，例如上面的proxy.jsp中的connect方法

- proxy.jsp(在Java Application Server中，请求网页资源)
```
<%@page session="false" %>
<%@page import="java.net.*,java.io.*" %>
try {
	String reqUrl = request.getQueryString();
	URL url = new URL(reqUrl);
	HttpURLConnection con = (HttpURLConnection) url.openConnection();
	con.setDoOutput(true);
	con.setRequestMethod(request.getMethod());
	int length = request.getContentLength();
	if (length > 0) {
	    con.setDoInput(true);
	    byte[] idata = new byte[length];
	    request.getInputStream().read(idata, 0, length);
	    con.getOutputStream().write(idata, 0, length);
	}
	response.setContentType(con.getContentType());
	BufferedReader rd = new BufferedReader(new InputStreamReader(con.getInputStream()));
	String line;
	while ((line = rd.readLine()) != null) {
	    out.println(line);
	}
	rd.close();
	} catch (Exception e) {
	response.setStatus(500);
}
```

- Nginx Location Rewrite(反向代理法）

适用场景：对方服务器无法为你设置CORS,请求方前端又不能改变代码设置script标签，也无法使用jsonp,于是转由nginx服务器Rewrite地址并请求后返回结果。

场景：需要请求的url(172.23.123.49/api/getPerson?id=10088),nginx代理后请求地址(http://localhost/Kawasaki/getPerson?id=10088)

前台调用ajax:
```
$.ajax({ 
    type: "GET", 
    url:"/getPerson?id=10088", 
    success: function(data){..}  
})　　
```

nginx.conf:
```
location ^~/Kawasaki/ {
    rewrite ^/Kawasaki/(.*)$ /$1 break;
    proxy_pass http://172.23.123.49/api/;
}
```

Tip：location正则写法
```
location  = / {} 精确匹配 /,地址后面不能带任何分支
```

```
location  / {} 所有地址匹配
```

```
location ~ /url/ {}  匹配地址，匹配符合，继续搜索
```

```
location ^~ /url/ {} 匹配地址，匹配符合以后，停止往下搜索正则，采用这一条。
```

```
location ~* \.(gif|jpg|jpeg)$ {} 匹配所有以 gif,jpg或jpeg 结尾的静态资源请求
```

以下有一个参考自@seanlook的常用配置方案
```
location = / {
    proxy_pass http://tomcat:8080/index
}
location ^~ /static/ {
    root /webroot/static/;
}
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}
location / {
    proxy_pass http://tomcat:8080/
}
```

- iframe（反感，不讨论，仅列出）

<table>
	<tr>
		<td>
			方式
		</td>
		<td>
			思路
		</td>
        <td>
			场景
		</td>
	</tr>
    <tr>
		<td>
			location.hash
		</td>
		<td>
			父页面和跨域页面之间通过监听location.hash变化，并传值。由于iframe页面无法修改不同域的父页面hash值，需要在iframe页面中增加一个和父页面同域的iframe嵌套,以parent.parent.location.hash去修改
            
		</td>
        <td>
			允许传参暴露于url，类型大小受限
		</td>
	</tr>
    <tr>
		<td>
			document.domain
		</td>
		<td>
			父页面设置document.domain与被嵌套跨域页面一致，随后再嵌套子页面
		</td>
        <td>
			主域相同，子域不同
		</td>
	</tr>
    <tr>
		<td>
			window.name
		</td>
		<td>
			被嵌套页面有数据要传递，数据附加到window.name上，到和父页面同域的页面上，父页面就可以获取到内嵌的window.name
		</td>
        <td>
			window.name在页面跳转后依然存在，测试中发现Chrome无法获取，@xinsiyu2008出现部份属性IE10中获取不到相同，解决：删除嵌套iframe时设置的属性
		</td>
	</tr>
</table>

- H5:postMessage(未测试，待补）

- H5:WebSockets（未测试，待补）



