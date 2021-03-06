## IOS支付架构设计

#### 功能模块

------

对于IOS苹果端APP，新增支付功能

- 增加APP内余额充值功能
- 增加余额购买订单功能

用户可以先在APP内充值余额，然后再通过余额完成订单的创建以及支付环节

#### 架构设计

------

**余额充值时序图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/IOS%E6%94%AF%E4%BB%98%E8%AE%BE%E8%AE%A1/IOS%E4%BD%99%E9%A2%9D%E5%85%85%E5%80%BC%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true)

- 这里所说的商品列表是指的Apple内指定的固定充值数额的产品(6元、30元、128元、328元、648元)
- 首先前端调用ios内充值接口，充值完成后获得收据，然后调用后端的余额充值接口，后端通过调用appStore服务器提供方法，验证前端提供的收据信息是否真实有效(价格、订单号等)，通过该收据来做幂等，验证通过之后，便调用资产asset服务当中余额充值的接口，为当前用户增加ios余额

**IOS订单时序图：**

![RUNOOB 图标](https://github.com/Tian-LQ/design/blob/main/IOS%E6%94%AF%E4%BB%98%E8%AE%BE%E8%AE%A1/IOS%E8%AE%A2%E5%8D%95%E6%97%B6%E5%BA%8F%E5%9B%BE.png?raw=true)

- 整个IOS订单支付流程分为两个主流程
- 订单创建流程
  - [x] 首先用户根据所选择的课程订单信息，创建相应订单，此时会验证余额是否充足，库存是否充足
  - [x] 所有信息验证通过之后便会生成这条订单信息
- 订单支付流程
  - [x] 校验订单数据
  - [x] 以支付类型-订单号(IOS余额支付-订单号)作为幂等键，在asset服务ios表当中插入一条记录(唯一键)
  - [x] 若存在重复幂等键，则表示出现重复调用，直接返回错误给前端
  - [x] 若插入成功，则表示是第一次支付该订单，正常执行余额扣减流程，此时shop服务调用asset服务，完成余额扣减功能
  - [x] asset服务完成user_asset表内的余额扣减之后，发送余额扣减成功mq，异步通知shop服务，当前订单余额扣减已经成功，此时shop服务当中的消费者订阅这一消息，收到余额扣减成功的消息后，将订单支付状态改为已支付
  - [x] 与此同时shop服务发送一条开课mq，异步通知course服务为该用户开课
  - [x] course服务为当前用户开课成功之后将订单状态最终修改为已成功