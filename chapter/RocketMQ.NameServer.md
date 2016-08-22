# RocketMQ.In-Action
What is RocketMQ? 
Why use RocketMQ?
What is NameServer?
What is Broker?
How Producer does work?
How Consumer does work?<br/>
The Introduction Of RocketMQ.

# Name Server
Name Server 主要负责管理集群中所有的Topic队列信息和Broker地址信息，客户端可以通过Name Server获取topic信息，通过topic获取broker信息，通过broker获取broker地址信息等等。<br/>

NamesrvStartup为Name Server的启动类，NamesrvStartup通过加装默认配置文件，实例化NamesrvController控制器类来执行Name Server操作。NamesrvController根据配置文件创建Netty服务端用于监听客户端的请求，Netty服务端调用DefaultRequestProcessor来处理客户端的请求，DefaultRequestProcessor根据消息类型做出相应的操作更新RouteInfoManager。

## NamesrvStartup
启动Name Server，通过调用NamesrvStartup类的main方法进行创建，该类首先加载系统默认配置文件，NamesrvConfig和NettyServerConfig，顾名思义NamesrcConfig就是Name Server相关的配置信息，例如rocketmq的home目录等，NettyServerConfig就是启动Netty服务端时的相关配置信息，例如监听端口、工作线程池数、服务端发送接收缓存池的大小等。RocketMQ是使用Netty作为底层RPC通信框架的。所以Name Server实际是启动了一个Netty服务端来监听消息，同时通过向Netty注册监听处理程序来处理消息的请求。
```Java
package com.alibaba.rocketmq.namesrv;

public class NamesrvStartup {

  /**
   * 程序或命令行启动，调用的入口
   * @param args
   */
  public static void main(String[] args) {
    main0(args);
  }

  public static NamesrcController main0(String[] args) {
    //...
    
    //创建NamesrvConfig配置文件
    final NamesrvConfig namesrvConfig = new NamesrvConfig();
    //创建NettyServerConfig配置文件
    final NettyServerConfig nettyServerConfig = new NettyServerConfig();
    nettyServerConfig.setListenPort(9876);//设置Netty服务端监听端口，默认9876
    
    //创建NamesrvController，传入配置文件构造controller
    final NamesrvController controller = new NamesrvController(namesrcConfig, nettyServerConfig);
    controller.initialize();//初始化controller
    
    //...
  }
  
}
```
## NamesrvController
NamesrvController是NamesrvStartup的控制器类，在这里创建了Netty服务端用于接收客户端的消息请求，并向Netty注册的消息请求处理程序，Netty服务端接收到消息请求后，调用消息请求处理程序处理消息请求。<br/>
NamesrcStartup创建了NamesrvConfig和NettyServerConfig配置文件后，通过这两个配置文件实例化NamesrvController控制器类，然后调用NamesrvController.initialize方法进行初始化。<br/>
NamesrvController在创建时，实例化了RouteInfoManager和BrokerHouseKeepingService两个对象。Name Server中最重要的就是RouteInfoManager类。
Name Server所有的Topic和Borker信息都保存在RouteInfoManager中。BrokerHouseKeepingService用于处理Broker状态事件，当Broker失效、异常或者关闭，则将Broker从RouteInfoManager中移除。<br/>
NamesrvController初始化时，根据NettyServerConfig创建了Netty服务端，并向Netty服务端注册了请求处理程序，同时还创建了一个用于处理消息请求的线程池。
```Java
package com.alibaba.rocketmq.namesrv;

public class NamesrvController {
  //Name Server 配置文件
  private final NamesrvCofnig namesrvConfig;
  //Netty 服务端配置文件
  private final NettyServerConfig nettyServerConfig;
  //Netty 服务端
  private RemotingServer remotingServer;
  //Broker状态监听处理程序
  private BrokerHouseKeepingServer brokerHouseKeepingService;
  //线程池 用于处理消息请求DefaultRequestProcessor
  private ExecutorService remotingService;
  
  //...
  
  /**
     * 实例化NamesrvController
     * @param namesrvConfig Name Server 配置文件
     * @param nettyServerConfig Netty 服务端配置文件
     */
  public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
    this.namesrvConfig = namesrvConfig;
    this.nettyServerConfig = nettyServerConfig;
    //...
    this.routeInfoManager = new RouteInfoManager();
    this.brokerHousekeepingService = new BrokerHousekeepingService(this);
  }
  
  /**
   * 初始化
   * @return
   */
  public boolean initialize() {
    //...
    
    //创建Netty服务端，nettyServerConfig为Netty服务端相关配置文件，例如前面在NamesrvStartup中配置了监听端口9876
    //brokerHouseKeepingServer broker状态处理程序
    this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
    //创建一个固定大小的线程池，根据nettyServerConfig配置文件提供的默认工作线程数，默认值为8
    this.remotingExecutor = Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new      ThreadFactoryImpl("RemotingExecutorThread_"));
    //注册消息处理程序
    this.registerProcessor();
    
    //...
  }
}
```
### RouteInfoManager
RouteInfoManager保存所有的topic和broker信息，Netty服务端接收到请求后，消息请求处理程序根据请求类型，例如注册Broker或者新建Topic，更新RouteInfoManager。
```Java
  package com.alibaba.rocketmq.namesrv.routeinfo;

  public class RouteInfoManager {
    //Topic列表信息
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    //Broker地址信息
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    //Broker集群信息
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    //Broker更新信息
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    //Broker过滤信息
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
    
    //...
    
    /**
      * 注册新的Broker，将新的Broker添加到brokderAddrTable等信息中
      */
    public RegisterBrokerResult registerBroker(...) {
      //...
      //更新Broker集群信息
      this.clusterAddrTable.put(clusterName, brokerNames);
      //更新Broker地址信息
      this.brokerAddrTable.put(brokerName, brokerData);
      //更新Broker状态信息
      this.brokerLiveTable.put(...);
      //更新Broker过滤信息
      this.filterServerTable.put(brokerAddr, filterServerList);
      //...
    }
  }
```
### BrokerHouseKeepingService
BrokerHouseKeepingService专门处理broker是否存活，如果broker失效、异常或者关闭，则将broker从RouteInfoManager移除。同时将与该broker相关的topic信息也一起删除。Netty服务端专门启动了一个线程用于监听管道失效、异常或者关闭等的事件队列，当事件队列里面有新事件时，则出列并判断事件的类型，然后调用BrokerHouseKeepingService对应的方法来处理该事件。
```Java
package com.alibaba.rocketmq.remoting.netty;

/**
 * Netty服务端公共抽象类
 */
public abstract class NettyRemotingAbstract {

  //...

  /**
   * 新建线程监听事件队列
   */
  class NettyEventExecuter extends ServiceThread {
  
    //事件队列，这里使用LinkedBlockingQueue队列，基于链表的阻塞队列，是线程安全的队列。新事件自动加入到队尾，
    private final LinkedBlockingQueue<NettyEvent> eventQueue = new LinkedBlockingQueue<NettyEvent>();
    //...
    
    /**
     * 添加新事件
     * @param event
     */
    public void putNettyEvent(final NettyEvent event) {
      this.eventQueue.add(event);
    }
    
    @Override
    public void run() {
      //请求事件处理程序 BrokerHouseKeepingService
      final ChannelEventListener listener = NettyRemotingAbstract.this.getChannelEventListener();
      
      while(!this.isStoped()) {
        //每3秒访问一次事件队列
        NettyEvent event = this.eventQueue.poll(3000, TimeUnit.MILLISECONDS);
        if (event != null && listener != null) {
          //判断事件类型
          switch (event.getType()) {
          case IDLE://失效
            //调用 BrokerHouseKeepingService 的 onChannelIdle
            listener.onChannelIdle(event.getRemoteAddr(), event.getChannel());
            break;
          case CLOSE://关闭
            //调用 BrokerHouseKeepingService 的 onChannelClose
            listener.onChannelClose(event.getRemoteAddr(), event.getChannel());
            break;
          case CONNECT://连接
            //调用 BrokerHouseKeepingService 的 onChannelConnect
            listener.onChannelConnect(event.getRemoteAddr(), event.getChannel());
            break;
          case EXCEPTION://异常
            //调用 BrokerHouseKeepingService 的 onChannelException
            listener.onChannelException(event.getRemoteAddr(), event.getChannel());
            break;
          default:
              break;
          }
        }
      }
    }
  }
  
}
```
```Java
  package com.alibaba.rocketmq.namesrv.routeinfo;

  /**
   * Broker状态事件处理程序，处理Broker状态的服务，实现了ChannelEventListener接口。
   * 当Netty有新事件时，将调用ChannelEventListener接口处理事件。
   */
  public class BrokerHouseKeepingService implements ChannelEventListener {
  
    //...
    
    @Override
    public void onChannelClose(String remoteAddr, Channel channel) {
      //管道关闭时，将broker从RouteInfoManager中移除
      this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
    
    @Override
    public void onChannelException(String remoteAddr, Channel channel) {
      //管道异常时，将broker从RouteInfoManager中移除
      this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }

    @Override
    public void onChannelIdle(String remoteAddr, Channel channel) {
      //管道失效时，将broker从RouteInfoManager中移除
      this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
    
    //...
  }
```
```Java
  package com.alibaba.rocketmq.namesrv.routeinfo;

  public class RouteInfoManager {

    //...
    
    /**
     * 销毁管道
     * @param remoteAddr 客户端地址信息
     * @param channel 客户端管道
     */
    public void onChannelDestroy(String remoteAddr, Channel channel) {
    
      //1.根据channel客户端管道找到brokerAddr对应的broker地址信息
      //2.根据brokerAddr地址将broker从brokerLiveTable和filterServerTable移除
      this.brokerLiveTable.remove(brokerAddrFound);
      this.filterServerTable.remove(brokerAddrFound);
      //3.根据brokerAddr遍历brokerAddrTable，将broker从brokerAddrTable中移除
      this.brokerAddrTable.remove();
      //4.根据brokerAddr遍历clusterAddrTable，将broker从clusterAddrTable中移除
      this.clusterAddrTable.remove();
      //4.根据brokerAddr遍历topicQueueTable，将broker从topicQueueTable中移除
      this.topicQueueTable.remove();
      
    }
  }
```
NettyEventExecuter线程会每隔3秒定时轮询事件队列eventQueue，当有新事件时，则从队列中取出事件，判断事件类型然后调用BrokerHouseKeepingService处理事件。NettyEventExecuter还提供了putNettyEvent方法用于添加事件到队列中。<br/>
### NettyRemotingServer
NettyRemotingServer就是Netty服务端，用于监听客户端的请求，然后更新RouteInfoManager路由表信息。<br/>
NamesrvController创建了Netty服务端NettyRemotingServer，根据NamesrvStartup对象提供的nettyServerConfig配置文件，以及将BrokerHouseKeepingService处理程序传入NettyRemotingServer的构造函数中。然后调用NettyRemotingServer的start方法，启动Netty。<br/>
Netty启动后，开始监听客户端的请求，当有新的请求到来时，将该请求封装成一个Task任务，然后调用线程池处理该Task。
```Java
package com.alibaba.rocketmq.remoting.netty;

/**
 * Netty 服务端
 */
public class NettyRemotingServer extends NettyRemotingAbstract implements RemotingServer {
  
  /**
   * 通过Netty配置文件和管道事件监听器实例化Netty服务端
   * @param nettyServerConfig Netty配置文件
   * @param channelEventListener 管道事件监听器 BrokerHouseKeepingService实现了该接口
   */
  public NettyRemotingServer(final NettyServerConfig nettyServerConfig, final ChannelEventListener channelEventListener) {
    //...
    //Netty 服务端启动辅助类
    this.serverBootstrap = new ServerBootstrap();
    //Netty 配置文件
    this.nettyServerConfig = nettyServerConfig;
    //管道事件监听器 BrokerHouseKeepingService实现了该接口
    this.channelEventListener = channelEventListener;
    //...
  }
  
  @Override
  public void start() {
    //正常的启动Netty
    this.serverBootstrap.group(this.eventLoopGroupBoss, this.eventLoopGroupSelector).channel(NioServerSocketChannel.class)
      .localAddress(new InetSocketAddress(this.nettyServerConfig.getListenPort()))//监听配置文件设置的9876端口
      .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        public void initChannel(SocketChannel ch) throws Exception {
          ch.pipeline().addLast(
            new NettyEncoder(),//自定义Netty编码类
            new NettyDecoder(),//自定义Netty解码类
            new IdleStateHandler(0, 0, nettyServerConfig.getServerChannelMaxIdleTimeSeconds()),//心跳空闲检查时间
            new NettyConnetManageHandler(),//处理管道连接、端口、异常等事件
            new NettyServerHandler());//处理管道消息事件
        }
      });
      
    this.nettyEventExecutor.start();//启动一个线程，监听事件队列。就是上面提到的管道断开、异常或关闭时的事件队列
  }
  
}
```
Netty启动时，向pipeline管道添加了两个自定义的ChannelHandler，一个是`NettyConnnetManageHandler`，一个是`NettyServerHandler`。

上文有提到Netty专门启动了一个线程NettyEventExecuter，用于监听（管道断开、异常或关闭时触发的事件）事件队列的。NettyEventExecuter线程还提供了一个putNettyEvent方法用于添加事件到队列中。那么这个方法由谁来调用呢？就是NettyConnetManageHanlder处理程序来调用的。

`NettyConnetManageHandler`继承于ChannelDuplexHandler，并且实现了channelInactive、userEventTriggered、exceptionCaught等方法。当有管道失效时，会自动触发channelInactive方法，然后在这个方法里面调用nettyEventExecuter.putNettyEvent(event)方法，将事件添加到事件队列中。
```Java
package com.alibaba.rocketmq.remoting.netty;

/**
 * Netty 服务端
 */
public class NettyRemotingServer extends NettyRemotingAbstract implements RemotingServer {
  //...
  
  class NettyConnetManageHandler extends ChannelDuplexHandler {
    //...
    
   @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
      //获取客户端远程地址
      final String remoteAddress = RemotingHelper.parseChannelRemoteAddr(ctx.channel());
      //创建一个NettyEvent事件，标记该事件类型为CLOSE，然后将该事件添加到nettyEventExecuter的eventQueue队列中
      NettyRemotingServer.this.putNettyEvent(new NettyEvent(NettyEventType.CLOSE, remoteAddress.toString(), ctx.channel()));
  }
}
```
`NettyServerHandler`用于处理请求信息，继承于SimpleChannelInboundHandler，并且实现了channelRead0方法。
```Java
package com.alibaba.rocketmq.remoting.netty;

/**
 * Netty 服务端
 */
public class NettyRemotingServer extends NettyRemotingAbstract implements RemotingServer {
  //...
  
  class NettyServerHandler extends SimpleChannelInboundHandler<RemotingCommand> {
    
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
      //调用NettyRemoteAbstract的processMessageReceived方法来处理请求
      processMessageReceived(ctx, msg);
    }
}
```
`NettyRemotingAbstract`根据请求类型，调用相应的处理程序。NamesrvController在创建NettyRemotingServer时，向NettyRemotingServer注册了默认的处理程序，并且传入了线程池用于处理该程序。`NettyRemotingAbstract`将该请求封装成一个Task，然后使用该线程池执行该Task。
```Java
package com.alibaba.rocketmq.remoting.netty;

/**
 * Netty服务端公共抽象类
 * @author shijia.wxr
 */
public abstract class NettyRemotingAbstract {
  //NamesrvController注册的默认处理程序，以及用于执行该处理程序的线程池
  protected Pair<NettyRequestProcessor, ExecutorService> defaultRequestProcessor;
  
  public void processMessageReceived(ChannelHandlerContext ctx, RemotingCommand msg) throws Exception {
    final RemotingCommand cmd = msg;
    //判断事件类型
    switch (cmd.getType()) {
      case REQUEST_COMMAND://请求事件
          processRequestCommand(ctx, cmd);
          break;
      case RESPONSE_COMMAND://响应事件
          processResponseCommand(ctx, cmd);
          break;
      default:
          break;
    }
  }
  
  /**
   * 请求事件处理程序
   * @param ctx
   * @param cmd
   */
  public void processRequestCommand(final ChannelHandlerContext ctx, final RemotingCommand cmd) {
    final Pair<NettyRequestProcessor, ExecutorService> matched = this.processorTable.get(cmd.getCode());
    //使用默认的请求处理程序
    final Pair<NettyRequestProcessor, ExecutorService> pair = null == matched ? this.defaultRequestProcessor : matched;
    
    //把请求封装成一个Task
    Runnable run = new Runnable() {
      @Override
      public void run() {
        //...
        
        //调用默认的请求处理程序处理请求 DefaultRequestProcessor的processRequest方法
        final RemotingCommand response = pair.getObject1().processRequest(ctx, cmd);
        
        //...
      }
    }
    //将Task任务提交到线程池中执行
    pair.getObject2().submit(run);
  }
}
```
### DefaultRequestProcessor
`DefaultRequestProcessor`为默认的请求处理程序，Netty服务端接收到所有请求，都会交由其处理。`DefaultRequestProcessor`在线程池中执行。

NamesrvController为NettyRemotingServer注册了请求处理程序DefaultRequestProcessor，当Netty服务端接收到消息请求时，调用remotingExecutor线程池执行DefaultRequestProcessor处理程序，DefaultRequestProcessor根据消息类型来做出相应的处理。
```Java
package com.alibaba.rocketmq.namesrv.processor;

/**
  * 默认请求处理程序
  */
public class DefaultRequestProcessor implements NettyRequestProcessor {
  
  /**
   * 处理请求
   * @param ctx 管道
   * @param request 请求消息
   * @return 响应消息
   * @throws RemotingCommandException
   */
  @Override
  public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
    //...
    //根据请求消息类型
    switch (request.getCode()) {
    case RequestCode.REGISTER_BROKER://注册新Broker
      return this.registerBroker(ctx, request);
    case RequestCode.UNREGISTER_BROKER://移除Broker
      return this.unregisterBroker(ctx, request);
    //...
    default:
      break;
    }
    return null;
  }
  
  /**
   * 注册新Broker
   * @param ctx
   * @param request
   * @return
   * @throws RemotingCommandException
   */
  public RemotingCommand registerBroker(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
    //...
    //通过调用RouteInfoManager的registerBroker方法来注册新的Broker
    this.namesrvController.getRouteInfoManager().registerBroker(...);
    //...
  }
}
```
