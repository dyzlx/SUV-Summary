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

​		Data-collector接收到的Job相关的Message的数量较多，每一个JobStartedMessage都要向Stp-Info服务发送StpName，Stp-Info如果没有这个Stp信息，就会请求TGF API，该API的返回速度较慢，会严重影响SUV处理Eiffel Message的效率，并且STP的信息（capalibity信息）很少改动，所以这里使用缓存，将STP的信息缓存起来，这样data-collector每次方位Stp-Info，只要访问缓存有数据，就返回成功。Stp-Info服务监听Message Bus中关于STP配置变更的消息来更新缓存和数据库。

​		使用Redis的Hash结构存储STP的capability信息，Key为StpName，每一对Field和Value，分别为该STP的一对capability key/value。

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



