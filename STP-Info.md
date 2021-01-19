# STP-Info模块总结

## 一、数据库表结构

![alt data-collector-db](./image/stp-info-db.png)

## 二、该模块中相关的Eiffel Message

​		Stp-Info模块将从Message Bus中收集2种Stp配置变更相关的Eiffel Message，分别为StpCapabilityAddedMessage和StpCapabilityDeletedMessage。Stp-Info service收到这些Message后，将及时的更新缓存和数据库中对应的值，确保data-processed服务计算的正确性。

### 1. StpCapabilityAddedMessage

​		该类型的Message表示某个Stp增加了一个capability，其中的关键字段包括StpName、CapabilityKey以及CapabilityValue。

### 2. StpCapabilityDeletedMessage

​		该类型的Message表示某个Stp删除了一个capability，其中的关键字段包括StpName和CapabilityKey。

## 三、缓存的实现

​		Data-collector接收到的Job相关的Message的数量较多，每一个JobStartedMessage都要向Stp-Info服务发送StpName，Stp-Info如果没有这个Stp信息，就会请求TGF API，该API的返回速度较慢，会严重影响SUV处理Eiffel Message的效率，并且STP的信息（capalibity信息）很少改动，所以这里使用缓存，将STP的信息缓存起来，这样data-collector每次访问Stp-Info，只要访问缓存有数据，就返回成功，同理data-processed服务每次查询stp的capability信息，都会先访问缓存，提高处理的速度。Stp-Info服务监听Message Bus中关于STP配置变更的消息来更新缓存和数据库。

​		使用Redis的Hash结构存储STP的capability信息，Key为StpName，每一对Field和Value，分别为该STP的一对capability key/value。每一个Hash数据的过期时间设置为10分钟。

​		Redis使用一个简单的哨兵模式的集群结构，一个主节点，一个从节点，一个哨兵节点。

## 四、数据处理流程

![alt data-collector-db](./image/stp-info-flow.png)

### 1. 接收data-collector的请求

​		Data-Collector在处理JobStartedMessage时，会将StpName从消息体中取出，然后将StpName发送给Stp-Info服务处理，Stp-Info模块接收到StpName后，处理流程如下：（对应图中左侧的部分）

- 首先用stpName字段判断该Stp是否存在与缓存中，存在的话就返回成功，说明该Stp的信息已经保存过了；
- 缓存中不存在，就查询数据库中是否存在，数据库中存在的话，就将该Stp信息写入缓存中然后返回成功；
- 数据库中也不存在，说明目前没有该STP的信息，就去查询TGF的API获取该STP信息，然后写入数据库再写入缓存；

### 2. 接收Eiffel Message

​		STP的capability信息会被TGF更新，更新的信息以Eiffel Message的形式发送到Message Bus，Stp-Info服务会消费这些Eiffel Message，处理流程如下：（对应图中右侧的部分）

- 接收到Eiffel Message后，首先判断该STP信息是否存在于DB中，如果不存在则说明还没有使用到该STP，那么就可以暂时忽略该STP的更新，因为STP更新的Eiffel Message发送后，TGF API返回的值肯定已经被更新，所以等到有新的JobMessage被data-collector接收到时，就会去查询TGF API得到最新的STP信息。
- 如果该STP存在于DB中，则首先更新缓存，然后更新DB。（该顺序不能颠倒，否则会读取缓存中的脏数据）

### 3. 接收data-processed的请求

​		data-processed在计算每个时间段的数据时，会从data-collector和stp-info获取数据，从stp-info主要是获取STP的capability的信息，首先从缓存中读取，缓存中不存在就从DB种读取；如果DB中不存在，则说明data-collector或者stp-info在处理该STP对应的Job的数据时出现了异常。

## 五、该模块中解决的主要问题

### 1. 多个请求访问不存在的STP造成多线程重复查库和重复访问TGF API（缓存击穿）

​		Data-Collector服务携带StpName访问Stp-Info服务，如果缓存中不存在该STP信息，那么就会去查询数据库，如果数据库不存在就会查询TGF API。加入现在data-collector同时收到多个JobStartedMessage，其使用了同一个STP，并且该STP在STP-info服务的缓存和数据库中都不存在，那么这几个STP-info服务就会收到同一个STP Name的多次请求，多个线程会同时查询数据库，再同时查询TGF API。造成资源浪费，当这样的请求太多还会严重影响数据库和TGF API的性能。

​		解决方法就是，当Stp-info服务收到多个相同的STP请求，且缓存中不存在时，只能让一个请求去访问数据库再访问TGF API，然后该线程回写缓存后，再让其他线程重新从缓存中读取。具体的做法就是使用Redis的分布式锁，多请求访问，缓存中不存在，一个线程加锁，访问数据库，访问TGF API，回写缓存，然后解锁，其他线程再尝试访问缓存。

​		这里需要注意下面几点：

- 这个分布式锁不能阻塞其他STP的访问请求，所以不同的STP name的请求应该加不同的分布式锁对象。
- Redis分布式锁可能造成死锁问题（由于未及时解锁，或者解锁出现异常），所以应该给分布式锁添加合适的过期时间啊。
- Redis分布式锁加锁操作和设置过期时间操作，以及判断锁和解锁操作的原子性问题，应该使用Lua脚本，或者原子命令。
- Redis分布式锁的误解锁问题，给每一个分布式锁匹配一个和加锁线程相关的唯一ID，解锁时需要判断该唯一ID。
- Redis分布式锁的可重入性问题。

### 2.  处理stp name请求和处理capability更新不同步（没有先后顺序）

​		从TGF的角度来说，capability的更新动作只会在某个STP没有执行Job时发生，即要么发生在某个Job执行之后，要么发生在某个Job执行之前。同时，从TGF发出Eiffel Message到这些message被data-collector和stp-info接收的时延一般在2s~3s左右。

​		从stp-info服务的角度来说，接收到capability的更新信息后，要么该stp信息已经存在于stp-info的数据库中，要么该stp信息不存在。不存在的情况分两种，一种是该STP还没有执行Job，另一种是该STP的JobMessage正在路上或者正在被data-collector处理。

​		stp信息不存在的情况下统一不去处理该capability的变更。等到data-collector处理了该STP的job，向stp-info发送stp name，此时stp-info自然会去查询TGF API得到最新的capability信息；存在的情况下直接更新就可以。

### 3. 更新Stp capability缓存的方式

​		当stp-info接收到capability更新的Eiffel Message时，其更新数据库和缓存的流程为：先更新数据库，然后删除缓存中对应的capability信息。这个更新顺序是发生脏的概率最小的方式。其他方式会大概率出现读取缓存中脏数据的可能性：

​		先Update Cache再Update DB：可能造成DB和Cache数据不一致

![alt data-collector-db](./image/update-cache-update-db.png)

​		先Update DB再Update Cache：造成从Cache中读取到脏数据v1

![alt data-collector-db](./image/update-db-update-cache.png)

​		先Delete Cache再Update DB：造成从Cache中读取到脏数据v0

![alt data-collector-db](./image/delete-cache-update-db.png)