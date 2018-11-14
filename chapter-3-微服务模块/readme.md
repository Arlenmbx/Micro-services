## 服务发布和引用
常见的有3类

| 描述方式 | 应用场景| 特点|
|------|------------|-----|
|restful api|跨语言平台，组织内外皆可|较为通用，性能差（底层tcp协议导致）|
|xml|通常用于java，内部使用|不支持跨语言，如做变更，消费者都需更新xml文件|
|IDL(grpc,thrift)|跨语言、性能好|修改pb 文件不能做到向前兼容|

 
 定义proto文件后，可以通过proto生成任意语言的客户端和服务端代码
 ```
 // The greeter service definition.
 service Greeter {
   // Sends a greeting
   rpc SayHello (HelloRequest) returns (HelloReply) {}
   rpc SayHelloAgain (HelloRequest) returns (HelloReply) {}
 
 }
 
 // The request message containing the user's name.
 message HelloRequest {
   string name = 1;
 }
 
 // The response message containing the greetings
 message HelloReply {
   string message = 1;
 }  

 ``` 
 

 
 ## 服务的注册与发现
 
 微服务架构通常需要三种角色，服务消费者、服务生产者和注册中心；
 
 ### 注册中心接口
 为了适应服务注册与发现的需求，注册中心需要提供以下接口
   * 注册服务
   * 反注册服务
   * 服务查询
   * 服务修改
   * 订阅接口
   * 变更查询接口
   * 心跳汇报接口：用于保证服务端存活状态
   
 ### 注册中心部署
 为了避免单点问题，需要做到集群部署，如 zookeeper 多点部署，各 node 通过paxos协议保持一致性原则；
 
 ### 注册中心数据存储结构
 
   * 通常为目录结构，每一个目录包含子目录和数据，在 zookeeper 中每一个目录称为一个 znode；
   * 数据可以做到多版本，比如一个数据的多个版本可以存放在多个节点/某个节点下的子节点；

### health check

客户端定时ping 注册中心，建立一个会话，同时生成一个 sessionid，并不断更新 session timeout,当timeout时间内，注册中心没有收到客户端的请求便认为服务失联，将数据从目录结构中删除；


### 服务变更
异步 watch 机制，当发现注册中心数据更新时，会同步数据更新本地 cache；

### 白名单机制

通常服务方的服务会有多套配置，如测试半，正式版 v1.0/v2.0。。。为了避免用户操作失误，将测试配置上传搭配注册中心，注册中心引入了白名单机制，白名单之外的不可以注册服务；
    
    
### Q&A: 注册中心和dns有什么区别？
  * 注册中心属于一级分布式架构，dsn属于多级服务架构
  * 注册中心具有 health check 机制
  * 注册中心可以实现负载均衡，通过和多个服务端建立连接池，在调用段通过一定的策略实现负载均衡
  * dns 手工配置比较麻烦，且更新后有延时
  
### 解决了上面的初始化问题，该如何进行请求？

单体应用更多是本地调用，微服务化后更多的是远程rpc调用，需要解决4个问题？
  * 客户端和服务端如何建立网络连接？ 
  
    * http通信，本质上就是tcp链接，需要三次握手、四次挥手；
    * socket通信，需要server端进行 bind()、listen 操作，等待客户端connect后进行accept，之后两者进行send/receive操作，最后close结束；
    * 如何保证cs链接有效，需要两种机制
      * health check：定时ping，保持心跳
      * 重试：指数退避，避免服务端因短时间连接数被占满短期内不可用的情况发生
    
    
  * 服务端如何处理请求？
  
    同步阻塞/同步非阻塞/异步非阻塞，比如linux里面的select、poll、epoll等server端处理技术；
    
    
  * 数据传输采用什么协议？
  
  http/dubbo等，协议需定义好消息头和消息体
  
  * 如何进行序列化和反序列化？
    序列化和反序列化就是数据编码和解码的过程，
    选用什么主要取决三个因素，跨语言支持、压缩比和压缩速率，通常pb的压缩比和压缩速率相比于json都要更好一些
    * 文本类（json/xml）
    * 二进制类（thrift，pb）