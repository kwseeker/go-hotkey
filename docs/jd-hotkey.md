# jd-hotkey

主要包括下面几个部分：

+ etcd

  用作配置中心（存储热key规则配置，以及热key各维度的统计信息）、服务注册与发现（存储客户端和worker各个实例信息和工作状态）

+ worker

  可以有多个，根据每秒待测key数量调整，worker节点信息会上报到etcd，然后客户端client（业务服务）才知道自己收集的待探测的key应该发到哪里进行统计计算；worker会获取热key规则对上报的key进行统计计算，结果（热key、黑名单等）会推送回所有客户端以及etcd。

+ client

  业务服务，会获取热key规则，通过规则收集及上报热key到hash计算后指定的worker

+ dashboard

  控制台，用于查看worker client 各个实例信息和工作状态、对热key规则进行增删改查、查看热key频率、已缓存数量等统计信息，持久化统计数据到MySQL

代码轻量（共9K多行代码），部署简单。不过文档很简陋，很多使用细节只能看源码推测。



## 安装部署

**安装etcd**:

官方提供了安装脚本，默认装到了/tmp，这里手动修改下。

```shell
ETCD_VER=v3.4.24

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GITHUB_URL}
INSTALL_DIR=/home/lee/bin

rm -f ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf ${INSTALL_DIR}/etcd-download-test && mkdir -p ${INSTALL_DIR}/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz -C ${INSTALL_DIR}/etcd-download-test --strip-components=1
rm -f ${INSTALL_DIR}/etcd-${ETCD_VER}-linux-amd64.tar.gz
# 服务端
${INSTALL_DIR}/etcd-download-test/etcd --version
# 客户端
${INSTALL_DIR}/etcd-download-test/etcdctl version
```

**启动etcd**:

这里选择单机启动，只是为了测试jd-hotkey。

```shell
./etcd
```

为方便查看装下etcdv3-browser（Web应用），图方便直接用docker镜像安装：

```shell
docker pull joinsunsoft/etcdv3-browser:1.0.0
docker run -d --name=etcdv3-browser -p 9980:80 joinsunsoft/etcdv3-browser:1.0.0
# http://localhost:9980 Username: ginghan Password: 123456
```

**启动worker**:

worker是spring boot web应用，内部启动了Tomcat（port:8080）和Netty服务器（port:11111），暂时先启动一个节点。

```shell
# jar包启动
java -jar $JAVA_OPTS worker-0.0.1-SNAPSHOT.jar --etcd.server=${etcdServer}
# IDE启动
# 命令行参数：默认 http://127.0.0.1:2379
--etcd.server=${etcdServer}
```

**启动dashboard**：

dashboard是spring boot web应用，页面使用模板引擎实现，端口8081，用户名密码：admin/123456。

需要先初始化MySQL数据库hotkey_db，导入resource下db.sql文件，修改application.yml（数据库useSSL=false）。

建了８张表：

+ biz_access_token
+ hk_change_log
+ hk_key_record
+ hk_key_timely
+ hk_rules
+ hk_statistics
+ hk_summary
+ hk_user

**启动client**:

源码client包只是根据规则采集上报热key的组件（包含Netty客户端），需要在业务服务中引入此组件，官方提供了测试应用sample，可以启动并在这个web应用中测试。



## 工作原理

### client上报

+ **获取热key规则**

  + **连接etcd**

  + **获取规则信息**

    + **规则定义**

      jd-hotkey对于热key规则定义没找到资料，看配置页面格式是json或xml,包含下面字段：

      ```
      key-(*代表任意以key为前缀), prefix-是否前缀, interval-间隔时间(秒), threshold-阈值, duration-缓存时间(秒),默认60
      ```

      推测规则用json表示是：

      ```json
      # 以key为前缀２秒内出现达到10次，即为热key,热key缓存60s
      {
      	"key": "key",
      	"prefix":true,
      	"interval":2,
      	"threshold":10,
      	"duration":60
      }
      #经过测试发现必须传成json数组格式，再改为下面格式成功
      [{
      	"key": "key",
      	"prefix":true,
      	"interval":2,
      	"threshold":10,
      	"duration":60
      }]
      ```

      所属APP: 添加规则必须指定所属APP，但是下拉可选列表为空，也不能手写，没查到资料，看前端源码发现来源于用户信息表，有个app_name列。

      > 所属APP来源：Ajax请求“/user/info”接口读取hk_user的app_name列，然后将值比对后填充到下拉列表。
      >
      > 所以下拉列表为空是因为app_name为空，这里填充“sample”。
      >
      > ```javascript
      > success : function(data) {
      > 	console.log(data)
      > 	var role = data.role;
      > 	if(role === "ADMIN"){
      > 		$("#apps").append("<option></option>");
      > 	}
      > 	var apps = data.appNames;
      > 	var appName = data.appName;
      > 	for (var i = 0; i < apps.length; i++) {
      > 		var app = apps[i];
      > 		if(app === appName){
      > 			$("#apps").append("<option selected = selected>" + apps[i] + "</option>");
      > 		}else{
      > 			$("#apps").append("<option>" + apps[i] + "</option>");
      > 		}
      > 	}
      > }
      > # UPDATE `hotkey_db`.`hk_user` SET `app_name`='sample' WHERE `id`='2';
      > ```

      Rules类数据结构

      ```java
      //自增主键
      private Integer id;
      //规则的json或xml字符串
      private String rules;
      //所属应用app
      private String app;
      private String updateUser;
      private Date updateTime;
      private Integer version;
      ```

  + **注册规则变更监听**

  + **注册hotkey变更监听**

+ **key收集上报**

  + **规则的使用**

    规则怎么用文档也没说，还是看源码，更新规则后客户端会收到一条通知（etcd的监听机制），从这条通知入手，定位源码切入点：

    ```
    com.jd.platform.hotkey.client.etcd.EtcdStarter - rules info changed. begin to fetch new infos. rule change is [kv {
      key: "/jd/rules/sample"
      create_revision: 4677
      mod_revision: 4677
      version: 1
      value: "[{\n\t\"key\": \"key\",\n\t\"prefix\":true,\n\t\"interval\":2,\n\t\"threshold\":10,\n\t\"duration\":60\n}]"
    }
    ]
    ```

    事件通知处理：事件中的信息只是打印日志用，然后拉取全量rule信息才是重点。

    ```java
    List<Event> eventList = watchUpdate.getEvents();
    JdLogger.info(getClass(), "rules info changed. begin to fetch new infos. rule change is " + eventList);
    //全量拉取rule信息
    fetchRuleFromEtcd();
    
    private boolean fetchRuleFromEtcd() {
        IConfigCenter configCenter = EtcdConfigFactory.configCenter();
        try {
            List<KeyRule> ruleList = new ArrayList<>();
            //从etcd获取自己的rule
            String rules = configCenter.get(ConfigConstant.rulePath + Context.APP_NAME);
            //1规则为空
            if (StringUtil.isNullOrEmpty(rules)) {
                JdLogger.warn(getClass(), "rule is empty");
                //会清空本地缓存队列
                notifyRuleChange(ruleList);
                return true;
            }
            //2规则不为空
            ruleList = FastJsonUtils.toList(rules, KeyRule.class);
            //通过EventBus继续传递
            notifyRuleChange(ruleList);
            return true;
        } catch (StatusRuntimeException ex) {
            ...
        }
    }
    
    //用的Guava的EventBus，EventBusCenter只是用单例模式封装一下
    EventBusCenter.getInstance().post(new KeyRuleInfoChangeEvent(rules));
    ```

    然后找EventBus接收的位置：

    

  + **获取worker信息**

  + **key hash计算，数据上报**

+ **client使用热key数据**

### dashboard管理

+ **连接etcd**

+ **热key规则CRUD**
+ **更新事件发布**
+ **client、worker实例状态监控**
+ **worker推送的统计信息查询**
+ **用户管理**

### worker计算与推送

+ **获取热key规则**
  + **连接etcd**
+ **计算热key统计数据**
+ **结果推送给etcd、client**