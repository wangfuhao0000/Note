Sentinel（哨岗、哨兵）是Redis**高可用性**解决方案。由一个或多个Sentinel实例组成的Sentinel系统可监视任意多个主服务器及其属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器来代替已下线的主服务器继续处理命令请求。

**1. 启动并初始化Sentinel**

启动一个Sentinel可使用两个命令，且效果完全一样

```
$ redis-sentinel /path/to/your/sentinel.conf
$ redis-server /path/to/yout/sentinel.conf --sentinel
```

当一个Sentinel启动时，它需要执行以下步骤：

1. 初始化服务器
2. 将普通Redis服务器使用的代码替换成Sentinel专用代码
3. 初始化Sentinel状态
4. 根据给定的配置文件，初始化Sentinel的**监视主服务器列表**
5. 创建连向主服务器的网络连接

**1.1 初始化服务器**

因为Sentinel本质上只是一个运行在特殊模式下的Redis服务器，所以启动Sentinel的第一步就是初始化一个普通的Redis服务器，大致过程和初始化一个Redis服务器类似，不同的是初始化Sentinel时**不会**载入RDB文件或者AOF文件。

**1.2 使用Sentinel专用代码**

第二步就是将部分普通Redis服务器使用的代码替换成Sentinel专用代码。例如：

- **服务器端口：**普通Redis服务器使用REDIS_SERVERPORT常量，而Sentinel使用REDIS_SENTINEL_PORT常量；
- **服务器的命令表：**普通Redis使用redisCommandTable，而Sentinel使用sentinelcmds

**PING、SENTINEL、INFO、SUBSCRIBE、UNSUBSCRIBE、PSUBSCRIBE**和**PUNSUBSCRIBE**这七个命令就是客户端可对Sentinel执行的全部命令。

**1.3 初始化Sentinel状态**

应用了Sentinel专用代码后，服务器会初始化一个sentinelState结构（“Sentinel状态”），此结构保存了服务器中所有和Sentinel功能有关的状态，服务器的一般状态还是由redisServer结构保存：

```
struct sentinelState {
    //当前纪元，用于实现故障转移
    uint64_t current_epoch;
    //保存所有被这个sentinel监视的主服务器
    //字典键为主服务器名字，值为指向sentinelRedisInstance结构的指针
    dict* masters;
    //最后一次执行事件处理器的时间
    mstime_t prvious_time;
    //一个FIFO队列，包含所有需要执行的用户脚本
    list* scripts_queue;
} sentinel;
```

**1.4 初始化Sentinel状态的master属性**（监视的**主服务器**）

此属性记录了 所有被Sentinel监视的**主服务器**相关信息，其中：

- 字典键为被监视主服务器的名字
- 字典值为被监视主服务器对应的sentinelRedisInstance结构

每个**sentinelRedisInstance结构**代表一个被Sentinel监视的Redis服务器实例，此实例可以是主服务器、从服务器或另一个Sentinel。

**1.5 创建连向主服务器的网络连接**

初始化Sentinel的最后一步是创建连向被监视主服务器的网络连接，Sentinel将成为主服务器的客户端，它可以向主服务器发送命令，并从命令回复中获取相关信息。对每个被Sentinel监视的主服务器来说，Sentinel会创建两个连向主服务器的异步网络连接：

- 一个是**命令连接**：用于向主服务器发送命令，并接收命令回复。
- 另一个是**订阅连接**：用于订阅主服务器的__sentienl__:hello频道。创建此连接是因为Redis的发布与订阅功能中，被发送的信息都不会保存在Redis服务器里，若发送信息时客户端不在线就会丢失这条信息，因此为了不丢失__sentienl__:hello频道的任何信息，Sentinel必须专门用一个订阅连接来接受该频道的信息。

**2. 获取主服务器信息**

Sentinel默认以十秒一次的频率，通过**命令连接**向被监视的主服务器发送**INFO**命令，并通过分析**INFO**命令的回复来获取主服务器的当前信息。通过**INFO**命令回复，Sentinel可获取以下两方面信息：

- 关于主**服务器本身**的信息：包括run_id域记录的**服务器运行ID**，以及role域记录的**服务器角色**。
- 关于主服务器属下的所有**从服务器信息**，每个从服务器都由一个"slave"字符串开头的行记录（包括地址、端口等），根据这些信息就可以直接**自动发现从服务器**，而无需用户提供从服务器地址信息。

根据**INFO**命令返回的run_id域和role域的信息，Sentinel将对主服务器的实例结构进行更新，因为可能会出现自己结构中的运行ID和主服务器实际ID不一致。至于主服务器返回的从服务器信息，则被用于更新主服务器实例结构的**slaves**字典，此字典记录了主服务器属下从服务器的名单。Sentinel在分析**INFO**命令中包含的从服务器信息时会检查从服务器对应的实例结构是否已经存在于slaves字典中，若有则更新内容，没有则创建。

**3. 获取从服务器信息**

当Sentinel发现主服务器有新的从服务器出现时，Sentinel除了会为这个新的从服务器创建相应实例结构外，Sentinel还会创建连接到从服务器的**命令连接**和**订阅连接**。连接后，Sentinel默认会以十秒每次的频率通过命令连接向从服务器发送**INFO**命令，并获得回复。从回复中可提取的信息有：

- 从服务器运行ID run_id
- 从服务器角色role
- **主服务器的IP master_host和主服务器端口号mster_port**
- 主从服务器连接状态master_link_status
- 从服务器优先级slave+priority
- 从服务器的复制偏移量slave_repl_offset

根据这些信息，Sentinel会对从服务器的实例结构进行更新（**上面的slaves字典**）。

**4. 向主服务器和从服务器发送信息（命令连接）**

默认情况下，Sentinel每两秒一次，通过**命令连接**向所有被监视的主服务器和从服务器发送以下格式命令：

**PUBLISH __sentinel__:hello "<s_ip>, <s_port>, <s_runid>, <s_epoch>, <m_name>, <m_ip>, <m_port>, <m_epoch>"**

这条命令向服务器的__sentinel__:hello频道发送了一条信息，信息内容有多个参数构成：

- 以s_开头的是**Sentinel本身**的信息
- 以m_开头的记录的是**主服务器**的信息。如果Sentinel正在监视的是主服务器，则这些参数记录的就是主服务器的信息；若正在监视的是从服务器，则这些参数记录的就是从服务器**正在复制的主服务器**的信息。

| 参数    | 意义             | 参数    | 意义                 |
| ------- | ---------------- | ------- | -------------------- |
| s_ip    | Sentinel的IP地址 | m_name  | 主服务器名字         |
| s_port  | Sentinel的端口号 | m_ip    | 主服务器IP地址       |
| s_runid | Sentinel的运行ID | m_port  | 主服务器端口号       |
| s_epoch | 当前的配置纪元   | m_epoch | 主服务器当前配置纪元 |

**5. 接收来自主服务器和从服务器的频道信息（订阅连接）**

当Sentinel与一个主服务器或者从服务器建立起**订阅连接**后，Sentinel就会通过订阅连接向服务器发送以下命令：

**SUBSCRIBE __sentinel__:hello**

Sentinel对__sentinel__:hello频道的订阅会一直持续到Sentinel与服务器连接断开为止。对于监视同一个服务器的多个Sentinel来说，一个Sentinel发送的信息会被其他Sentinel接收到，这些信息会被用于更新其他Sentinel对发送信息Sentinel的认知，也会被用于更新其他Sentinel对被监视服务器的认知。

**5.1 更新Sentinels字典**

Sentinel为主服务器创建的实例结构中的sentinels字典保存了Sentinel本身，还保存了同样监视这个主服务器的其他Sentinel资料。所以当一个Sentinel接收到其他Sentinel发来的信息时，目标Sentinel会从中分析并提取一下两方面参数：

- 与Sentinel有关的参数
- 与监视的主服务器有关的参数

根据这两类参数分别更新对应的实例结构，其中对于Sentinel实例结构的更新包括创建一个新的结构。因此通过订阅连接，用户在使用Sentinel的时候并不需要提供各个Sentinel的地址信息，监视同一个主服务器的多个Sentinel可自动发现对方。

**5.2 创建连向其他Sentinel的命令连接**

当Sentinel通过**频道信息**发现一个新的Sentinel时，不仅会更新相应结构，还会创建一个连向新Sentinel的**命令连接**，而新Sentinel也同样会创建连向这个Sentinel的**命令连接**，最终监视同一个主服务器的多个Sentinel将形成相互连接的网络。而主观下线检测和客观下线检测就是使用Sentinel之间的命令连接来进行通信。

要注意的是Sentinel之间不会创建订阅连接，这是因为Sentinel需通过接收主服务器或者从服务器发来的频道消息来发现未知的新Sentinel，所以才需要订阅连接。而相互已知的Sentinel只需要命令连接来互相通信就行了。

**6. 检测主观下线状态（自己认为）**

默认Sentinel会以每秒一次的频率向所有与它创建了**命令连接**的实例（包括主服务器、从服务器、其他Sentinel在内）发送**PING**命令，并通过**PING**命令回复来判断实例是否在线。

- 若返回+PONG、-LOADING、-MASTERDOWN三种回复之一，则认为是有效回复
- 除了上面三种回复的其他回复都认为是无效回复

Sentinel配置文件中的down-after-milliseconds选项指定了Sentinel判断实例进入主观下线所需的时间长度：若一个实例在down-after-milliseconds毫秒内连续向Sentinel返回无效回复，则Sentinel会认为此实例已经进入主观下线状态。而且此选项是适用于所有发消息的实例，包括主服务器及其属下的从服务器和同样监视此主服务器的其他Sentinel。

需要注意，监视同一主服务器的多个Sentinel设置的主观下线时长可能不同，因此当一个Sentinel将主服务器判断为主观下线时，其他Sentinel可能仍然会认为主服务器处于在线状态。

**7. 检测客观下线状态（商量后认为）**

当Sentinel将一个主服务器判断为主观下线状态后，为确认此服务器是否真的下线了，会向同样监视此服务器的其他Sentinel进行询问，看它们是否也认为主服务器已下线。当Sentinel从其他Sentinel那里接受到**足够数量的已下线判断**之后，Sentinel就会将从服务器判定为客观下线，并对主服务器执行故障转移操作。

其中的通信使用命令：SENTINEL is-master-dowm-by-addr <ip> <port> <current_epoch> <runid>

同样要注意的是，对于监视同一个主服务器的多个Sentinel来说，他们将主服务器判断为客观下线的条件可能也不同：当一个Sentinel将主服务器判断为客观下线时，其他Sentinel可能并不是那么认为的。

**8. 选举领头Sentinel****（重要）**

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个Sentinel会进行协商，选举出一个**领头Sentinel**，并由领头Sentinel对下线主服务器执行故障转移操作。以下是Redis选举领头Sentinel的规则和方法：

- 监视同一主服务器的多个在线Sentinel都可能成为领头Sentinel
- 每次进行领头Sentinel选举后，不论成功失败，所有Sentinel的配置纪元的值都会自增一次。
- 在一个配置纪元里，所有Sentinel都有一次将某个Sentinel设置为局部领头Sentinel的机会，且局部领头一旦设置，在这个配置纪元里就不能再更改。
- 每个发现主服务器进入客观下线的Sentinel都要求其他Sentinel将自己设置为局部领头Sentinel
- 当一个Sentinel向另一个Sentinel发送SENTINEL is-master-down-by-addr命令，且命令参数不是*符号而是源Sentinel的运行ID时，这表示源Sentinel要求目标Sentinel将前者设置为后者的局部领头Sentinel
- Sentinel设置局部领头的规则是先到先得，即一个Sentinel投出选票后，就不能投其它的Sentinel了。
- 目标Sentinel接收到SENTINEL is-master-down-by-addr命令后，将向源Sentinel返回一条命令回复，回复中的leader_runid参数和leader_epoch参数分别记录了目标Sentinel的局部领头Sentinel的运行ID和配置纪元。
- 源Sentinel接收到目标Sentinel返回的命令回复后，会检查回复中leader_epoch参数的值和自己的配置纪元是否相同，若相通则源Sentinel继续取出回复中的leader_runid参数，若leader_runid参数的值和源Sentinel的运行ID一致，则表示目标Sentinel将源Sentinel设置成了局部领头Sentinel。
- 若有某个Sentinel被半数以上的Sentinel设置成了局部领头Sentinel，那它成为领头Sentinel。
- 因为领头Sentinel的产生需要半数以上Sentinel的支持，且每个Sentinel在每个配置纪元里只能设置一次局部领头Sentinel，所以在一个配置纪元里只会出现一个领头Sentinel
- 若在给定时限内没有一个Sentinel被选举为领头Sentinel，那么各个Sentinel将在一段时间后再次进行选举，直到选出领头Sentinel为止

**9. 故障转移**

选举出领头Sentinel后，领头Sentinel将对已下线的主服务器执行故障转移操作，包含以下三个步骤：

1. 在已下线主服务器属下的所有从服务器里面，挑选出一个从服务器，并将其转换为主服务器
2. 让已下线主服务器属下的所有从服务器改为复制新的主服务器
3. 将已下线主服务器设置为新的主服务器的从服务器，当次就的主服务器重新上线时会变为新的主服务器的从服务器。

**9.1 选出新的主服务器**

故障转移第一步就是从已下线主服务器属下的从服务器中选一个状态良好、数据完整的从服务器，然后向它发送**SLAVEOF no one**命令，将此从服务器转换为主服务器。**选择步骤**如下：

1. 领头Sentinel会将已下线主服务器的所有从服务器保存到一个列表里，然后按规则一一筛选。
2. 删除列表中所有已下线或断线的从服务器**（是在线的）**
3. 删除列表中所有最近五秒内没有回复过领头Sentinel的INFO命令的从服务器，保证剩余从服务器都是**最近成功通信过的**
4. 删除所有与已下线主服务器连接断开超过down-after-milliseconds * 10毫秒的从服务器，保证剩下从服务器没有过早与主服务器断开连接，**保存的数据是比较新的**
5. 之后，领头Sentinel根据从服务器优先级进行排序，选出**优先级最高**的从服务器。
6. 若有多个相同的最高优先级，则选出其中**偏移量最大的**从服务器（保证存储最新元素）
7. 如果还未选出，则按照运行ID进行排序，选出**运行ID最小的**从服务器

**9.2 修改从服务器的复制目标**

当新的主服务器出现后，领头Sentinel要让已下线主服务器属下的所有从服务器去复制新的主服务器，这一动作可通过向从服务器发送**SALVEOF**命令来实现。

**9.3 将旧的主服务器变为从服务器**

最后，将已下线的主服务器设置为新的主服务器的从服务器。因为主服务器已下线，所以此设置是保存在**主服务器实例结构**中的。当旧的主服务器上线后，Sentinel向它发送**SLAVEOF**命令，让它成为新的主服务器的从服务器。