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
启动Netty时的相关配置信息。RocketMQ是使用Netty作为底层RPC通信框架的。
启动类，NamesrcStartup<br/>
```Java
public static NamesrcController main0(String[] args) {
  final NamesrvConfig namesrvConfig = new NamesrvConfig();
  final NettyServerConfig nettyServerConfig = new NettyServerConfig();
  nettyServerConfig.setListenPort(9876);
  
  final NamesrvController controller = new NamesrvController(namesrcConfig, nettyServerConfig);
  controller.initialize();
```
NameServer真正是通过NamesrvController来启动的：NameServer创建了Namesrv和NettyServer配置文件后，通过这个两个配置文件实例化了NamesrvController控制类，
然后调用controller.initialize方法进行初始化。
## RouteInfoManager
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
