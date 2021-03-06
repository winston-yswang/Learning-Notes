### 1、RPC 是什么？

RPC 称远程过程调用（Remote Procedure Call），用于解决分布式系统中服务之间的调用问题。 通俗地讲，就是开发者能够像调用本地方法一样调用远程的服务。 所以，RPC的作用主要体现在这两个方面：

- 屏蔽远程调用跟本地调用的区别，让我们感觉就是调用项目内的方法；
- 隐藏底层网络通信的复杂性，让我们更专注于业务逻辑。

#### RPC 框架基本架构

RPC 框架包含三个最重要的组件，分别是客户端、服务端和注册中心。在一次 RPC 调用流程中，这三个组件是这样交互的：

- 服务端在启动后，会将它提供的服务列表发布到注册中心，客户端向注册中心订阅服务地址；
- 客户端会通过本地代理模块 Proxy 调用服务端，Proxy 模块收到负责将方法、参数等数据转化成网络字节流；
- 客户端从服务列表中选取其中一个的服务地址，并将数据通过网络发送给服务端；
- 服务端接收到数据后进行解码，得到请求信息；
- 服务端根据解码后的请求信息调用对应的服务，然后将调用结果返回给客户端。

![rpc](C:\Users\Lenovo\Desktop\笔记\CS-Learning-Notes\notes\pics\rpc.png)

从上面这张图中，可以看见 RPC 框架一般有这些组件：服务治理(注册发现)、负载均衡、容错、序列化/反序列化、编解码、网络传输、线程池、动态代理等角色，当然有的RPC框架还会有连接池、日志、安全等角色。

1. **客户端（服务消费端）** ：调用远程方法的一端。
2. **客户端 Stub（桩）** ： 这其实就是一代理类。代理类主要做的事情很简单，就是把你调用方法、类、方法参数等信息传递到服务端。
3. **网络传输** ： 网络传输就是你要把你调用的方法的信息比如说参数啊这些东西传输到服务端，然后服务端执行完之后再把返回结果通过网络传输给你传输回来。网络传输的实现方式有很多种比如最近基本的 Socket或者性能以及封装更加优秀的 Netty（推荐）。
4. **服务端 Stub（桩）** ：这个桩就不是代理类了。我觉得理解为桩实际不太好，大家注意一下就好。这里的服务端 Stub 实际指的就是接收到客户端执行方法的请求后，去指定对应的方法然后返回结果给客户端的类。
5. **服务端（服务提供端）** ：提供远程方法的一端。

#### 具体调用过程

1）服务消费方（client）调用以本地调用方式调用服务；

2）client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体；

3）client stub找到服务地址，并将消息发送到服务端；

4）server stub收到消息后进行解码；

5）server stub根据解码结果调用本地的服务；

6）本地服务执行并将结果返回给server stub；

7）server stub将返回结果打包成消息并发送至消费方；

8）client stub接收到消息，并进行解码；

9）服务消费方得到最终结果。

**RPC的目标就是要2~8这些步骤都封装起来，让用户对这些细节透明。**

参考文章：[如何手撸一个较为完整的RPC框架](https://juejin.cn/post/6992867064952127524) 



### 2、RPC框架需要解决的问题

RPC要达到的目标：远程调用时，要能够像本地调用一样方便，让调用者感知不到远程调用的逻辑。

#### 2.1 怎么做到透明化远程服务调用？

使用代理，一般RPC框架采用动态代理方式，通过实现一个RpcProxyClient 代理类，代理类的invoke 方法中封装与远端服务通信的细节，消费方可以从RPCProxyClient 获得服务提供方的接口，当执行方法时就会调用invoke 方法。

#### 2.2 怎么对消息进行编码和解码？

远程调用中，因为两个进程的地址空间不一样，所以无法通过调用函数指针调用函数，也没法通过内存传递参数。

使用网络通信的第一步，就是要确定客户端和服务端相互通信的消息结构。客户端的请求消息结构一般需要包括以下内容：

- 接口名称、方法名称、参数类型和参数值；
- requestId，标识唯一请求id等等。

同理，服务端返回的消息结构一般包括：返回值，状态码和requestId等。

一旦确定了消息的数据结构后，下一步就是要考虑序列化与反序列化。通常要重合考虑通用性、性能和可扩展性。

#### 2.3 如何实现网络通信？

消息数据结构被序列化为二进制流后，下一步就要进行网络通信。一般直接是使用基于netty的通信框架。

#### 2.4 如何实现服务发布和订阅？

分布式架构中，一个服务势必会有多个实例，需要解决如何获取实例的问题。采用zookeeper实现服务自动注册与发现功能，zookeeper可以充当一个服务注册表（Service Registry），让多个服务提供者形成一个集群，让服务消费者通过服务注册表获取具体的服务访问地址（ip+端口）去访问具体的服务提供者。

此外，zookeeper提供了“心跳检测”功能，它会定时向各个服务提供者发送一个请求（实际上建立的是一个 Socket 长连接），如果长期没有响应，服务中心就认为该服务提供者已经“挂了”，并将其剔除。

服务消费者也会去监听相应路径，一旦路径上的数据有任务变化（增加或减少），zookeeper都会通知服务消费方服务提供者地址列表已经发生改变，从而进行更新。

#### 2.5 Redis缓存

如果每次都去注册中心查询列表，效率很低，那么就要加缓存。有了缓存，就要考虑缓存的更新问题

#### 2.6 异步调用AIO

客户端总不能每次调用完都等着服务端返回数据，所以就要支持异步调用。AIO作为异步非阻塞，AIO发起IO操作之后，通知服务器去完成函数操作，这个时间客户端可以去做其他的事情。等到服务器完成操作之后，就会调用客户端的接口，返回结果数据。

#### 2.7 使用线程池

针对高并发的客户端请求，服务端可以维护一个线程池，针对每次的客户端的请求，可以直接从线程池中拿出一个线程，直接去执行客户端的任务，省去了每次创建线程的开销，同时提高了服务端的响应速度。

参考文章：[深入理解 RPC](https://juejin.cn/post/6844903443237175310) 、[设计RPC框架应考虑的问题](https://blog.csdn.net/weixin_41907511/article/details/102834674) 、[一个优秀的RPC框架需要考虑的问题](https://blog.csdn.net/weixin_37704921/article/details/89212111?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2.pc_relevant_aa&utm_relevant_index=5)



### 3、自己动手从0开始实现一个分布式 RPC 框架

表达思路：[如何设计一个 RPC 框架](https://juejin.cn/post/6870276943448080392)

参考文章：[自己动手从0开始实现一个分布式 RPC 框架](https://zhuanlan.zhihu.com/p/388848964)



### 4、RPC心跳机制相关讨论

参考文章：[面试官：要不我们聊一下“心跳”的设计？](https://juejin.cn/post/7041405419713429511)



### 5、负载均衡是怎么做的，如何考虑？

参考文章：[五种负载均衡策略](https://www.whywhy.vip/archives/40)