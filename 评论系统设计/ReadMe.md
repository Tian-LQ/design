## 评论系统架构设计

#### 功能模块

------

针对不同作品(书籍、视频、漫画)，我们能够为其添加附加评论信息

- 发布评论
- 读取评论
- 删除评论
- 管理评论

评论属于读很多，写多的场景，因此在设计时需要根据这一特点来做考量

#### 架构设计

------

**整体架构设计图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1.png?raw=true)

- comment 表示评论服务的接入层，一般处理业务相关的逻辑，面向WEB端场景
- comment-service 真正的评论功能处理逻辑
- comment-job 用于处理消息队列当中的消息

**读场景下的设计时序图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F%E8%AF%BB%E5%9C%BA%E6%99%AF.png?raw=true)

**以下为读取评论时的处理逻辑：**

- 首先用户的读请求进入到comment，进行一些业务性质的不同策略的处理，然后下发到comment-service
- 在comment-service中，进行真正的评论读请求处理逻辑
  - [x] 先查询本地缓存，若存在则直接返回
  - [x] 若本地缓存miss，则查询redis缓存获取评论数据，若存在则直接返回
  - [x] 若redis缓存miss，则查询mysql数据库，查询到结果后直接返回
  - [x] 在数据库查询完成之后，通过消息队列kafka，异步解耦完成缓存重建的过程
    - 构造一条评论的基本元数据信息(唯一标识ID)记录，发送到kafka中
    - 随后由comment-job服务订阅kafka中的消息，完成消息消费，首先做去重幂等的逻辑，先去redis当中查找有无当前cache相关信息，若存在，则表示已经消费过了(cache rebuild finished)，丢弃这条消息直接返回。若不存在，则表示消息未消费，正常处理消费消息
    - 通过mysql数据库加载出评论完整数据信息，然后再回填到redis缓存中，完成最终的cache rebuild

**热点读场景，造成缓存穿透：**

为了应对热点作品下的评论会出现热点读场景，因此在comment-service中引入了singleflight模式，在热点读场景下，会存在大量的请求都请求相同的评论(资源)，此时会造成redis的读压力暴增，而且如果出现cache miss的情况，这部分压力还会下沉到mysql数据库层，因此引入了单飞这一模式，将相同的请求进行归并处理，这样一来只需查询一次即可让所有请求复用其结果，并且也只投递一条cache rebuild record到kafka上。

**写场景下的设计时序图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1/%E8%AF%84%E8%AE%BA%E7%B3%BB%E7%BB%9F%E5%86%99%E5%9C%BA%E6%99%AF.png?raw=true)

**以下为发布评论时的处理逻辑：**

- 由于可能出现热点作品下高tps的发布评论操作，因此发布评论这里也用到了kafka来实现消峰的效果
- 首先写请求进入到comment业务层，紧接着下发到comment-service
- 此时comment-service只发送一条写评论的record给kafka，便直接返回
- comment-job订阅kafka中的消息，在接收到消息后，完成写mysql以及cache build的操作

这里对于投递消息时，采用通过评论的作品关键信息来进行哈希分组：hash(comment_object) % N，其中N代表分区partitions的数量，这样一来相同作品下的评论都会投递到同一分区当中，comment-job内的consumer消费者实例便可以通过kafka的单分区消费的有序性，顺序的写入评论数据，从而保障业务的合理性。