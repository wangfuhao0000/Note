### 一、创建并修改Lua环境

为了在Redis中执行Lua脚本，Redis在服务器中内嵌了一个Lua环境，并对这个Lua环境进行了一系列修改，从而确保这个Lua环境可以满足Redis服务器的需要。Redis服务器创建并修改Lua环境的整个过程由以下步骤组成：

1. 创建一个基础的Lua环境，之后的所有修改都是针对这个环境进行的
2. 载入多个函数库到Lua环境
3. 创建全局表格redis，这个表格包含了对Redis进行操作的函数，比如用于在Lua脚本中执行Redis命令的redis.call函数
4. 使用Redis自制的随机函数来替换Lua原有的带有副作用的随机函数，从而避免在脚本中引入副作用
5. 创建排序辅助函数，Lua环境使用这个辅佐函数来对一部分Redis命令的结果进行排序，从而消除这些命令的不确定性。
6. 创建redis.pcall函数的错误报告辅助函数，此函数可提供更详细的出错信息。
7. 对Lua环境中的全局环境进行保护，防止用户在执行Lua脚本的过程中，将额外的全局变量添加到Lua环境中。
8. 将完成修改的Lua环境保存到服务器状态的lua属性中，等待执行服务器传来的Lua脚本

**1. 创建Lua环境**

服务器首先调用Lua的C API函数lua_open，创建一个新的Lua环境。为了让这个Lua环境可以满足Redis的操作要求，接下来服务器将对这个Lua环境进行一系列修改

**2. 载入函数库**

修改环境的第一步就是将一些函数载入到Lua环境里：

- 基础库：包含Lua的核心函数，如assert、error等，为防止用户从外部文件中引入不安全代码，库中的loadfile函数会被删除
- 表格库：用于处理表格的通用函数，如table、concat等
- 字符串库：
- 数学库
- 测试库
- Lua CJSON库
- Struct库
- Lua cmsgpack库

通过这些功能强大的函数库，Lua脚本可直接对执行Redis命令获得的数据进行复杂的操作。

**3. 创建redis全局表格**

服务器在Lua环境中创建一个redis表格，并将它设为全局变量，此表格包含以下函数：

- 用于执行Redis命令的redis.call和redis.pcall函数
- 用于记录Redis日志的redis,log函数，以及相应的日志级别常量：redis.LOG_DEBUG、redis.LOG_VERBOSE。。。
- 用于计算SHA1校验和的redis.sha1hex函数
- 用于返回错误信息的redis.error_reply函数和redis.status_reply函数

其中最重要的是redis.call和redis.pcall函数，通过它们用户可直接在Lua脚本中执行Redis命令：

```
redis> EVAL "return redis.call('PING')" 0
```

**4. 使用Redis自制的随机函数来替换Lua原有的随机函数**

为保证相同的脚本可在不同的机器上产生相同的结果，Redis要求所有传入服务器的Lua脚本，以及Lua环境中的所有函数，都必须是无副作用的纯函数。Redis用自制的函数替换了原有的Math库中的math.random和math.randomseed函数。替换后的函数有两个特征：

- 对于相同的seed，math.random总产生相同的随机数序列，这个函数是一个纯函数
- 除非在脚本中使用math.randomseed显式地修改seed，否则每次运行脚本时Lua环境都使用固定的math.randomseed(0)语句初始化seed

**5. 创建排序辅助函数**

对于Lua脚本来说，另一个可能产生不一致数据的地方是那些带有不确定性质的命令，比如对于一个集合键来说，因为集合元素是无序的，所以即使两个集合的元素完全相同，他们的输出结果也可能不同。Redis将SMEMBERS这种在相同数据集上可能产生不同输出的命令称为“带有不确定性的命令”，它们包括：

- SINTER
- SUNION
- SDIFF
- SMEMBERS
- HKEYS
- HVALS
- KEYS

为了消除这些命令带来的不确定性，服务器会为Lua环境创建一个排序辅助函数__redis__compare_helper，当Lua脚本执行完一个带有不确定性的命令后，程序会使用此函数作为对比函数，自动调用table.sort函数对命令的返回值做一次排序，以此来保证相同的数据集总是产生相同的输出。

**6. 创建redis.pcall函数的错误报告辅助函数**

服务器为Lua环境创建一个名为__redis__err__handler的错误处理函数，当脚本调用redis.pcall函数执行Redis命令，并且被执行的命令出现错误时，此函数就会打印出错误代码的来源和发生错误的行数，为程序的调试提供方便。

**7. 保护Lua的全局环境**

服务器对Lua环境中的全局环境进行保护，确保传入服务器的脚步不会因为忘记使用local关键字而将额外的全局变量添加到Lua环境里面。因为全局变量保护的原因，当一个脚步试图创建一个全局变量时，服务器将报告一个错误；试图获取一个不存在的全局变量也会引发一个错误。但Redis并未禁止用户修改已存在的全局变量，所以要非常谨慎。

**8. 将Lua环境保存到服务器状态的lua属性里**

上面Redis服务器对Lua环境的修改完成了，最后一步服务器会将Lua环境和服务器状态的lua属性关联起来。因为Redis使用串行化的方式执行Redis命令，所以在任何特定时间里，最多都会只有一个脚本能够被放进Lua环境里运行，因此整个服务器只需要创建一个Lua环境即可。

### 二、Lua环境协作组件

Redis创建了两个用于与Lua环境进行协作的组件，它们分别负责执行Lua脚本中的Redis命令的伪客户端，以及用来保存Lua脚本的lua_scripts字典。

**1. 伪客户端**

因为执行Redis命令必须有相应的客户端状态，所以为了执行Lua脚本中包含的Redis命令，Redis服务器专门为Lua环境创建了一个客户端，并由这个伪客户端负责处理Lua脚本中包含的所有Redis命令。Lua脚本使用redis.call函数或者redis.pcall函数执行一个Redis命令：

1. Lua环境将redis.call函数或者redis.pcall函数想要执行的命令传给伪客户端
2. 伪客户端将脚本想要执行的命令传给命令执行器
3. 命令执行器执行完伪客户端传给它的命令，并将命令的执行结果返回给伪客户端
4. 伪客户端接收命令执行器返回的命令结果，并将这个命令结果返回给Lua环境
5. Lua环境接收到命令结果后，将该结果返回给redis.call或者redis.pcall函数
6. 接收到结果的redis.call或者redis.pcall函数会将命令结果作为函数返回值返回给脚本中的调用者。

![img](https://note.youdao.com/yws/public/resource/cde132b80b71352363b55f0fc83cd5ff/xmlnote/WEB3292c00da0662ce1c0026526ed137034/956931AE6C97460793296ECA9850ED11/24880)

**2. lua_scripts字典**

这个字典的键为某个Lua脚本的SHA1校验和，而字典的值为SHA1校验和对应的Lua脚本。Redis会将所有被EVAL命令执行过的Lua脚本，以及所有被SCRIPT LOAD命令载入过的Lua脚本都保存到lua_scripts字典里。

![img](https://note.youdao.com/yws/public/resource/cde132b80b71352363b55f0fc83cd5ff/xmlnote/WEB3292c00da0662ce1c0026526ed137034/EBE4F059699549F4A52E22E72E9AD7D3/24890)

次字典有两个作用，一个是实现SCRIPT EXISTS命令，另一个是实现脚本复制功能。

### 三、EVAL命令的实现

它的实现分为以下三个步骤：

1. 根据客户端给定的Lua脚本，在Lua环境中定义一个Lua函数
2. 将客户端给定的脚本保存到lua_scripts字典，等待将来进一步使用
3. 执行刚刚在Lua环境中定义的函数，以此来执行客户端给定的Lua脚本

我们以下面这个命令为例子，来看EVAL命令执行的三个步骤：

```
127.0.0.1:6379[10]> EVAL "return 'hello world'" 0
"hello world"
```

**1. 定义脚本函数**

当客户端向服务器发送EVAL命令，要求执行某个Lua脚本时，服务器首先要做的就是在Lua环境中，为传入的脚本定义一个与这个脚本对应的Lua函数，函数名由f_前缀加上脚本的SHA1校验和组成，函数的体是脚本本身。使用函数来保存客户端传入的脚本有以下好处：

- 执行脚本的步骤非常简单，只需要调用与脚本对应的函数即可
- 通过函数的局部性让Lua环境保持清洁，减少了垃圾回收的工作量，避免了使用全局变量
- 如果某个脚本对应的函数在Lua环境中被定义过至少一次，那么只要记住这个脚本的SHA1校验和，服务器就可以在不知道脚本本身的情况下，直接通过调用Lua函数来执行脚本，这是EVALSHA命令的实现原理

**2. 将脚本保存到lua_scripts字典**

将客户端传入的脚本保存到服务器的lua_scripts字典里，键为Lua脚本的SHA1校验和，值为Lua脚本本身

**3. 执行脚本函数**

在为脚本定义完函数并且存到字典中后，还需要进行一些设置钩子、传入参数之类的准备动作，才能正式开始执行脚本

![img](https://note.youdao.com/yws/public/resource/cde132b80b71352363b55f0fc83cd5ff/xmlnote/WEB3292c00da0662ce1c0026526ed137034/DE643CD463B34FAFAB329A15D78C6605/24925)

至此开头的实例命令执行完毕，之后服务器只要将保存在输出缓冲区里的结果返回给执行EVAL命令的客户端就可以了。

### 四、EVALSHA命令的实现

它就是根据传入的校验和，检查对应的函数（因为我们定义了这个函数）是否存在于Lua环境中，如果存在则执行函数并返回结果。

### 五、脚本管理命令的实现

除了EVAL和EVALSHA命令外，Redis中与Lua脚本有关的命令还有四个，分别是：SCRIPT FLUSH、 SCRIPT EXISTS、 SCRIPT LOAD、和 SCRIPT KILL

**1. SCRIPT FLUSH**