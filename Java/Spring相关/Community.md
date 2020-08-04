

## entity

### Page

存储和分页有关的信息，其中的四个属性定义如下：

```java
//当前页码
private int current = 1;
// 每页的数据个数
private int limit = 10;
// 数据总数（用域计算页数）
private int rows;
// 查询路径（用于复用分页链接）
private String path;
```

### Comment

评论实体，注意一条评论**可以是针对某个问题，也可以是针对别人的评论进行回复**。所以评论实体有两个属性：`entityType`和`entityId`来表名这条评论针对的是哪一类的哪一个。

```java
private int id;
private int userId;
private int entityType;
private int entityId;
private int targetId;
private String content;
private int status;
private Date createTime;
```



### Event

要封装一个Event来向消息队列发送。

```java
private String topic;
private int userId;  // 消息的发送者
private int entityType;
private int entityId;
private int entityUserId;
private Map<String, Object> data = new HashMap<>();  // 一些元数据
```

### LoginTicket

当一个用户登录后需要记住这个用户的登录信息，也就是所谓的session。所以需要这样的一个实体来进行存储，其中的`status`表示登录信息，`expired`字段代表这个ticket的过期时间。而`ticket`字段则是实际内容，也就是用户需要携带的cookie里面的值。

```java
private int id;
private int userId;
private String ticket;
private int status;
private Date expired;
```

### Message

一条私信的内容，`fromId`和`toId`则能够唯一的标识两个人的会话，而`conversationId`则标识两个人会话中具体的某一条消息。

```java
private int id;
private int fromId;
private int toId;
private String conversationId;
private String content;
private int status;
private Date createTime;
```





## 其它

### CommunityConstant

首先有一些通用的字段，我们可以将它写到一个接口里。当某个类需要这些字段的时候，就让这个类实现此接口，这样就可以直接使用里面的字段。因为**接口中的所有字段都是隐式的public，static和final**。这样所有的类就共享这些变量了。