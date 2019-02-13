
# 快速入门


> [官方文档已经很清楚了](https://rocketmq.apache.org/docs/quick-start/)


## 安装

```bash
# 下载
$ wget http://mirrors.tuna.tsinghua.edu.cn/apache/rocketmq/4.4.0/rocketmq-all-4.4.0-source-release.zip

# 编译
$ unzip rocketmq-all-4.4.0-source-release.zip
$ cd rocketmq-all-4.4.0/
$ mvn -Prelease-all -DskipTests clean install -U

# 
$ cp -r distribution/target/apache-rocketmq /opt/websuite/apache-rocketmq-bin
```


## 基本操作

### Start Name Server

```bash
$ nohup sh /opt/websuite/apache-rocketmq-bin/bin/mqnamesrv &

# 查看日志
$ tail -f ~/logs/rocketmqlogs/namesrv.log
```

### Start Broker

```bash
$ nohup sh /opt/websuite/apache-rocketmq-bin/bin/mqbroker -n localhost:9876 &

# 查看日志
$ tail -f ~/logs/rocketmqlogs/broker.log
```

### 查看启动的 JVM 进程

```bash
$ jcmd
...
87872 org.apache.rocketmq.namesrv.NamesrvStartup
87949 org.apache.rocketmq.broker.BrokerStartup -n localhost:9876
...
```

### Send & Receive Messages

在发送/接收消息之前，我们需要告诉客户端 name servers 的位置。 

RocketMQ提供了多种方法来实现这一目标。 为简单起见，我们使用环境变量 `NAMESRV_ADDR`

```bash
$ export NAMESRV_ADDR=localhost:9876

$ /opt/websuite/apache-rocketmq-bin/bin/tools.sh  org.apache.rocketmq.example.quickstart.Producer

$ /opt/websuite/apache-rocketmq-bin/bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

### Shutdown Servers

```bash
# 关闭 broker
$ /opt/websuite/apache-rocketmq-bin/bin/mqshutdown broker

# 关闭 namesrv
$ /opt/websuite/apache-rocketmq-bin/bin/mqshutdown namesrv
```



## 一些细节

### tools.sh 工具

获取 JAVA_HOME、classpath 等一些环境变量信息，执行 指定的 class 文件

### org.apache.rocketmq.example.quickstart.Producer

向 `TopicTest`、`TagA` 发送 1000 条 `"Hello RocketMQ " + i` 消息。

> 源码： https://github.com/apache/rocketmq/blob/master/example/src/main/java/org/apache/rocketmq/example/quickstart/Producer.java











