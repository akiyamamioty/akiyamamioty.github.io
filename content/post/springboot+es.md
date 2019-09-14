---
title: "springboot2.0 + es5.5 + rabbitmq搭建搜索服务踩坑"
date: 2019-09-14T16:05:38+08:00
draft: false
tags: ["Spring", "Elasticsearch", "MQ"]
categories: ["Spring", "Elasticsearch"]
---

前一阵子准备为项目搭建一个简单的搜索服务，虽然业务数据库mongodb提供了文本搜索的支持，但是在大量文档需要通过关键词进行定位时，es明显更加适合去作为一个搜索引擎（虽然我们之前大部分使用到了ELK那套分析和可视化的特性）。Elasticsearch建立在Lucene之上并且支持极其快速的查询和丰富的查询语法，偶尔也可以作为一个轻量级的NoSQL。但是对复杂查询和聚合操作的能力并不是很强。

本篇不会提及如何搭建一个简单搜索服务，而是记录一下大约一周工作时间内遇见的几个坑。。


## 为什么选择elasticsearch 5.x?
新服务没有任何历史包袱，理论上应该用最新的6.x，然而spring-data-elasticsearch只支持到的5.x，时间紧也无法很好直接封装一层api，也是因为ELK那套东西之前版本混乱，无奈es从2.x直接到了5.x。查询了一下5.x和2.x的差别，简单说就是**磁盘空间-50%，索引时间-50%，查询性能+25%。**

**同时也由于spring-data-elasticsearch必须升级到3.0.7，导致spring必须升级到2.x，也直接导致了后面踩到的坑。**

## 踩坑记录
### **docker安装es会默认安装x-path plugin**

虽然spring-data支持es5.x，但是功能并不非常完善，因此如果安装了x-path插件，需要引入org.elasticsearch.client:x-pack-transport:5.5.0，版本必须和es版本一致，并且自己实现TransportClient，如下:

```
@Component
public class ESconfig {
	@Bean
	public TransportClient transportClient() throws UnknownHostException {
		TransportClient client = new PreBuiltXPackTransportClient(Settings.builder()
					.put("cluster.name", "docker-cluster")
					.put("xpack.security.user", "elastic:changeme")
					.build())
					.addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("0.0.0.0"), 9300));
		return client;
	}
}
```

这也是因为不想再到docker里去处理x-path这个插件而选择的一个比较快捷的解决方案，没必要的情况下，暂时也不用接触到es本身的一些东西。

### **使用默认序列化方式, mq会保存message的class信息导致deserialized失败**

一直没有提到标题中的rabbitmq，因为只是单纯的用它作为一个消息队列，当数据发生变化时，将消息id丢入mq，由search服务这边的consumer去消费。
  
问题就是在消息丢入mq时，封装成了一个自己的Object， 导致使用rabbitTemplate.receiveAndConvert时失败，因为message会带着Object的package信息。无奈之下，consumer只能直接获取queue里的message bytes, 用ObjectMapper.readValue的方法将json形式转换成一个Object。

### **gradle配置可以使用-Dloader.main指定启动函数**
正是因为引入了mq，所以search服务需要启动一个consumer，用的方法是另外实现一个不启动Web服务的Application，并且配置一个SimpleMessageListenerContainer和MessageListenerAdapter如下:

```
@Bean
SimpleMessageListenerContainer container(ConnectionFactory connectionFactory,
        MessageListenerAdapter listenerAdapter,
        MQconfig properties) {
    SimpleMessageListenerContainer container = new SimpleMessageListenerContainer();
    container.setConnectionFactory(connectionFactory);
    container.setQueueNames(properties.getQueueName());
    container.setMessageListener(listenerAdapter);

    return container;
}

@Bean
MessageListenerAdapter listenerAdapter() {
    MessageListenerAdapter listenerAdapter = new MessageListenerAdapter(itemConsumer,
            "consume");
    return listenerAdapter;
}
```
问题在于gradle配置的时候，找了很久如何使得build出来的jar包可以指定-Dloader.main指定启动Application，解决方法如下：
在xxx.gradle文件里添加

```
bootJar {
    manifest {
        attributes 'Main-Class': 'org.springframework.boot.loader.PropertiesLauncher'
    }
}
```
在springboot 1.5.9的项目里，需要指定启动Application，需要添加

```
springBoot{
    layout = "ZIP"
}
```
查看是否生效的办法是build以后 直接解压jar包，在xxx(项目名)/META-INFO/MANIFEST.MF里查看，如果

```
Main-Class: org.springframework.boot.loader.PropertiesLauncher
```
则正确，如果

```
Main-Class: org.springframework.boot.loader.JarLauncher
```
则依旧会启动文件里的Start-Class


### **es无法修改Index的mapping**
由于只是单纯使用了es的文本检索功能，导致实际应用时有许多搜索结果不尽如人意的地方，比如搜索“桌子”， 无法搜索到 “电脑桌/办公桌”等xx桌内容，这样的情况还有很多。 因此加入了synonym dictionary，在需要分词的字段上不使用本身的ik_smart分词器，这样某些字段的mapping需要改为

```
// analyzer是自己的分词器名字
@Field(type = FieldType.Text, index = true, analyzer = "synonym")
private String description;
```
由于es的mapping无法修改，只能通过手动创建一个新的mapping，再通过reIndex方法去backfill数据（es5.x自带了reIndex 的api）。网上有通过alias的方法，在某些修改场景下，不需要重新启动/部署应用就可以平滑的修改mapping，具体可以查询了解一下。

**以上差不多搭建一个搜索服务踩到的一些坑，有几个消耗了大量时间和精力去解决，在此列出来希望希望有借鉴意义。之后搜索服务有优化的地方，还会继续慢慢更新**