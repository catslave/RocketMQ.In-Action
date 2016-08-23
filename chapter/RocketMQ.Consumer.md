# RocketMQ.Consumer
消费者，消费消息。生产者将消息发送到Broker，Broker将消息发送给消费者，消费者接受消息然后处理消息。消费者也可以订阅消息。Broker收到消息后会主动将消息推送给已订阅的消费者。

## Subscribe
创建一个新的消费者，可以给该消费者订阅某个Topic下的消息，然后为消费者添加消息处理程序，当有新消息到来时，自动调用消息处理程序处理消息。

消费者默认有两种实现，`DefaultMQPushConsumer`和`DefaultMQPullConsumer`。`DefaultMQPushConsumer`类型消费者采用订阅的方式，订阅某个Topic，当Broker有这类Topic新消息时会主动推送给这类消费者。`DefaultMQPullConsumer`类型消费者，自己主动轮询监听是否有最新的消息。默认我们使用`DefaultMQPushConsumer`类型消费者。

消费者订阅Topic消息，`DefaultMQPushConsumerImpl`将订阅消息封装成`SubscriptionData`，保存到订阅队列中，最后更新Broker的订阅服务列表。Broker收到消息后，根据订阅服务列表过滤出订阅者，将消息发送给指定的订阅者进行消费。
```Java
package com.alibaba.rocketmq.client.impl.consumer;

public class DefaultMQPushConsumerImpl implements MQConsumerInner {
	//...

	/**
     * 订阅Topic
     * @param topic Topic名称
     * @param subExpression 过滤表达式
     * @throws MQClientException
     */
	public void subscribe(String topic, String subExpression) throws MQClientException {

		//将topic封装成订阅信息 SubscriptionData
		SubscriptionData subscriptionData = FilterAPI.buildSubscriptionData(this.defaultMQPushConsumer.getConsumerGroup(),
			topic, subExpression);
		//将订阅信息保存到订阅服务队列中
		this.rebalanceImpl.getSubscriptionInner().put(topic, subscriptionData);
		if(this.mQClientFactory != null) {
			//发现心跳消息以及更新Broker的订阅列表
			this.mQClientfactory.sendHeartbeatToAllBrokerWithLock();
		}
	}
}
```
```Java
package com.alibaba.rocketmq.client.impl.factory;

public class MQClientInstance {
	//...

	public void sendHeartbeatToAllBrokerWithLock() {
		//...
		//发送心跳检查
		this.sendHeartbeatToAllBroker();
		//更新Broker的订阅服务列表
		this.uploadFilterClassSource();
		//...
	}

	public void uploadFilterClassSource() {
		//....

		this.uploadFilterClassSourceToAllFilterServer(
			consumerGroup, className, topic, filterClassSource);

		//...
	}

	public void uploadFilterClassSourceToAllFilterServer(
		final String consumerGroup, final String fullClassName, final String topic,
		final String filterClassSource) throws UnsupportedEncodingException {
		//...

		this.mQClientAPIImpl.registerMessageFilterClass(fsAddr, consumerGroup, topic, fullClassName, //...);

		//...
	}
}
```
```Java
package com.alibaba.rocketmq.client.impl;

public class MQClientAPIImpl {
	//...

	public void registerMessageFilterClass(final String addr,
		final String consumerGroup,
		final String topic,
		final String className,
		//...) {

		//...

		//设置请求消息类型为REGISTER_MASTER_FILTER_CLASS
		RemotingCommand.createRequestCommand(RequestCode.REGISTER_MASTER_FILTER_CLASS, requestHeader);
		//发送消息事件到Broker，更新Broker的订阅服务列表
		this.remotingClient.invokeSync(addr, request, timeoutMillis);

		//...
	}
}
```
## Consume Message
消费者成功订阅消息后，Broker接收到新消息会立即通过订阅服务列表过滤出订阅的消费者，然后将消息发送给消费者。消费者客户端接收到消息后，调用已注册的消息处理程序消费消息。

消费者启动时，`DefaultMQPushConsumerImpl`会创建一个消费者客户端实例`MQClientInstance`，并且启动一个默认的消息处理服务`ConsumeMessageConcurrentlyService`。当客户端实例`MQClientInstance`接收到消息请求时，调用`ConsumeMessageConcurrentlyService`的consumeMessageDirectly方法。该方法会调用消费者注册的消息处理程序进行消息的处理。
```Java
package com.alibaba.rocketmq.client.impl.consumer;

public class DefaultMQPushConsumerImpl implements MQConsumerInner {
	//...

	/**
     * 消费者注册消息处理程序，当接收到新消息时会自动调用该处理程序
     * 处理消息。
     * @param messageListener
     */
    public void registerMessageListener(MessageListener messageListener) {
        this.messageListenerInner = messageListener;
    }
}
```
```Java
package com.alibaba.rocketmq.client.impl;

public class MQClientAPIImpl {
	//...

	public MQClientAPIImpl(final NettyClientConfig nettyClientConfig, 
		final ClientRemotingProcessor clientRemotingProcessor, //...) {
		//...

		//创建一个Netty客户端，监听消息
		this.remotingClient = new NettyRemotingClient(nettyClientConfig, null);
		//默认的消息处理程序，根据消息类型做出相应的处理
		this.clientRemotingProcessor = clientRemotingProcessor;
		//给Netty客户端注册消息处理程序，监听的消息类型为CONSUME_MESSAGE_DIRECTLY
		//表示立即处理消息
		this.remotingClient.registerProcessor(RequestCode.CONSUME_MESSAGE_DIRECTLY, 
			this.clientRemotingProcessor, null);
	}
}
```
```Java
package com.alibaba.rocketmq.client.impl;

/**
 * 客户端远程处理程序
 */
public class ClientRemotingProcessor implements NettyRequestProcessor {
	//...

	@Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
        switch (request.getCode()) {
        //...
        case RequestCode.CONSUME_MESSAGE_DIRECTLY:
            return this.consumeMessageDirectly(ctx, request);
        default:
            break;
        }
        return null;
    }

    private RemotingCommand consumeMessageDirectly(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
    	//...

    	this.mqClientFactory.consumeMessageDirectly(msg, requestHeader.getConsumerGroup(), requestHeader.getBrokerName());

    	//....
    }
}
```
```Java
package com.alibaba.rocketmq.client.impl.consumer;

public class ConsumeMessageConcurrentlyService implements ConsumeMessageService {
	//....

	@Override
    public ConsumeMessageDirectlyResult consumeMessageDirectly(MessageExt msg, String brokerName) {
    	//...

    	//最终调用消费者注册的消息处理程序消费消息
    	ConsumeConcurrentlyStatus status = this.messageListener.consumeMessage(msgs, context);

    	//...
    }
}
```