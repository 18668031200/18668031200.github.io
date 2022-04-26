---
title: rocketmq1
author: ygdxd
type: 原创
date: 2019-05-07 16:09:33
tags: rocketmq
categories: rocketmq
---

rocketmq namesrv
============================


Namesrv简介
--------------------------------------------------

namesrc 即 NameServer,类似于服务注册中心，它是RocketMq的调度中心，它提供了路由管理，服务注册及服务发现，服务剔除等服务。正是由于它，系统可以迅速地感知消息服务器的健康状况，对消息服务器的单点故障做出反应防止整个系统瘫痪。它还可以对消息进行负载均衡处理防止Broker宕机。
<!-- more -->

启动Namesrv
--------------------------------------------------



{% codeblock NamesrvStartup.java lang:java %}

public static NamesrvController main0(String[] args) {

        try {
            NamesrvController controller = createNamesrvController(args);
            start(controller);
            String tip = "The Name Server boot success. serializeType=" + RemotingCommand.getSerializeTypeConfigInThisServer();
            log.info(tip);
            System.out.printf("%s%n", tip);
            return controller;
        } catch (Throwable e) {
            e.printStackTrace();
            System.exit(-1);
        }

        return null;
    }
    
{% endcodeblock %}


第一行createNamesrvController(args)根据参数创建一个NamesrvController实例，然后就这个实例当做参数调用start()方法，最后输出日志。先看start(final NamesrvController controller)方法

{% codeblock NamesrvStartup.java lang:java %}

public static NamesrvController start(final NamesrvController controller) throws Exception {
...
boolean initResult = controller.initialize();

...
Runtime.getRuntime().addShutdownHook(new ShutdownHookThread(log, new Callable<Void>() {
            @Override
            public Void call() throws Exception {
                controller.shutdown();
                return null;
            }
        }));

        controller.start();
        
        return controller;
    }

{% endcodeblock %}

可以看到它先调用了controller.initialize();这个条语句。然后给系统增加了一个关闭的钩子 在系统关闭之前回去调用controller的shutdown()方法。最后调用controller.start().

可以看出这个NamesrvController就是整个启动过程关键类，而createNamesrvController(args)是生成它的方法。

{% codeblock NamesrvStartup.java lang:java %}

...
//首先解析命令
Options options = ServerUtil.buildCommandlineOptions(new Options());
        commandLine = ServerUtil.parseCmdLine("mqnamesrv", args, buildCommandlineOptions(options), new PosixParser());       
...
//生成业务配置实例
final NamesrvConfig namesrvConfig = new NamesrvConfig();
//生成netty远程通信配置实例
final NettyServerConfig nettyServerConfig = new NettyServerConfig();
...
//命令上-c 参数 系统会去读取配置文件并设置到NamesrvConfig中
if (commandLine.hasOption('c')) {
InputStream in = new BufferedInputStream(new FileInputStream(file));
...
namesrvConfig.setConfigStorePath(file);
}
//-p 参数也会设置
if (commandLine.hasOption('p')) {
...
}
...
//logback 设置
JoranConfigurator configurator = new JoranConfigurator();

configurator.doConfigure(namesrvConfig.getRocketmqHome() + "/conf/logback_namesrv.xml");

...
最后生成NamesrvController 实例
final NamesrvController controller = new NamesrvController(namesrvConfig, nettyServerConfig);

        // remember all configs to prevent discard
controller.getConfiguration().registerConfig(properties);
{% endcodeblock %}


可以看到NamesrvController 中有2个关键的配置类。



NamesrvConfig和NettyServerConfig
--------------------------------------------------

NamesrvConfig。一个业务配置类，在类中有rocketmqHome（根目录，通过读取环境变量来设置）
kvConfigPath 对应 kvConfig.json，NameServer存储KV配置属性的持久化路径。
configStorePath 对应namesrv.properties，nameServer默认配置文件路径。
orderMessageEnable，是否支持顺序消息，默认不支持。
productEnvName 生产环境名称


NettyServerConfig。通信配置类
listenPort：netty监听端口
serverWorkerThreads： netty业务线程池数量
serverCallbackExecutorThreads： netty public 任务线程池线程个数，默认为4个。用于处理消息发送，心跳检测等。
serverSelectorThreads： IO线程池线程个数，主要是NameServer、Broker端解析请求的线程个数，这类线程主要是处理网络请求，解析请求包，然后转发到个业务线池完成具体的业务操作，然后将结果返回调用方。
serverOnewaySemaphoreValue：send oneway 消息请求并发度（Broker端参数）
serverAsyncSemaphoreValue：异步消息发送最大并发度（Boreker端参数）
serverChannelMaxIdleTimeSeconds：网络连接最大空闲时间，默认120s。如果连接空闲时间超过该参数设置的值，连接将关闭。
serverSocketSndBufSize：网络socket发送缓冲区大小，默认64k。
serverSocketRcvBufSize：网络socket接受缓冲区大小，默认64k。
serverPooledByteBufAllocatorEnable： ByteBuffer是否开启缓存，建议开启
useEpollNativeSelector：是否启用Epoll IO模型，Linux环境建议开启。


NamesrvController
--------------------------------------------------

NameServer控制类。里面包含了很多重要的配置和管理信息。前面通过namesrvConfig 和nettyServerConfig 初始化了一个NamesrvController实例。

{% codeblock NamesrvController.java lang:java %}

this.namesrvConfig = namesrvConfig;
        this.nettyServerConfig = nettyServerConfig;
        this.kvConfigManager = new KVConfigManager(this);
        this.routeInfoManager = new RouteInfoManager();
        this.brokerHousekeepingService = new BrokerHousekeepingService(this);
        this.configuration = new Configuration(
            log,
            this.namesrvConfig, this.nettyServerConfig
        );
        this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");


{% endcodeblock %}

这里还生成了一个路由信息管理类，一个broker下线的监听业务类BrokerHousekeepingService，用于连接改变时去更改this.routeInfoManager的路由信息。

######RouteInfoManager的路由信息

{% codeblock RouteInfoManager.java lang:java %}

    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;


{% endcodeblock %}

topicQueueTable：Topic消息队列路由信息，消息发送时根据路由表进行负载均衡。
brokerAddrTable：Broker基础信息，包含brokerName,所属集群名称，主备broker地址。
clusterAddrTable：Broker集群信息，存储集群中所有的Broker名称。
brokerLiveTable：Broker存活信息，NameServer每次收到心跳包时都会更新该信息。
filterServerTable：过滤器列表。


而RouteInfoManager还拥有BrokerHousekeepingService的具体实现。

比如关闭连接

{% codeblock BrokerHousekeepingService.java lang:java %}
@Override
    public void onChannelClose(String remoteAddr, Channel channel) {
        this.namesrvController.getRouteInfoManager().onChannelDestroy(remoteAddr, channel);
    }
{% endcodeblock %}



{% codeblock RouteInfoManager.java lang:java %}
public void onChannelDestroy(String remoteAddr, Channel channel) {
        String brokerAddrFound = null;
        if (channel != null) {
            try {
                try {
                		//获取读锁找到具体的broker信息
                    this.lock.readLock().lockInterruptibly();
                    Iterator<Entry<String, BrokerLiveInfo>> itBrokerLiveTable =
                        this.brokerLiveTable.entrySet().iterator();
                    while (itBrokerLiveTable.hasNext()) {
                        Entry<String, BrokerLiveInfo> entry = itBrokerLiveTable.next();
                        //找到
                        if (entry.getValue().getChannel() == channel) {
                            brokerAddrFound = entry.getKey();
                            break;
                        }
                    }
                } finally {
                    //解锁
                    this.lock.readLock().unlock();
                }
            } catch (Exception e) {
                log.error("onChannelDestroy Exception", e);
            }
        }

        if (null == brokerAddrFound) {
            brokerAddrFound = remoteAddr;
        } else {
            log.info("the broker's channel destroyed, {}, clean it's data structure at once", brokerAddrFound);
        }

        if (brokerAddrFound != null && brokerAddrFound.length() > 0) {

            try {
                try {
                    //加写锁，此时只有其余线程阻塞
                    this.lock.writeLock().lockInterruptibly();
                    //删除存活信息和对应的过滤器
                    this.brokerLiveTable.remove(brokerAddrFound);
                    this.filterServerTable.remove(brokerAddrFound);
                    String brokerNameFound = null;
                    boolean removeBrokerName = false;
                    //寻找对应的基础信息
                    Iterator<Entry<String, BrokerData>> itBrokerAddrTable =
                        this.brokerAddrTable.entrySet().iterator();
                    while (itBrokerAddrTable.hasNext() && (null == brokerNameFound)) {
                        BrokerData brokerData = itBrokerAddrTable.next().getValue();

                        Iterator<Entry<Long, String>> it = brokerData.getBrokerAddrs().entrySet().iterator();
                        while (it.hasNext()) {
                            Entry<Long, String> entry = it.next();
                            Long brokerId = entry.getKey();
                            String brokerAddr = entry.getValue();
                            if (brokerAddr.equals(brokerAddrFound)) {
                                brokerNameFound = brokerData.getBrokerName();
                                //删除
                                it.remove();
                                log.info("remove brokerAddr[{}, {}] from brokerAddrTable, because channel destroyed",
                                    brokerId, brokerAddr);
                                break;
                            }
                        }

                        if (brokerData.getBrokerAddrs().isEmpty()) {
                            removeBrokerName = true;
                            itBrokerAddrTable.remove();
                            log.info("remove brokerName[{}] from brokerAddrTable, because channel destroyed",
                                brokerData.getBrokerName());
                        }
                    }

                    if (brokerNameFound != null && removeBrokerName) {
                        Iterator<Entry<String, Set<String>>> it = this.clusterAddrTable.entrySet().iterator();
                        while (it.hasNext()) {
                            Entry<String, Set<String>> entry = it.next();
                            String clusterName = entry.getKey();
                            Set<String> brokerNames = entry.getValue();
                            boolean removed = brokerNames.remove(brokerNameFound);
                            if (removed) {
                                log.info("remove brokerName[{}], clusterName[{}] from clusterAddrTable, because channel destroyed",
                                    brokerNameFound, clusterName);

                                if (brokerNames.isEmpty()) {
                                    log.info("remove the clusterName[{}] from clusterAddrTable, because channel destroyed and no broker in this cluster",
                                        clusterName);
                                    it.remove();
                                }

                                break;
                            }
                        }
                    }
						//在topick路由表进行更新
                    if (removeBrokerName) {
                        Iterator<Entry<String, List<QueueData>>> itTopicQueueTable =
                            this.topicQueueTable.entrySet().iterator();
                        while (itTopicQueueTable.hasNext()) {
                            Entry<String, List<QueueData>> entry = itTopicQueueTable.next();
                            String topic = entry.getKey();
                            List<QueueData> queueDataList = entry.getValue();

                            Iterator<QueueData> itQueueData = queueDataList.iterator();
                            while (itQueueData.hasNext()) {
                                QueueData queueData = itQueueData.next();
                                if (queueData.getBrokerName().equals(brokerNameFound)) {
                                    itQueueData.remove();
                                    log.info("remove topic[{} {}], from topicQueueTable, because channel destroyed",
                                        topic, queueData);
                                }
                            }

                            if (queueDataList.isEmpty()) {
                                itTopicQueueTable.remove();
                                log.info("remove topic[{}] all queue, from topicQueueTable, because channel destroyed",
                                    topic);
                            }
                        }
                    }
                } finally {
                    //最后进行解锁操作
                    this.lock.writeLock().unlock();
                }
            } catch (Exception e) {
                log.error("onChannelDestroy Exception", e);
            }
        }
    }
    
    {% endcodeblock %}
    
   可以看到每当有broker下线时，整个路由信息都会通过加锁的方式进行实时更新。
    
    
   在初始化了NamesrvController实例以后，下面就是调用它的初始化方法initialize()。
   
   {% codeblock NamesrvController.java lang:java %}
   
   //读取KV配置
   this.kvConfigManager.load();
   //生成netty远程服务
   this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
    //线程池初始化
    this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
//注册处理类
this.registerProcessor();
//跳过定时线程初始化和ssl
...
    
   {% endcodeblock %}
   
   最后调用start()方法，其实就是netty的启动。
   
   
 
参考
--------------------------------------------------

《rocketmq技术内幕》
   
    
