# RocketMQ.Producer
消息的生产者，负责产生消息。通过Name Server获取到Topic信息，从Topic中选择一个
消息队列进行消息发送。根据选择的消息队列，获取该消息队列的Broker地址，
然后生产者与该Broker建立连接，将消息发送到Broker上。
## DefaultMQProducer
一类消息划分为一个Topic，一个Topic包含多条消息队列。生产者在发送消息时，需要指定消息所属的Topic类型，然后根据Topic选择其中一条消息队列进行消息发送。顶级接口`MQAdmin`定义了创建Topic方法。`MQProducer`接口
继承`MQAdmin`并定义了发送消息方法。`DefaultMQProducer`实现了`MQProducer`接口，作为生产者的默认实现类。

Producer调用start方法启动，DefaultMQProducer调用DefaultMQProducerImpl的start方法，DefaultMQProducerImpl为该生产者创建了一个消息队列客户端实例MQClientInstance，MQClientInstance分别创建了用于客户端远程通信辅助类MQClientAPIImpl、拉取消息服务类PullMessageService（DefaultMQPushConsumerImpl才使用）以及RebalanceService负载均衡服务类（消费者才使用到）。
## Send Message
发送消息需要指定消息所属的Topic，生产者首先根据Topic去Name Server获取Topic的队列信息，如果Topic在Name Server中不存在，Name Server为使用系统默认的Topic消息队列返回给生产者。生产者根据Name Server返回回来的消息队列更新本地的Topic信息表。然后从Topic信息表中选择一条消息队列进行通信。生产者根据消息队列的Broker去Name Server查找该Broker的地址信息。最后与该Broker建立连接，将消息发送给该Broker。
```Java
package com.alibaba.rocketmq.client.impl.producer;

public class DefaultMQProducerImpl implements MQProducerInner {
	//...

	private SendResult sendDefaultImpl(Message, msg, //...) {
		//...

		//1.根据消息的Topic查找消息队列信息
		TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic);
		//...
		//2.从Topic中选择一条消息队列
		MessageQueue tmpmq = topicPublishInfo.selectOneMessageQueue(lastBrokerName);
		//3.消息传递
		sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, timeout);
		//...
	}

	private SendResult sendKernelImpl(final Message msg, final MessageQueue mq, //...) {
		//4.根据Broker查找Broker远程地址
		String brokerAddr = this.mQClientFactory.findBrokerAddressInPublish(mq.getBrokerName());
		//5.消息传递
		SendResult sendResult = this.mQClientFactory.getMQClientAPIImpl().sendMessage(
			brokerAddr, // Broker地址
			mq.getBrokerName(),	// Broker名称
			msg, //	消息
			//...
			);
		//...
	}
}
```
```Java
package com.alibaba.rocketmq.client.impl;

public class MQClientAPIImpl {
	//...

	public SendResult sendMessage(
		final String addr, final String brokerName, final Message msg, //...) {
		//...

		//默认同步调用发送消息
		return this.sendMessageSync(addr, brokerName, msg, //...);

		//...
	}

	private SendResult sendMessageSync(
		final String addr, final String brokerName, final Message msg, //...) {

		//调用Netty客户端往指定addr地址发送消息请求
		RemotingCommand response = this.remotingClient.invokeSync(addr, request, timeoutMillis);
		//...
	}

}
```
```Java
package com.alibaba.rocketmq.remoting.netty;

public class NettyRemotingClient extends NettyRemotingAbstract implements RemotingClient {
	//...

	@Override
	public RemotingCommand invokeSync(String addr, final RemotingCommand request, long timeoutMillis) {
		//根据addr地址获取channel管道，如果不存在则新建channel
		final Channel channel = this.getAndCreateChannel(addr);

		//...

		//实际发送消息的地方
		this.invokeSyncImpl(channel, request, timeoutMillis);

		//...
	}
}
```
```Java
package com.alibaba.rocketmq.remoting.netty;

public abstract class NettyRemotingAbstract {
	//...

	public RemotingCommand invokeSyncImpl(final Channel channel, final RemotingCommand request,
		final long timeoutMillis) {
		//...

		//往管道写入消息并刷新
		channel.writeAndFlush(request);

		//...
	}
}
```
## TopicPublishInfo
消息必须属于某个Topic，一个Topic包含一个消息队列。消息队列保存着对应Broker的名称信息。
```Java
pakcage com.alibaba.rocketmq.client.impl.producer;

/**
 * Topic信息
 */
public class TopicPublishInfo {
	//保存一个消息队列
	private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
}
```
```Java
pakcage com.alibaba.rocketmq.common.message;

/**
 * 消息队列信息
 */
public class MessageQueue implements Comparable<MessageQueue>, Serializable {
	//保存Topic名称
	private String topic;
	//保存Broker名称
	private String brokerName;
}
```