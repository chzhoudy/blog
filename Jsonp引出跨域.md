## Jsonp引出跨域 ##

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

JSONP(JSON with Padding)是JSON的一种“使用模式”。因为HTML中的```<script>```元素是一个例外，利用```<script>```可以直接请求到跨域的资源，而jsonp的本质就是用带有callback函数名的URL去请求非同源资源，返回拼装好的带函数名的值，根据callback函数去处理。

例如：
     
    1.有一个非同源资源地址:www.a.com/get.do，用户期望返回JSON数据为["a":"aa","b":"bb]。

    2.我们在script标签里用带有jsonp参数的URL去请求。www.a.com/get.do?callback=fn

    3.经过服务端处理返回Tag:fn(["a":"aa","b":"bb])
   
    4.然后在页面上实现function fn(result){}接受处理result

> 服务端返回的不是JSON，而是带有回调函数名字的值，所以如果仍然直接以json去处理它，会弹出Uncaught SyntaxError: Unexpected token : 解决的方法很简单，服务端在拼装返回值的时候加上request.getParameter("callback")和"()"。

服务端处理如下（Java):
```
public void deal(HttpServletRequest request, HttpServletResponse response) {  
    String callback = (String)request.getParameter("callback");  
    String jsonData = "{\"a\":\"aa\", \"b\":"bb"}";  
    String retStr = callback + "(" + jsonData + ")";  
    response.getWriter().print(retStr);  
}  
```

平时经常使用的Jquery类框架同样支持JSONP，我们来看一下Zepto.js关于ajaxJSONP的代码：

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

**其他方法**

- proxy.jsp

在开发ArcGIS请求对方地图服务器的时候使用过proxy.jsp

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

- CORS(Cross-Origin Resource Sharing)

response.addHeader("Access-Control-Allow-Origin","*");

支持浏览器：chrome 3+,Firefox 3.5+,Opera 12+,Safari 4+,IE 8+

- nginx反向代理实现跨域

- iframe

- H5:postMessage

- H5:WebSockets



