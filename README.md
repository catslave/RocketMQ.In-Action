# rocketmq.in-action
what is rocketmq? 
why use rocketmq?
what is nameserver?
what is broker?
the introduction of rocketmq

# Name Server
Name Server 主要负责管理Topic队列信息和Broker地址信息，客户端通过topic获取队列信息，通过topic获取broker信息，通过broker获取broker的地址信息等。

## Name Server
Name Server启动，首先加载默认配置文件，NamesrvConfig和NettyServerConfig，顾名思义NamesrcConfig就是NameServer相关的配置信息，NettyServerConfig就是
启动Netty时的相关配置信息。RocketMQ是使用Netty作为底层RPC通信框架的。<br/>
启动类，NamesrcStartup.class<br/>
```Java
public static NamesrcController main0(String[] args) {
  final NamesrvConfig namesrvConfig = new NamesrvConfig();//创建NamesrvConfig配置文件
  final NettyServerConfig nettyServerConfig = new NettyServerConfig();//创建NettyServerConfig配置文件
  nettyServerConfig.setListenPort(9876);//设置Netty服务端监听端口，默认9876
  
  //创建NamesrvController，传入配置文件构造controller
  final NamesrvController controller = new NamesrvController(namesrcConfig, nettyServerConfig);
  controller.initialize();//实例化controller
```
NameServer是通过NamesrvController来启动的：NameServer创建了Namesrv和NettyServer配置文件后，通过这个两个配置文件实例化了NamesrvController控制类，
然后调用controller.initialize方法进行初始化。
## NamesrvController
NamesrvController实例化了RouteInfoManager和BrokerHouseKeepingService两个对象。NameServer中最重要的就是RouteInfoManager了。
RouteInfoManager就是管理topic和broker真正的地方
```Java
  public class RouteInfoManager {
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
  }
```
BrokerHouseKeepingService专门处理broker是否存活，如果broker失效或异常，则将broker从RouteInfoManager移除
```Java
   @Override
    public void onChannelClose(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
    
    @Override
    public void onChannelException(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    @Override
    public void onChannelIdle(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
```
同时NamesrvController创建了Netty服务端NettyRemotingServer，根据NamesrvStartup提供的nettyServerConfig配置文件，以及将BrokerHouseKeepingService时间监听处理程序传入NettyRemotingServer的构造函数中
```Java
  this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHouseKeepingServer);
```
NamesrvController还创建了一个remotingExecutor线程池，用于处理Netty服务端接收到消息请求
```Java
  this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
```
NamesrvController为NettyRemotingServer注册了消息请求处理器DefaultRequestProcessor，当Netty服务端接收到消息请求时，调用remotingExecutor线程池执行DefaultRequestProcessor处理程序，DefaultRequestProcessor根据消息类型来做出相应的处理
```Java
  this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
```
## DefaultRequestProcessor
DefaultRequestProcessor实际处理消息请求的类，请求的消息类型有：TOPIC，BROKER等。
```Java
  switch (request.getCode()) {
    case RequestCode.REGISTER_BROKER:
    case RequestCode.UNREGISTER_BROKER:
    case RequestCode.GET_ROUTEINTO_BY_TOPIC:
    case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
  }
```
根据不同的消息类型，操作RouteInfoManager管理的相应HashMap。
