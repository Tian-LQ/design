## 历史记录架构设计

#### 功能模块

------

针对于视频观看进度，视频浏览，我们需要一个历史查看的用户场景

- 添加记录，删除记录，清空历史
- 读取历史信息

历史记录场景，属于写极高，读高的业务场景，因此需要在设计时根据这一特点来做考量

#### 架构设计

------

**整体架构设计图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1.png?raw=true)

- history表示服务的接入层，一般处理业务相关的逻辑
- history-service是历史记录服务的主要服务层，处理核心逻辑
- history-job用于消费kafka当中的数据

**写场景下的设计时序图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E5%86%99%E5%9C%BA%E6%99%AF.png?raw=true)

**以下为写历史记录数据时的处理逻辑：**

- 先用户的读请求进入到history，进行一些业务性质的不同策略的处理，然后下发到history-service
- 首先实时的将历史数据信息写入redis，然后定时定量的聚合数据，发送至kafka，去异步的执行数据持久化逻辑
  - [x] 针对于同一个用户观看一个作品的历史记录信息，我们只需要保存最后一次数据即可，即last-write win
  - [x] 向kafka投递消息时，我们需要通过用户ID去做分区，这样才能保证同一个用户观看不同视频的历史数据信息的有序性
    - 由于仅仅通过用户ID去做聚合，整个消息的体量还是非常大的，因此引入了region sharding的方法
    - 我们将一组用户作为打包的单位，也就是说我们将用户ID % 100作为打包的要素，所有余数相同的放在一起，然后再去定时定量的发送给kafka，这样子将数据的收敛比又提高了100倍
    - 然后再根据用户ID % 100的余数，通过哈希算法来选择发送到kafka的哪个partition
  - [x] 这里发送给kafka的数据，仅仅是用户的ID和作品的ID这样的元数据信息，减少record的大小
- 与此同时history-job订阅了kafka当中的消息，再接收到历史记录持久化消息之后，由于record当中存储的是key值，因此再去redis当中返查到历史数据的完整信息，再批量的写入到HBase当中(HBase适合高密度写入场景)

**读场景下的设计时序图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E8%AF%BB%E5%9C%BA%E6%99%AF.png?raw=true)

**以下为读历史记录数据时的处理逻辑：**

- 对于读历史数据记录这里分为两个场景
- 获取历史作品流量记录列表(我的历史记录)
  - [x] 对于这类数据，定量的保存在redis当中，读请求会直接访问redis，如果出现数量不足的情况，则会直接访问hbase数据库，然后直接返回给用户
  - [x] 这里由于用户查看久远的历史记录的使用场景非常低频，因此不会回刷到redis当中
- 获取用户在当前作品下的历史观看进度记录
  - [x] 查找redis获取，如果存在则直接返回
  - [x] 如果redis当中没有，则直接返回没有，并不会去回查数据库，因为这类读场景触发非常高频，每当用户点开一个视频的时候，便会触发改场景，而绝大多数用户点开的视频都是新的视频，因此这样的视频是不存在历史数据的，而用户近期内观看的视频记录信息，又都保存在redis当中，因此无需做hbase的返查

[整个历史记录系统设计的核心在于write back的缓存模式使用，以及batch打包数据，减少IO次数的效果](https://github.com/Tian-LQ/design/blob/main/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E5%8E%86%E5%8F%B2%E8%AE%B0%E5%BD%95%E8%AF%BB%E5%9C%BA%E6%99%AF.png?raw=true)