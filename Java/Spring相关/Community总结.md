## Service

### DiscussPostService

首先使用了Caffeine组件来作为缓存，会混存一些页面的数据，例如帖子的内容及帖子的数量等。然后在启动Spring时先将数据库中的一些数据放入到Caffeine中，这是通过一个带有`@PostCOnstruct`注解的`init()`方法实现的。

当需要取出某些数据时，查看一下是不是未登录的用户，如果是则直接使用Caffeine里面的数据即可。

```java
// 帖子列表缓存, String存储的是页码和偏移量，所以相当于会缓存多个页的数据
private LoadingCache<String, List<DiscussPost>> postListCache;

// 帖子总数缓存
private LoadingCache<Integer, Integer> postRowsCache;

public List<DiscussPost> findDiscussPosts(int userId, int offset, int limit, int orderMode) {
    if (userId == 0 && orderMode == 1) {   // 直接取缓存里面的
        return postListCache.get(offset + ":" + limit);
    }
    return discussPostMapper.selectDiscussPosts(userId, offset, limit, orderMode);
}

public int findDiscussPostRows(int userId) {
    if (userId == 0) {                   // 直接取缓存里面的
        return postRowsCache.get(userId);
    }
    return discussPostMapper.selectDiscussPostRows(userId);
}
```

#### PostConstruct注解

使用了注解`@PostConstruct`，注意这是在javax内的注解而不是Spring的。@PostConstruct该注解被用来修饰一个非静态的void（）方法。被@PostConstruct修饰的方法会在服务器加载Servlet的时候运行，并且只会被服务器执行一次。**PostConstruct在构造函数之后执行，init()方法之前执行**。通常我们会是在Spring框架中使用到@PostConstruct注解，该注解的方法在整个Bean初始化中的执行顺序：

> Constructor(构造方法) -> @Autowired(依赖注入) -> @PostConstruct(注释的方法)

#### addPost

当添加内容到数据库中时，需要过滤一些html标签和**敏感词**。html标签使用Spring里自带的工具即可，而敏感词我们自己可以手写一个。

### UserService

#### 注册

注册主要是验证要填的参数不能为空，以及用户名和邮箱不能与现有的用户冲突。验证通过后会将用户信息存储到数据库：

- 密码部分：随机生成一个UUID取后五位作为`salt`值，然后与用户密码进行拼接并用md5进行加密。将加密后的值作为密码存放到数据库中。

- `status`设置为0表示未激活。生成一个激活链接主要包括`userId`和`activationCode`，它们相当于形成了一个唯一的链接，发送给注册用户的邮箱中。只有两者都映射正确时，才能激活对应user的账户（status设置为1）。

#### getUser(userId)

查询用户信息的时候先尝试从Redis缓存里找，如果没有找到则从数据库中找并放到缓存里面。缓存的过期时间是一小时。

#### 激活

用户会存储一个字段：`activationCode`。注册的时候随机生成一个激活码存储起来，而且用户的状态字段`status`为0，并发送邮件给新注册用户。当进行激活操作时，用户点击那个激活链接，因为那个链接附带有`userId`和`activationCode`，只有这两个对应起来时才会成功激活。而且这个链接只有用户的邮箱才能收到，所以保证了安全性。

首先根据链接里面的`userId`从数据库中取出用户，并将链接中的`activateCode`和查询的用户的`activationCode`进行比较，相等说明链接是有效的，激活成功。

激活成功后将用户的`status`设置为1，这样就算激活用户了。

#### 登录

登录主要是**输入不能为空**，**保证用户存在**，**保证密码正确**。通过验证后，生成一个`LoginTicket`，这个凭证存储与登录有关的信息，包括`userId`，`ticket`等。其实就是相当于将这个`ticket`和一个用户的登录信息关联了起来。并将`<ticket, loginTicket>`存储到redis中，当查询用户信息时就可以根据用户传递过来的`ticket`来判断是不是这个用户，以及登录状态如何等。

#### 登出

登出也比较简单，也是根据用户传递过来的`ticket`，将它对应`loginTicket`中的`status`设置为1，表示已经登出了。所以有关用户的登录信息都放在了`loginTicket`中，而它又与用户传过来的`ticket`（通过cookie进行存储）关联到了一起。

### LikeService

#### 点赞

点赞这个行为是两方面的：

- 一个是用户被点赞的数量，当被点赞时用户的被点赞数量就+1，所以需要一个**`entityUserId`**。
- 一个是某个实体被该用户点赞了，我们用一个`Set`来存储给它点赞的人的id，`Set`对应的键为`<EntityType, EntityId>`。

当然如果扩展的话可以添加一个集合，存储点赞用户点赞过的实体。

#### 点赞状态

查看一个实体是否被当前用户点赞，可以从实体被点赞的用户id集合中查找是否包含当前用户的id。

## Controller

### LoginController

#### 验证码功能

首先使用google的kaptcha来生成**验证码文本`text`**和对应的**图像`image`**，然后生成一个`kaptchaOwner`，并将它的文本作为cookie传递给客户端，并将键值对`<kaptchaOwner, text>`存储到Redis中。这样当用户进行登录时会携带这个`kaptchaOwner`，然后找到Redis中对应的`text`和用户输入的验证码文本`inputText`来进行对比。

其实核心思想就是生成验证码后需要标识这个验证码属于谁，就把它作为Cookie发给某个客户端，这样就能知道当某个客户端传输过来的文本应该和哪个文本进行对比。即在服务端而不是前端进行校验。

将验证码输出到前端时需要先设置`ResponseType`为`/image/png`，案后获得`response.getOutputStream()`，最后进行写入。

```java
// 将图片输出给服务器
response.setContentType("/image/png");
try {
    OutputStream os = response.getOutputStream();
    ImageIO.write(image, "png", os);
} catch (IOException e) {
    logger.error("响应验证码失败： " + e.getMessage());
}
```

#### 登录

先对验证码进行校验，存储传过来的`inputCode`，根据传过来的`kaptchaOwner`取出后端对应的`code`进行对比，一致则继续检查账号和密码。

验证完毕后，就会将生成的`loginTicket`放到Cookie内，设置过期日期。这样当用户打开页面时从Cookie中传递过来`loginTicket`就能查到该用户的登录信息。

#### 登出

使用了以下语句：

```java
userService.logout(ticket);
SecurityContextHolder.clearContext();
```

后一句的目的是什么？？



## Util

### HostHolder

我们自己定义一个`HostHolder`用来代替session，内部封装的是一个`ThreadLocal<User>`。当用户登录完成后，就会将用户的信息放到这个里面，到时候需要当前登录用户信息的时候，只需要注入`HostHolder`并调用`getUser()`方法就能得到了。

