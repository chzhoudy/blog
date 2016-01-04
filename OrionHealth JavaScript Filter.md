

概述： javascript filter 通过执行脚本来处理通过 Rhapsody的信息，因为是脚本语言，可以轻松得进行复杂的计算和操作任务。

Filter 添加了三个全部对象：
  

	1. Input 一个Read-only Message 数组对象提供 input 消息给filter

	2. Output 一个Message collection对象，允许你使用另外一条消息作为模板，添加消息到 collection

	3. Log 提供Rhapsody 日志系统入口



# JavaScript Object Reference #

## 全局函数 ##

<table>
	<tr>
		<td>
			Function
		</td>
		<td>
			Description
		</td>
	</tr>
	<tr>
		<td>
			getCounter(counterName)
		</td>
		<td>
			根据指定的名字返回当前全局计数器，返回值一般为整形，如果不存在为0
		</td>
	</tr>
	<tr>
		<td>
			setCounter(counterName, int)
		</td>
		<td>
			根据指定名字和值设定全局计数器
		</td>
	</tr>
	<tr>
		<td>
			incCounter(counterName[, increment])
		</td>
		<td>
			根据指定的名字和值递增全局计数器
		</td>
	</tr>
	<tr>
		<td>
			getVariable(variableName[, decryptVariable])
		</td>
		<td>
			根据值返回Rhapsody环境变量，默认的decryptVariable为true
		</td>
	</tr>
	<tr>
		<td>
			substituteVariables(string [, decryptVariables])
		</td>
		<td>
			替换环境变量，例如MyVar值为75.My variable value is $(MyVar) 则返回My variable value is 75
		</td>
	</tr>
	<tr>
		<td>
			dataMap(input, file, separator, default)
		</td>
		<td>
			根据传入的input file创建datamap
		</td>
	</tr>
	<tr>
		<td>
			getLookupTable(string tableName)
		</td>
		<td>
			根据指定的表名查找表
		</td>
	</tr>
	<tr>
		<td>
			lookup(tableName, column1, value1 [,column2, value2, ...]) lookup(tableName, {column1: value1 [,column2: value2, ...]})
		</td>
		<td>
			根据指定的列名和值作为搜索标准查找表，返回列名作为属性值的javascript对象
		</td>
	</tr>
</table>

## dataMap 函数 ##

dataMap 从input端转化一个字符或者域到output端的文本文件传输表中

存在两种dataMap函数：
 

- 转化传入的字符串
- 直接根据域转化

传输输入的字符串：

``
output = dataMap(inputString, file)
output = dataMap(inputString, file, separator)
output = dataMap(inputString, file, separator, default)
output = dataMapColumns(inputString, inputColumn, outputColumn, file)
output = dataMapColumns(inputString, inputColumn, outputColumn, file, separator)
output = dataMapColumns(inputString, inputColumn, outputColumn, file, separator, default)
``

传入域路径：

``
message.dataMap(fieldPath, file)
message.dataMap(fieldPath, file, separator)
message.dataMap(fieldPath, file, separator, default)
message.dataMapColumns(fieldPath, inputColumn, outputColumn, file)
message.dataMapColumns(fieldPath, inputColumn, outputColumn, file, separator)
message.dataMapColumns(fieldPath, inputColumn, outputColumn, file, separator, default)
``

例如： 如果JavaScript filter需要转化城市缩写(AKL,WGTN,CHCH)到城市全名（Auckland , Wellington , Christchurch）

输入的消息为：
```
MSH|^~\&|||||||ZZZ|||
ZZZ|AKL| 
```

输出的信息为：

```
MSH|^~\&|||||||ZZZ|||
ZZZ|Auckland|
```

如果 C:\city-translations.txt 包含转化规则

```
AKL,Auckland,City of Sails
WGTN,Wellington,Capital City
CHCH,Christchurch,Garden City
%default%,unknown,unknown
```
3种转化方式：

```
for (var i = 0; i < input.length; i++) {
    var next = output.append(input[i]);

    next.dataMap("ZZZ/City", "C:\\city-translation.txt") ;
}
```

```
for (var i = 0; i < input.length; i++) {
    var next = output.append(input[i]);

    var cityAbbreviation = input[i].getField("ZZZ/City");
    var cityName =  dataMap(cityAbbreviation, "C:\\city-translation.txt") ;
    next.setProperty("city", cityName);
}
```

```
for (var i = 0; i < input.length; i++) {
    var next = output.append(input[i]);

    var cityAbbreviation = input[i].getField("ZZZ/City");
    var citySlogan = dataMapColumns(cityAbbreviation, 0, 2, "C:\\city-translation.txt");
    next.setProperty("citySlogan", citySlogan);
}
```









