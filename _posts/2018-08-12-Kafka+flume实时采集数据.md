一、模拟产生日志

在IDEA的resource文件夹下面新建log4j.properties定义日志格式,其中flume和log4j的整合配置可以查看Log4j Appender
```  
#设置日志格式
log4j.rootCategory=ERROR,console，flume
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
#flume和log4j整合
log4j.appender.flume = org.apache.flume.clients.log4jappender.Log4jAppender
log4j.appender.flume.Hostname = 192.168.116.10
log4j.appender.flume.Port = 41414
log4j.appender.flume.UnsafeMode = true  
```  
利用java的log4j模拟产生日志  
```  
import org.apache.log4j.Logger;
 
public class LoggerGenerator {
    private static Logger logger=Logger.getLogger(LoggerGenerator.class.getName());
    public static void main(String[] args)throws Exception{
        int i=0;
        while (1==1){
            Thread.sleep(1000);
            logger.info("now is "+i);
            i++;
        }
 
    }
}

```  
具体依赖和配置过程可以查看 Flume远程实时采集Windows产生的log4j产生的数据

二、配置Flume

1.配置

1.1配置flumekafka

我这里使用Flume版本是1.6，所以写法和官网的不一样。如果启动flume的过程中报错.producer.requiredAcks=1, channel=c2, kafka.topic=streaming, type=org.apache.flume.sink.kafka.KafkaSink} }
2018-12-29 06:17:26,698 (conf-file-poller-0) [ERROR - org.apache.flume.node.AbstractConfigurationProvider.loadSinks(AbstractConfigurationProvider.java:427)] Sink k2 has been removed due to an error during configuration
org.apache.flume.conf.ConfigurationException: brokerList must contain at least one Kafka broker

应该就是flume版本不同配置不同导致的。
```  
#命名代理
b1.sources = r2
b1.sinks = k2
b1.channels = c2
# 配置监控文件
b1.sources.r2.type =avro
b1.sources.r2.bind=0.0.0.0
b1.sources.r2.port = 41414
#b1.sources.r2.interceptors = i1
#b1.sources.r2.interceptors.i1.type = timestamp
# 配置sink
b1.sinks.k2.type =org.apache.flume.sink.kafka.KafkaSink
b1.sinks.k2.topic = streaming
b1.sinks.k2.brokerList= 192.168.116.10:9092
#b1.sinks.k2.kafka.bootstrap.servers = 192.168.116.10:9092
b1.sinks.k2.producer.requiredAcks = 1
b1.sinks.k2.batchSize = 20
# 配置channel
b1.channels.c2.type = memory
# 将三者串联
b1.sources.r2.channels = c2
b1.sinks.k2.channel = c2
```  
这里地方配置有点复杂，具体配置可以看官网

1.3启动flume

可以在命令行输入flume-ng查看帮助命令，我的启动脚本是

flume-kafka：
```  
flume-ng agent -n b1 -c /usr/local/src/apache-flume-1.6.0-bin/conf -f /usr/local/src/apache-flume-1.6.0-bin/conf/flumekafka.conf -Dflume.root.logger=INFO,console
```  
 
```  
flume-ng agent -n a1 -c $FLUME_HOME/conf -f $FLUME_HOME/conf/streaming.conf -Dflume.root.logger=INFO,console >> /usr/tmp/flume/1.log & （后台启动）
```  
三、配置Kafka

1.启动zookeeper

到每台机器的zookeeper的bin目录输入./zkServer.sh start

2.以后台形式启动kafka

在每台机器上的kafka安装目录的bin目录下输入： kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties &

确保zookeeper和kafka都已经启动了



3.新建主题

3.1新建一个主题：kafka-topics.sh --create --zookeeper master:2181,slave1:2181,slave2:2181 --replication-factor 1 --partitions 1 --topic streaming

如果出现报错as the leader reported an error: NOT_LEADER_FOR_PARTITION可以到kafka安装目录的logs文件夹下面的server.log查看详细报错。

3.2查看新建的主题kafka-topics.sh --describe --zookeeper master:2181,slave1:2181,slave2:2181 --topic streaming



也可以通过 kafka-topics.sh --list --zookeeper master:2181,slave1:2181,slave2:2181查看所有的主题

如果要删除topic，需要到zookeeper的bin文件夹下依次输入

./zkCli.sh   启动zookeeperClient

ls /brokers/topics  查看有多少topic

rmr /brokers/topics/test  删除test的topic

3.3新开一个窗口启动一个消费者

kafka-console-consumer.sh --zookeeper master:2181,slave1:2181,slave2:2181 --topic streaming

这个时候先启动flume再启动IEDA的日志模拟器，

等待生成20个（flume里面设置batch20）可以看到日志已经被消费了。
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume1.png)  
