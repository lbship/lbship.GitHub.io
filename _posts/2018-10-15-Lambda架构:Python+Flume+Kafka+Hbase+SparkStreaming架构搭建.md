# 一、模拟网站实时产生数据

- 1.利用python模拟产生日志

这里的日志选用慕课网日志，原始的日志文件是这样的：



需要进行处理，这里选用python脚本处理和模拟生成日志，代码如下：
```  
import time

def timeformate(s):
    s=s.split('/')
    years=s[2].split(':')[0]
    days=s[0]
    months={'Jan':1,'Feb':2,'Mar':3,'Apr':4,'May':5,'Jun':6,'Jul':7,'Aug':8,'Sep':9,'Oct':10,'Nov':11,'Dec':12}
    try:
        month=months[s[1]]
    except:
        month=time.localtime().tm_mon #为了防止出现拼写错误报错，如果没有就取当前月份
    hours=s[2].split(':')[1]
    minutes=s[2].split(':')[2]
    secondes=s[2].split(':')[3]
    #利用字典把月份简写转换成数字
    timestring=str(years)+str(month)+str(days)+str(hours)+str(minutes)+str(secondes)
    #b=time.strptime(timestring,"%Y-%m-%d %H:%M:%S")
    return timestring
def main():
    f=open(r'D:\vm\access\10000_access.log', 'r').readlines()
    for line in f:
        time.sleep(3) #设置2秒钟产生一条日志
        L=line.strip().split(' ')
        ip=L[0]
        times=timeformate(L[3][1:]) #日期特殊处理
        url=L[5]+L[6]+L[7]
        status=L[8]
        soure=L[14]
        log="{}\t{}\t{}\t{}\t{}\n".format(ip,times,url,status,soure)
        print(log)
        with open(r'D:\vm\access\1\1.log','a+')as file:
            file.write(log) #写入本地文件
    f.close()
if __name__ == '__main__':
    main()
```  

处理完成的日志如下，设置每两秒产生一条日志：
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume4.png) 


# 二、设置Flume
```  
a1.sources=r1
a1.sinks=k1
a1.channels=c1
#第一步:配置source
a1.sources.r1.type=exec
a1.sources.r1.channels=c1
#配置需要监控的日志输出目录
a1.sources.r1.command=tail -F /mnt/hgfs/vm/access/1/1.log
a1.sources.r1.shell=/bin/sh -c
#配置sink
a1.sinks.k1.type =org.apache.flume.sink.kafka.KafkaSink
a1.sinks.k1.topic = streaming
a1.sinks.k1.brokerList= 192.168.116.10:9092
a1.sinks.k1.kafka.bootstrap.servers = 192.168.116.10:9092
a1.sinks.k1.producer.requiredAcks = 1
a1.sinks.k1.batchSize = 5
#配置channel
a1.channels.c1.type=memory
#将三者串联
a1.sources.r1.channels=c1
a1.sinks.k1.channel=c1  
```  
# 三、设置Kafka

- 1.启动zookeeper

到每台机器的zookeeper的bin目录输入./zkServer.sh start

- 2.以后台形式启动kafka

在每台机器上的kafka安装目录的bin目录下输入： kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties 

- 3.新建一个主题：kafka-topics.sh --create --zookeeper master:2181,slave1:2181,slave2:2181 --replication-factor 1 --partitions 1 --topic streaming

- 4.启动一个kafka消费窗口测试一下：

输入kafka-console-consumer.sh --zookeeper master:2181,slave1:2181,slave2:2181 --topic streaming

这时窗口已经处于阻塞状态，然后启动python生成日志，启动flume：flume-ng agent -n a1 -c /usr/local/src/apache-flume-1.6.0-bin/conf -f /usr/local/src/apache-flume-1.6.0-bin/conf/logflume.conf -Dflume.root.logger=INFO,console

可以看到kafka控制台已经得到日志了，说明kafka已经成功消费了通过flume采集过来的python日志。 目前已经完成日志-flume-kafka的环节了，接下来设置kafka-SparkStreaming环节。
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume5.png) 




# 四、设置SparkStreaming
```  
package Sparkstreaming

import domain.Clicklog
import org.apache.spark.SparkConf
import org.apache.spark.streaming.kafka.KafkaUtils
import org.apache.spark.streaming.{Seconds, StreamingContext}

object KafkaStreaming {
  def main(args: Array[String]): Unit = {
    if(args.length !=4){
      System.err.println("error")
    }
    val Array(zk,group,topics,numThreads)=args
    val conf=new SparkConf().setMaster("local[2]").setAppName("kafkastreaming")
    val ssc=new StreamingContext(conf,Seconds(5))
    val topicMap=topics.split(",").map((_,numThreads.toInt)).toMap
    val messages=KafkaUtils.createStream(ssc,zk,group,topicMap)
    val cleanlog=messages.map(_._2).map(line=>{
      val logs=line.split("\t")
      val ip=logs(0)
      val time=logs(1)
      val status=logs(3)
      val source=logs(4)
      val courses=logs(2).split("/")(1)
      var course="null"
      if(courses.startsWith("course")){course=courses}
      Clicklog(ip,time,course,status,source)
    })
    cleanlog.print()
      //.flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_).print()
    ssc.start()
    ssc.awaitTermination()


  }
}  
```  
在domain包里面建立一个case class 实体类

package domain
case class ClickLog(ip:String, time:String, course:String, status:Int, source:String)
执行一下streaming程序，输入4个参数192.168.116.10:2181,192.168.116.11:2181,192.168.116.12:2181 test streaming 1 启动，可以发现streaming已经采集到我们储存到click log里面的信息了，到目前为止，已经完成了日志-flume-kafka-streaming的整套架构。


![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume6.png) 










