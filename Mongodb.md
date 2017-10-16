## Mongodb ##

**Init**


下载地址：http://dl.mongodb.org/dl/win32/x86_64

文档地址：https://docs.mongodb.com/

设置配置文件 mongod.cfg

<table>
	<tr>
		<td>
			设置
		</td>
		<td>
			描述
		</td>
	</tr>
    <tr>
		<td>
			dbpath
		</td>
		<td>
			设置数据文件目录
		</td>
	</tr>
	<tr>
		<td>
			verbose
		</td>
		<td>
			增加控制台日志的信息量，可用v vv vvv vvvv来提高详细等级。例如verbose=true vvv=ture
		</td>
	</tr>
	<tr>
		<td>
			logpath
		</td>
		<td>
			指定日志文件的位置和文件名
		</td>
	</tr>
	<tr>
		<td>
			logappend
		</td>
		<td>
			根据指定的名字和值递增全局计数器
		</td>
	</tr>
	<tr>
		<td>
			port
		</td>
		<td>
			TCP端口，监听客户端连接，默认27017
		</td>
	</tr>
	<tr>
		<td>
			bind_ip
		</td>
		<td>
			数据库将在这些地址上监听，默认所有端口。可以用逗号分隔开
		</td>
	</tr>
	<tr>
		<td>
			maxConns
		</td>
		<td>
			最多接受多少个连接
		</td>
	</tr>
	<tr>
		<td>
			auth
		</td>
		<td>
			如果true启用数据库身份证验证
		</td>
	</tr>
	<tr>
		<td>
			httpinterface
		</td>
		<td>
			禁用用于访问服务器状态和日志的HTTP接口，默认false
		</td>
	</tr>
    <tr>
		<td>
			rest
		</td>
		<td>
			数据库的简单REST接口，默认false，28017监控页面上无法使用 List all commands
		</td>
	</tr>
</table>

配置文件启动： > mongod --config mongod.cfg

监控页面：http://localhost:28017

**Java**

连接数据库：

```
MongoClient mongoClient = new MongoClient("localhost",27017); //使用指定+端口
MongoClientURI uri = new MongoClientURI("mongodb://rhit:rhit@localhost:27017/test"); //使用连接URI
MongoClient mongoClientUrl = new MongoClient(uri);
```

获取数据库对象：

```
MongoDatabase db = mongoClient.getDatabase("test"); 
```

访问和操作集合中的文档：

```
MongoCollection collection = db.getCollection("testCollection");
```

CRUD：

```
//批量插入
public static void insert(MongoCollection<Document> collection){
    List<Document> documents = new ArrayList<Document>();
    for (int i = 0; i < 10; i++) {
        documents.add(new Document("name", "dreamoftch").append("age", (20+i)).append("createdDate", new Date()));
    }
    collection.insertMany(documents);
}

//删除单条
public static void deleteCollection(MongoCollection<Document> collection){
    collection.deleteOne(Filters.lt("age",24));
}

//单条更新
public static void updateCollection(MongoCollection<Document> collection){
    collection.updateOne(Filters.eq("name","dreamoftch"),
            new Document("$set",new Document("name","new").append("age",16).append("createdDate", new Date())));
}

//查询集合
public static void listCollections(MongoDatabase database) {
    MongoIterable<String> allCollections = database.listCollectionNames();
    for (String collection : allCollections) {
        System.out.println("Collection name: " + collection);
    }
}

//过滤查询
public static void listCollectionWithFilter(MongoCollection<Document> collection){
    for (Document document : collection.find(Filters.and(Filters.eq("name", "dreamoftch"), Filters.gt("age", 25)))) {
        System.out.println(document);
    }
}

//批量查询
public static void listDocuments(MongoCollection<Document> collection){
     for(Document document : collection.find()){
         System.out.println(document);
     }
}

//过滤查询+反向排序
public static void listDocumentWithFilterAndInReverseOrder(MongoCollection<Document> collection){
    for (Document document : collection.find(
            Filters.and(Filters.eq("name", "dreamoftch"), Filters.gt("age", 26)))
                   .sort(Sorts.descending("age")))
            {
                 System.out.println(document);
            }
}
```

