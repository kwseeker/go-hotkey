# jd-hotkey worker 工作原理

worker 是 SpringBoot 应用。

## 启动阶段

先看下注册了哪些Bean：

```java
//热键的缓存对象，是Caffeine实例
Cache<String, Object> hotKeyCache()
//客户端变更监听器，客户端可能新增、断线、删除
IClientChangeListener clientChangeListener()
//配置中心（etcd）的连接实例
IConfigCenter client()
//统计键的访问次数的计数器，异步阻塞地方式从COUNTER_QUEUE（LinkedBlockingQueue）中获取key访问上报信息（KeyCountItem：KeyCountModel的列表）
//然后根据热键配置规则统计并记录键的访问次数， 存储到 HashMap中
//HashMap key:  appName + Constant.COUNT_DELIMITER + ruleKey (即KeyCountModel.ruleKey)
//HashMap value: hotHitCount + "-" + totalHitCount, 这两个值是累加的
//当HashMap中Entry的size() >= 300, configCenter.putAndGrant(), 即存一批一起存到etcd (etcd也是KV数据库)
CounterConsumer counterConsumer()
//创建了一组KeyConsumer任务，并在 Executors.newCachedThreadPool() 线程池中启动
//KeyConsumer任务从BlockingQueue<HotKeyModel> QUEUE中以阻塞的方式读取 HotKeyModel 数据
//判断如果有新key上报或key删除，将对应的key添加到监听器（KeyListener）或从监听器删除
Consumer consumer()	//DispatcherConfig
//将HotKeyModel数据推送到 BlockingQueue<HotKeyModel> QUEUE
@Component KeyProducer   
//添加或删除对key的监听
@Component KeyListener
//Etcd的客户端
@Component EtcdStarter
//初始化了公共静态变量，当作数据容器使用
@Component InitStarter
//启动Netty Server端，在pipeline中装载了一组消息处理Filter,  ClientChangeListener 用于关闭客户端连接
@Component NodesServerStarter
//用于获取ApplicationContext的ApplicationContextAware接口实现
@Component ApplicationContextProvider
```

主要的类

```java
public class KeyCountItem implements Delayed {
    private String appName;
    private long createTime;
    //list是一个client的10秒内的数据，一个rule如果每秒都有数据，那list里就有10条；是10秒一次上报？
    private List<KeyCountModel> list;
}

public class KeyCountModel {
    /**
     * 对应的规则名，如 pin_2020-08-09 11:32:43
     */
    private String ruleKey;
    /**
     * 总访问次数
     */
    private int totalHitCount;
    /**
     * 热后访问次数
     */
    private int hotHitCount;
    /**
     * 发送时的时间
     */
    private long createTime;
}
```

## 请求处理

worker 通过 Netty（路径：com.jd.platform.hotkey.worker.netty）与 client 交互，

通过 etcd 客户端（路径：com.jd.platform.hotkey.worker.starters.EtcdStarter，etcd-java）与 etcd 交互。

![worker交互流程图]()

由此可见worker功能也很简单，主要就是：

+ 添加或删除对 key 的监控并通知所有client
+  统计监控的key的访问次数并上报到etcd （判断key是否是热key并不在worker中完成）

### Netty交互

```java
private class ChildChannelHandler extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) {
        NodesServerHandler serverHandler = new NodesServerHandler();
        serverHandler.setClientEventListener(clientChangeListener);
        //使用@Order注解排序依次是
        //HeartBeatFilter 	心跳包处理，ping-pong
        //AppNameFilter 	jd-hotkey客户端上报自己的appName
        //HotKeyFilter			热key消息，包括从netty来的和mq来的。收到消息，都发到队列去
        //KeyCounterFilter	对热key访问次数和总访问次数进行累计
        serverHandler.addMessageFilters(messageFilters);

        ByteBuf delimiter = Unpooled.copiedBuffer(Constant.DELIMITER.getBytes());
        ch.pipeline()
            .addLast(new DelimiterBasedFrameDecoder(Constant.MAX_LENGTH, delimiter))
            .addLast(new MsgDecoder())
            .addLast(new MsgEncoder())
            .addLast(serverHandler);
    }
}
```

### etcd客户端交互

```java
public class JdEtcdBuilder {
    /**
     * @param endPoints 如https://127.0.0.1:2379 有多个时逗号分隔
     */
    public static JdEtcdClient build(String endPoints) {
        return new JdEtcdClient(EtcdClient.forEndpoints(endPoints).withPlainText().build());
    }
}
```

etcd客户端配置

```yaml
etcd:
  server: ${etcdServer:http://127.0.0.1:2379} #etcd的地址，重要！！！
  workerPath: ${workerPath:default} #该worker放到哪个path下，譬如放/app1下，则该worker只能被app1使用，不会为其他client提供服务
```

