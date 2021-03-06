## Nashorn ##

**题记**： 在与 @SeeyouTomorrow 讨论React的使用场景时，他提出了在Java服务器端渲染React控件的构想，以提升初次加载的速度，实现同构，方便SEO。现在，Nashorn的出现让这一切成为了可能，我们以Germany Develper [winterbe](https://github.com/winterbe) DEMO作为范例。

**前言**：在JDK8，Oracle工程师upgrade JVM JavaScript解释器——Nashorn(German：犀牛)，取代了之前Rhino,依据JSR292完全支持ES5.1规范和扩展。

**[JSR292](https://jcp.org/en/jsr/detail?id=292)**：（JSR=Java Specification Requests) Supporting Dynamically Typed Languages on the Java Platform

功能点摘要：

- jjs 解释器可以在cmd运行JavaScript
- Java 代码中用 jdk.nashorn.api.scripting运行JavaScript
- 可以在JavaScript继承Java类，并实现接口，慎重使用，仍在变化
- JavaScript 中可以运行JavaFX程序

### 1.1 命令行运行Nashorn ###

你可以直接在命令行工具键入jjs,启动并执行javascript命令，jjs 是REPL模式，和Python REPL类似。

```
// Run as jjs < test.js
'Hello, World'.length
function factorial(n) { return n <= 1 ? 1 : n * factorial(n - 1) }
factorial(10)
var input = new java.util.Scanner(new java.net.URL('http://baidu.com').openStream())
input.useDelimiter('$')
var contents = input.next()
contents.substring(0, 1000)
```

Output:

```
 12
 3628800
 java.util.Scanner[delimiters=$][position=0][match valid=false][n
 put=false][source closed=false][skipped=false][group separator=\,][decimal
 ator=\.][positive prefix=][negative prefix=\Q-\E][positive suffix=][negati
 fix=][NaN string=\Q?\E][infinity string=\Q∞\E]
 <html><meta http-equiv="refresh" content="0;url=http://www.baidu.com/"></html>
```

### 1.2 Java运行Nashorn ###

jdk.nashorn.api.scripting.*

- NashornScriptEngine：JSR-223 标准下的脚本引擎。不会直接创建实例，通过 NashornScriptEngineFactory.getScriptEngine()创建。

javax.script.* 

- ScriptEngine：ScriptEngine 是基础接口，该接口的方法在此规范的每个实现中都必须具有完整的功能。
脚本执行使用 ScriptEngine 的 eval 方法和 Invocable 接口的方法。
- ScriptEngineManager：为 ScriptEngine 类实现一个发现和实例化机制，还维护一个键/值对集合来存储所有 Manager 创建的引擎所共享的状态。此类使用服务提供者机制枚举所有的 ScriptEngineFactory 实现。

初始化每一个js文件，加载至ThreadLocal中：

```
NashornScriptEngine nashornScriptEngine =
                    (NashornScriptEngine) new ScriptEngineManager()
                            .getEngineByName("nashorn");
try {
    nashornScriptEngine.eval(read("static/nashorn-polyfill.js"));
} catch (ScriptException e) {
    throw new RuntimeException(e);
}
return nashornScriptEngine;
```

对于方法的调用： 此处engineHolder 即ThreadLocal

```
Object html = engineHolder.get().invokeFunction("renderServer", comments);
```

p.s. ThreadLocal并非字面意思，应该定义为ThreadLocalVariable，它单独为每一个线程提供独立变量副本。包含4个基本方法：

- get():T  返回当前线程的局部变量副本  
- set(T):void  设置当前线程的局部变量值
- remove():void 删除当前线程局部变量（全部清空)
- initialValue():T 在set或get后执行，返回当前线程的初始化

### 1.3 用Script 操纵 Java###

- 访问类
  类型访问：
```
var ArrayList = Java.type("java.util.ArrayList");
var intType = Java.type("int");
var StringArrayType = Java.type("java.lang.String[]");
var int2DArrayType = Java.type("int[][]");
```
  返回实例：
```
var anArrayList = new Java.type("java.util.ArrayList");
```
  返回Object然后访问静态Field和方法
```
var File = Java.type("java.io.File");
File.createTempFile("nashorn", ".tmp");
```
- 导入包和类
```
importPackage(java.awt);
importClass(java.awt.Frame);
var frame = new java.awt.Frame("hello");
frame.setVisible(true);
```
  with的用法:
```
var Gui = new JavaImporter(java.awt, javax.swing);
with (Gui){
  var awtframe = new Frame("AWT Frame");
  var jframe = new JFrame("Swing JFrame");
}; 
```
- 使用数组
```
var anArray = [1, "13", false];
var javaIntArray = Java.to(anArray, "int[]");
print(javaIntArray[0]); // prints the number 1
print(javaIntArray[1]); // prints the number 13
print(javaIntArray[2]); // prints the number 0
接受js数组，你可以用Java.to()转化到java数组
// Convert the JavaScript array to a Java String[] array
var javaStringArray = Java.to(anArray, Java.type("java.lang.String[]"));
print(javaStringArray[0]); // prints the string "1"
print(javaStringArray[1]); // prints the string "13"
print(javaStringArray[2]); // prints the string "false"
接受java数组，你可以用Java.from()转化为js数组
var File = Java.type("java.io.File");
var listCurDir = new File(".").listFiles();
var jsList = Java.from(listCurDir);
print(jsList);
```
- 实现接口
```
var r  = new java.lang.Runnable() {
    run: function() {
        print("running...\n");
    }};
var th = new java.lang.Thread(r);
th.start();
th.join();
```
- 继承抽象类
```
var TimerTask =  Java.type("java.util.TimerTask");
var task = new TimerTask({ run: function() { print("Hello World!") } });
```
- 继承父类的方法
使用Java.super()函数来访问父类的方法
```
var Exception = Java.type("java.lang.Exception");
var ExceptionAdapter = Java.extend(Exception);
var exception = new ExceptionAdapter("My Exception Message") {
    getMessage: function() {
        var _super_ = Java.super(exception);
        return _super_.getMessage().toUpperCase();
    }}
``` 

浅尝 Nashorn ，下面我们来分析Java服务器端渲染场景：

因为搜索引擎的爬虫依赖的是服务端响应，服务端渲染不可或缺。除此之外，可以提升站点的性能，因为在加载js脚本的同时，浏览器就可以进行页面渲染。
React适合服务器端渲染的原因：
- 每一个组件在虚拟DOM中完成渲染，再更新浏览器DOM变化的部分
- React提供了服务器端渲染的函数:React.renderToString 和 React.renderToStaticMarkup

我们来看看官网对于这两个方法的说明：

*React.renderToString*：Render a ReactElement to its initial HTML. This should only be used on the server. React will return an HTML string. You can use this method to generate HTML on the server and send the markup down on the initial request for faster page loads and to allow search engines to crawl your pages for SEO purposes.If you call React.render() on a node that already has this server-rendered markup, React will preserve it and only attach event handlers, allowing you to have a very performant first-load experience.

该函数是一个同步阻塞函数，返回一个字符串。在其中你无法使用render方法，因为服务器端不存在DOM。React提供使用了data-react-id给此函数来区分DOM节点，这也是每当组件的state及props发生变化时，React都可以更新指定DOM节点

*React.renderToStaticMarkup*:Similar to renderToString, except this doesn't create extra DOM attributes such as data-react-id, that React uses internally. This is useful if you want to use React as a simple static page generator, as stripping away the extra attributes can save lots of bytes.

与上面不同之处仅在没有data属性，适合生成静态页面，例如：生成打印预览、转化模板页。

总结：当且仅当不打算在客户端渲染这个组件的时候，才应该选择 *React.renderToStaticMarkup*

### 2.1 Building Isomorphic Webapps on the JVM with React.js and Spring Boot ###

- Client-side rendering with React

renderClient中预处理 JSON data 

```
//commentBox.js
var renderClient = function (data) {
    React.render(
        <CommentBox data={data} url='comments.json' pollInterval={5000} />,
        document.getElementById("content")
    );
};
```
React在页面预加载的时候客户端初始化组件
```
//index.jsp
$(function () {
    renderClient(${data});
});
```

- Server-side rendering with Nashorn

用 *ScriptEngineManager* 创建Nashorn engine

```
//React.java
NashornScriptEngine nashornScriptEngine =
 (NashornScriptEngine) new ScriptEngineManager().getEngineByName("nashorn");
nashornScriptEngine.eval(read("static/nashorn-polyfill.js"));
nashornScriptEngine.eval(read("static/vendor/react.js"));
nashornScriptEngine.eval(read("static/vendor/showdown.min.js"));
nashornScriptEngine.eval(read("static/commentBox.js"));
```

用Java.from()转化Java数组,renderToString 生成描绘view的html字符串

```
//commentBox.js
var renderServer = function (comments) {
    var data = Java.from(comments);
    return React.renderToString(
        React.createElement(CommentBox, {
            data: data, 
            url: "comments.json", 
            pollInterval: 5000
        })
    );
};
```

- MainController

```
@Autowired
public MainController(CommentService service) {
    this.service = service;
    this.react = new React();
    this.mapper = new ObjectMapper();
}
```

初始化JSON,服务器端预渲染输出字符串
```
@RequestMapping("/")
public String index(Map<String, Object> model) throws Exception {
    List<Comment> comments = service.getComments();
    String commentBox = react.renderCommentBox(comments);
    String data = mapper.writeValueAsString(comments);
    model.put("content", commentBox);
    model.put("data", data);
    return "index";
}
```

在初始化JSON的同时，在ThreadLocal中调用renderServer方法，进行服务端渲染
```
Object html = engineHolder.get().invokeFunction("renderServer", comments);
```

组件在loadCommentsFromServer中方法中，接受服务器端的数据，更新state
```
loadCommentsFromServer: function () {
    $.ajax({
        url: this.props.url,
        dataType: 'json',
        success: function (data) {
            this.setState({data: data});
        }.bind(this),
        error: function (xhr, status, err) {
            console.error(this.props.url, status, err.toString());
        }.bind(this)
    });
}
```

**后记**：Nashorn让Java服务器端渲染React成为可能，Nashorn的冰山一角也让我们看到了Oracle工程师在设计时的用心。Facebook将React精简到view层，监听数据流的变化，避免了不必要的DOM操作，按需更新组件。而React设计哲学将用户界面看做组件状态的集合，根据状态决定界面的呈现，这就简化了我们思考方式：对界面的管理，即对组件状态的管理。组件包含了state(状态)和props(属性),state更多的是一个组件内部去维护，而props则是组件需要管理的数据。在多个组件组合呈现的页面上考虑父子组件状态的管理尤为重要，高内聚低耦合在设计时依然重要。两把利刃淬火成钢，尝试不同组合，摸索出最具威力的配合。
