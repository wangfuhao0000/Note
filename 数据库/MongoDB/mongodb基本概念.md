## 概念

mongodb中基本的概念是**文档、集合、数据库**。

| SQL术语/概念 | MongoDB属于/概念 | 解释说明                             |
| ------------ | ---------------- | ------------------------------------ |
| database     | database         | 数据库                               |
| table        | collection       | 数据库表/集合                        |
| row          | document         | 数据记录行/文档                      |
| column       | field            | 数据字段/域                          |
| index        | index            | 索引                                 |
| table joins  |                  | 表连接，MongoDB不支持                |
| primary key  | primary key      | 主键，MongoDB自动将_id字段设置为主键 |

### 数据库

一个mongodb中可建立多个数据库，MongoDB的默认数据库为"db"，存储在data目录中。**`show dbs`**命令可现实所有数据库的列表，**`db`**命令可显示当前数据库对象，**`use`**命令可连接到一个指定的数据库。有一些数据库名是保留的，可以直接访问这些有特殊作用的数据库。

- **admin**： 从权限的角度来看，这是"root"数据库。要是将一个用户添加到这个数据库，这个用户自动继承所有数据库的权限。一些特定的服务器端命令也只能从这个数据库运行，比如列出所有的数据库或者关闭服务器。
- **local:** 这个数据永远不会被复制，可以用来存储限于本地单台服务器的任意集合
- **config**: 当Mongo用于分片设置时，config数据库在内部使用，用于保存分片的相关信息。

### 文档

**文档是一组键值对**，MongoDB的不同文档间不需要设置相同的字段，且相同的字段不需要相同的数据类型。一个简单的文档例子如下：

```json
{"site":"www.runoob.com", "name":"菜鸟教程"}
```

![img](https://www.runoob.com/wp-content/uploads/2013/10/Figure-1-Mapping-Table-to-Collection-1.png)

要注意的是：

1. 文档中的键/值对是有序的。
2. 文档中的值不仅可以是在双引号里面的字符串，还可以是其他几种数据类型（甚至可以是整个嵌入的文档)。
3. MongoDB区分类型和大小写。
4. MongoDB的文档不能有重复的键。
5. **文档的键是字符串**。除了少数例外情况，键可以使用任意UTF-8字符。

### 集合

集合就是MongoDB文档组，类似RDBMS中的表格。集合存在于数据库中，且没有固定结构。所以可以将不同格式的文档插入到同一个集合中。例如下面三个文档格式不同，也可插入到同一个集合中，第一个文档插入时，集合就会被创建。

```json
{"site":"www.baidu.com"}
{"site":"www.google.com","name":"Google"}
{"site":"www.runoob.com","name":"菜鸟教程","num":5}
```

#### capped collections

capped collections就是**固定大小**的集合，它有很高的性能以及**队列过期**的特性。它会自动的维护对象的插入顺序，非常适合类似记录日志的功能。和创建普通的collection不同，**创建capped collections需要显式地创建，指定collection的大小，单位是字节**，这样它的存储空间就会提前分配好。

Capped collections 可以按照文档的插入顺序保存到集合中，而且这些文档在磁盘上存放位置也是按照插入顺序来保存的，所以当我们更新Capped collections 中文档的时候，更新后的文档不可以超过之前文档的大小，这样话就可以确保所有文档在磁盘上的位置一直保持不变。也正是这种顺序插入，提高了添加数据的效率。

```powershell
db.createCollection("mycoll", {capped:true, size:100000})
```

注意的是指定的存储大小包含了数据库的头信息。且

- 在 capped collection 中，你能添加新的对象。
- 能进行更新，然而，**对象不会增加存储空间。如果增加，更新就会失败 **。
- 使用 Capped Collection **不能删除一个文档，可以使用 drop() 方法删除 collection 所有的行**。
- 删除之后，你必须显式的重新创建这个 collection。

### MongoDB数据类型

| 数据类型           | 数字    | 描述                                                         |
| ------------------ | ------- | ------------------------------------------------------------ |
| String             | 2       | 字符串，MongoDB中，UTF-8编码的字符串才是合法的。             |
| Integer            | 16/18   | 整型数值                                                     |
| Boolean            | 8       | 布尔值                                                       |
| Double             | 1       | 双精度浮点值                                                 |
| Min/Max keys       | 255/127 | 将一个值于BSON（二进制JSON）元素的最低值和最高值相对比       |
| Array              | 4       | 用于将数组或列表或多个值存储为一个键                         |
| Timestamp          | 17      | 时间戳，记录文档修改或添加的具体时间                         |
| Object             | 3       | 内嵌文档                                                     |
| Null               | 10      | 空值                                                         |
| Symbol             | 14      | 符号，基本上等同于字符串类型，但它一般用于特殊符号类型的语言 |
| Date               | 9       | 日期信息                                                     |
| Object ID          | 7       | 对象ID，用于创建文档的ID                                     |
| Binary Data        | 5       | 二进制数据                                                   |
| Code               | 13      | 代码类型，用于在文档中存储JavaScript代码                     |
| Regular expression | 11      | 正则表达式类型，存储正则表达式                               |

#### ObjectId

它类似唯一主键，可以很快地生成和排序，包含12bytes，含义是：

- 前4个字节标识创建的unix时间戳
- 接下来的3个字节是机器标识码
- 紧接着2个字节由进程id组成PID
- 最后三个字节是随机数

![img](https://www.runoob.com/wp-content/uploads/2013/10/2875754375-5a19268f0fd9b_articlex.jpeg)

MongoDB中存储的文档必须有一个_id键，这个键可以是任何类型的，默认是个ObjectId对象，且再ObjectId中已经保存了创建的时间戳，只需要通过getTimestamp函数就可获取文档的创建时间：

```javascript
> var newObject = ObjectId()
> newObject.getTimestamp()
ISODate("2020-07-02T01:59:17Z")
```

#### 字符串

BSON字符串都是UTF-8编码

#### 时间戳

BSON有一个特殊的时间戳类型用域MongoDB内部使用，它是一个64位的值，其中：

- 前32位是一个time_t值
- 后32为是再某秒中操作的一个递增的序数

单个mongodb实例中，时间戳通常唯一；在复制集中，oplog有一个ts字段，使用了BSON时间戳标识操作时间。

> BSON时间戳类型主要用于MongoDB内部使用，大部分的开发情况下，可以使用BSON日期类型。

#### 日期

标识当前距离Unix新纪元的毫秒数，日期类型是**有符号的**，负数标识1970年前的日期。

## 数据库操作

### 创建数据库

- MongDB使用`use`命令创建数据库，如果数据库不存在则进行创建，否则就切换到指定的数据库。

- 使用`db`命令可以查看当前使用的是哪一个数据库

- 使用`show dbs`命令可查看所有的数据库。要注意只使用`use`创建的数据库是不会显示的，我们必须向新建的netease数据库插入一些数据才行。

```powershell
> use netease
switched to db netease
> db
netease
> show dbs  # 没有创建的netease
admin   0.000GB
config  0.000GB
local   0.000GB
runoob  0.000GB
test    0.000GB
> db.collec1.insert({name:"网易"})  # 向netease数据库中的collec1集合插入一个文档
WriteResult({ "nInserted" : 1 })
> show dbs
admin    0.000GB
config   0.000GB
local    0.000GB
netease  0.000GB   # 出现了新建的数据库
runoob   0.000GB
test     0.000GB
```

> MongoDB中，集合只有在内容插入后才会创建。

### 删除数据库

- 删除数据库的语法如下，首先使用`use`切换到指定的数据库，然后执行`db.dropDatabase()`来进行删除。

```powershell
db.dropDatabase()

> show dbs
admin    0.000GB
config   0.000GB
local    0.000GB
netease  0.000GB
runoob   0.000GB
test     0.000GB
> use test   # 切换数据库
switched to db test
> db.dropDatabase()
{ "dropped" : "test", "ok" : 1 }
> show dbs
admin    0.000GB
config   0.000GB
local    0.000GB
netease  0.000GB
runoob   0.000GB
```

- 删除集合的语法如下，可以使用`show tables`或者`show collections`来查看所有的集合。

```powershell
db.collection.drop()

> use netease
switched to db netease
> show tables 
collec1
> db.collec1.drop()
true
> show tables
>
```

> 所以现在能够总结出来，当钱换到某个数据库后，`db.operations()`代表了对数据库的操作，而`db.collection.operations()`代表了对当前数据库中某个集合的操作。

### 创建集合

使用`createCollection()`方法来创建集合：

```
db.createCollection(name, options)
```

- name：要创建的集合名称
- options：可选参数，指定有关内存大小及索引的选项

options的可选值如下：

| 字段   | 类型 | 描述                                                         |
| ------ | ---- | ------------------------------------------------------------ |
| capped | 布尔 | 如果为true，则创建固定集合。**该值为true时，必须指定size参数。** |
| size   | 数值 | 为固定集合指定一个最大值，以KB为单位。                       |
| max    | 数值 | 指定**固定集合**中包含**文档**的最大数量                     |

> 插入文档时，MongoDB会先检查固定集合的size字段，然后检查max字段。

下面是创建集合的例子，如果直接插入一些文档，那么MongoDB会自动创建集合。

```powershell
> db.createCollection("collec1")
{ "ok" : 1 }
> db.createCollection("cappCollec",{capped:true, size:6142800, max:10000})
{ "ok" : 1 }
> db.autoCollec.insert({name:"auto"})
WriteResult({ "nInserted" : 1 })
> show collections
autoCollec
cappCollec
collec1
```

### 删除集合

MongoDB使用`drop()`方法来删除集合，如果成功删除选定集合，返回true，否则返回false。

```powershell
db.collection.drop()

> db.autoCollec.drop()
true
> show collections
cappCollec
collec1
```

### 插入文档

文档的数据结构和JSON基本一样，所有存储在集合中的数据都是BSON格式，它时一种二进制形式的存储格式。MongoDB使用`insert()`或`save()`方法向集合中插入文档，语法如下：

```powershell
db.COLLECTION_NAME.insert(document)
```

- `save()`：若_id主键存在则更新数据，不存在则插入。该方法在新版本中已废弃。
- 使用`insert()时`，若插入的数据主键已经存在，则会抛出异常提示主键重复，且不保存要插入的数据。

3.2 版本中新增了`insertOne()`和`insertMany()`两个方法，分别用于向集合中插入一个或多个文档。语法如下：

```powershell
db.collection.insertOne(
   <document>,
   {
      writeConcern: <document>
   }
)

db.collection.insertMany(
   [ <document 1> , <document 2>, ... ],
   {
      writeConcern: <document>,
      ordered: <boolean>
   }
)
```

- document：要写入的文档
- writeConcern：写入策略，默认为1即要求确认写操作，0是不要求
- ordered：指定是否**按顺序写入**，默认是true

### 更新文档

MongoDB使用`update()`和`save()`方法更新集合中的文档

#### update()方法

用域更新**已存在的文档**，语法如下：

```powershell
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
   }
)
```

- query：update的查询条件，类似sql中的where后面
- update：update的对象和一些更新的操作符（如$, $inc...），类似sql中的set后面的值
- upsert：可选，标识如果不存在update的记录，是否插入objNew，默认是false不插入
- multi：可选，默认是false即只更新找到的第一条记录，true表示把符合条件的全部记录进行更新
- writeConcern：可选，抛出异常的级别



```powershell
> db.collec1.update({title:'MongoDB教程'},{$set:{title:'MongoDB'}})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
> db.collec1.find().pretty()
{
        "_id" : ObjectId("5efd55a9b782b43a64f374e4"),
        "title" : "MongoDB",
        "by" : "runoob",
        "tags" : [
                "database",
                "NoSQL"
        ]
}
```

#### save()方法

save()方法通过传入的文档来替换已有文档，**_id主键存在就更新，不存在就插入**。语法如下：

```powershell
db.collection.save(
   <document>,
   {
     writeConcern: <document>
   }
)
```

### 删除文档

`remove()`函数用来移除集合中的数据，语法如下：

```powershell
db.collection.remove(
   <query>,
   <justOne>
)
# 2.6版本后的语法
db.collection.remove(
   <query>,
   {
     justOne: <boolean>,
     writeConcern: <document>
   }
)
```

- query：可选，删除的文档的条件
- justOneL可选，设为true或1则只删除一个文档，默认为false删除所有匹配条件的文档
- writeConcern：可选，抛出异常的级别

```powershell
> db.collec1.find()
{ "_id" : ObjectId("5efd55a9b782b43a64f374e4"), "title" : "MongoDB", "by" : "runoob", "tags" : [ "database", "NoSQL" ] }
{ "_id" : ObjectId("5efd5a8eb782b43a64f374e5"), "title" : "MongoDB教程", "by" : "www" }
> db.collec1.remove({title:"MongoDB教程"})
WriteResult({ "nRemoved" : 1 })
> db.collec1.find()
{ "_id" : ObjectId("5efd55a9b782b43a64f374e4"), "title" : "MongoDB", "by" : "runoob", "tags" : [ "database", "NoSQL" ] }
```

如果要删除所有的数据，可以使用下面的方式（类似SQL中的`truncate`命令）：

```powershell
db.collect1.remove({})
```

相当于我们的条件什么都没有，这样所有的文档都会符合条件，进而被删除。

### 查询文档

`find()`方法用来查询文档，并以非结构化的方式显示，当然可以在后面添加`pretty()`方法来使得结果更加易读。语法如下：

```powershell
db.collection.find(query, projection)
db.collection.find().pretty()  # 以易读的方式读取数据
```

- query：可选，使用查询操作符指定查询条件
- projection：可选，使用投影操作符指定返回的键，查询时返回文档中所有键值

还有一个`findOne()`对象，它只会返回一个文档。

#### MongoDB与RDBMS Where语句比较

| 操作       | 格式                     | 范例                                        | RDBMS中的类似语句       |
| :--------- | :----------------------- | :------------------------------------------ | :---------------------- |
| 等于       | `{<key>:<value>}`        | `db.col.find({"by":"菜鸟教程"}).pretty()`   | `where by = '菜鸟教程'` |
| 小于       | `{<key>:{$lt:<value>}}`  | `db.col.find({"likes":{$lt:50}}).pretty()`  | `where likes < 50`      |
| 小于或等于 | `{<key>:{$lte:<value>}}` | `db.col.find({"likes":{$lte:50}}).pretty()` | `where likes <= 50`     |
| 大于       | `{<key>:{$gt:<value>}}`  | `db.col.find({"likes":{$gt:50}}).pretty()`  | `where likes > 50`      |
| 大于或等于 | `{<key>:{$gte:<value>}}` | `db.col.find({"likes":{$gte:50}}).pretty()` | `where likes >= 50`     |
| 不等于     | `{<key>:{$ne:<value>}}`  | `db.col.find({"likes":{$ne:50}}).pretty()`  | `where likes != 50`     |

#### MongoDB AND条件

MongoDB的`find()`方法可传入多个键，每个键以逗号隔开，即常规SQL的AND条件。语法如下：

```powershell
db.col.find({key1:value1, key2:value2}).pretty()
```

代表的意思就是对于集合中的所有文档（行），查找有键key1对应值为value1且有键key2对应值为value2的文档，其中的键值对就类似于表中某一个属性是某个值，即`where key1=value1 AND key2=value2`。

> **注意当文档里面还包含一个对象时，可以进行嵌套**，例如：

```
db.inventory.find( { "size.uom": "in" } )
```

这个语句查询inventory集合中size字段内的uom字段等于in的文档。

#### AND和OR联合使用

```powershell
>db.col.find({"likes": {$gt:50}, $or: [{"by": "菜鸟教程"},{"title": "MongoDB 教程"}]}).pretty()
```

#### projection投影

如果想只返回指定的一些属性值，传递一个projection document给`find()`方法，而projection document的定义如下：

- `<field>: 1`：代表在返回的文档中要包含这个field
- `<field>: 0` ：代表在返回的文档中不要包含这个field

注意**_id**属性默认是返回的，不需要指定，如果不想返回此属性则将其设置为0。

```
db.inventory.find( {}, { _id: 0, item: 1, status: 1 } );
```

## 操作符

### 条件操作符

MongoDB中条件操作符包括：

- (>)大于-$gt
- (<)小于-$lt
- (>=)大于等于-$gte
- (<=)小于等于-$lte

假设我们向数据库中添加了下面三个文档：

```powershell
> db.col.insert({
...     title: 'PHP 教程',
...     description: 'PHP 是一种创建动态交互性站点的强有力的服务器端脚本语言。',
...     by: '菜鸟教程',
...     url: 'http://www.runoob.com',
...     tags: ['php'],
...     likes: 200
... })
WriteResult({ "nInserted" : 1 })
> db.col.insert({title: 'Java 教程',
...     description: 'Java 是由Sun Microsystems公司于1995年5月推出的高级程序设计语言。',
...     by: '菜鸟教程',
...     url: 'http://www.runoob.com',
...     tags: ['java'],
...     likes: 150
... })
WriteResult({ "nInserted" : 1 })
> db.col.insert({title: 'MongoDB 教程',
...     description: 'MongoDB 是一个 Nosql 数据库',
...     by: '菜鸟教程',
...     url: 'http://www.runoob.com',
...     tags: ['mongodb'],
...     likes: 100
... })
```

如果想要获取**"col"**集合中**"likes"**大于100的数据，可以使用下面命令：

```
db.col.find({likes:{$gt:100}})
```

如果是大于等于，则只需要将`$gt`改为`$gte`，而小于和小于等于是类似的，只是替换对应字母。



如果要获取符合多个条件的文档，例如想得到**"col"**集合中**"likes"**大于100，小于200的数据，可使用以下命令：

```
db.col.find({likes:{$lt:200, $gt:100}})})
```

### $type操作符

$type操作符是基于BSON类型来检查集合中匹配的数据类型，并返回结果。如果要获取**"col"**集合中title为String的数据，可以使用以下命令：

```
> db.col.find({title:{$type:2}})
```

### Limit与Skip方法

#### Limit方法

`limit()`方法接受一个数字参数，该参数指定从MongoDB中读取的记录条数，使用方法如下：

```powershell
db.COLLECTION_NAME.limit(NUMBER)

> db.col.find({}, {title:1, _id:0}).limit(2)  #使用了投影操作
{ "title" : "PHP 教程" }
{ "title" : "Java 教程" }
```

#### skip方法

使用`skip()`来跳过指定数量的数据，同样此方法接受一个数字参数。使用方法如下：

```powershell
db.COLLECTION_NAME.find().limit(NUMBER).skip(NUMBER)
```

> skip()方法默认参数是0

### Sort方法排序

使用`sort()`方法对数据进行排序，它可以通过参数指定排序的字段，并使用1和-1来指定排序的方式，1为升序、-1为降序。基本用法如下：

```powershell
db.COLLECTION_NAME.find().sort({KEY:1})
```

例如对col集合中的数据按字段likes的降序排列：

```powershell
> db.col.find({},{title:1,_id:0}).sort({likes:-1})
{ "title" : "PHP 教程" }
{ "title" : "Java 教程" }
{ "title" : "MongoDB 教程" }
```

> 注意当`skip()`、`limit()`和`sort()`三个放在一起时，执行的顺序是先`sort()`->`skip()`->`limit()`。

## 其他

### MongoDB索引

索引是特殊的数据结构，它存储在一个易于遍历读取的数据集合中，索引是对数据库表中一列或多列的值进行排序的一种结构。创建索引的方法是`createIndex()`方法，它的基本语法如下：

```
db.collection.createIndex(keys, options)

> db.col.createIndex({"title":1})
{
        "createdCollectionAutomatically" : false,
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "ok" : 1
}
```

当然在`createIndex()`方法中，可以设置使用多个字段创建索引：

```powershell
db.col1.createIndex({"title":1,"description":-1})
```

`createIndex()`方法的可选参数列表如下：

| Parameter          | Type          | Description                                                  |
| :----------------- | :------------ | :----------------------------------------------------------- |
| background         | Boolean       | 建索引过程会阻塞其它数据库操作，background可指定以后台方式创建索引，即增加 "background" 可选参数。 "background" 默认值为**false**。 |
| unique             | Boolean       | 建立的索引是否唯一。指定为true创建唯一索引。默认值为**false**. |
| name               | string        | 索引的名称。如果未指定，MongoDB的通过连接索引的字段名和排序顺序生成一个索引名称。 |
| dropDups           | Boolean       | **3.0+版本已废弃。**在建立唯一索引时是否删除重复记录,指定 true 创建唯一索引。默认值为 **false**. |
| sparse             | Boolean       | 对文档中不存在的字段数据不启用索引；这个参数需要特别注意，如果设置为true的话，在索引字段中不会查询出不包含对应字段的文档.。默认值为 **false**. |
| expireAfterSeconds | integer       | 指定一个以秒为单位的数值，完成 TTL设定，设定集合的生存时间。 |
| v                  | index version | 索引的版本号。默认的索引版本取决于mongod创建索引时运行的版本。 |
| weights            | document      | 索引权重值，数值在 1 到 99,999 之间，表示该索引相对于其他索引字段的得分权重。 |
| default_language   | string        | 对于文本索引，该参数决定了停用词及词干和词器的规则的列表。 默认为英语 |
| language_override  | string        | 对于文本索引，该参数指定了包含在文档中的字段名，语言覆盖默认的language，默认值为 language. |

通过在创建索引时添加`background:true`的选项，也可以在后台创建索引：

```
db.values.createIndex({open: 1, close: 1}, {background: true})
```

下面也列出了一些和索引相关的操作：

1. 查看集合索引

```
db.col.getIndexes()
```

2. 查看集合索引大小

```
db.col.totalIndexSize()
```

3. 删除集合所有索引

```
db.col.dropIndexes()
```

4. 删除集合指定索引

```
db.col.dropIndex("索引名称")
```

### 聚合

聚合主要用于处理数据（如统计平均值，求和等），并返回计算后的数据结果，和SQL中的count(*)有点类似。使用方法`aggregate()`来完成聚合操作，语法格式如下：

```
db.COLLECTION_NAME.aggregate(AGGREGAE_OPERATION)
```

