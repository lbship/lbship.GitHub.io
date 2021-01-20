整体架构图

![image](https://upload-images.jianshu.io/upload_images/9716180-0de0b1e63c531267?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

工具

Flink 1.11.2

Scala 2.11

Tableau 2020.2

**一、模拟发送数据**

新建一个类KafkaProducer用来模拟产生消费数据，代码如下:

```
package TopNitems
 
import java.text.SimpleDateFormat
import java.time.{LocalTime, ZonedDateTime}
import java.time.format.DateTimeFormatter
import java.util.{Date, Locale, Properties}
 
import scala.io.Source
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord}
 
import Array._
import scala.util.Random.shuffle
 
 
object KafkaProducers {
  def main(args: Array[String]): Unit = {
    SendtoKafka("test")
  }
  def SendtoKafka(topic:String): Unit = {
    val pro=new Properties()
    pro.put("bootstrap.servers", "192.168.226.10:9092")
    pro.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    pro.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    val producer=new KafkaProducer[String,String](pro)
    var member_id= range(1,10)
    var goods=Array("Milk","Bread","Rice","Nodles","Cookies","Fish","Meat","Fruit","Drink","Books","Clothes","Toys")
    //var ts=DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss",Locale.CHINA).format( ZonedDateTime.now())
    while (true) {
      var ts=new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date())
      var msg = shuffle(member_id.toList).head + "\t" + shuffle(goods.toList).head + "\t" + ts+"\t"+"\n"
      print(msg)
      var record = new ProducerRecord[String, String](topic, msg)
      producer.send(record)
      Thread.sleep(2000)
    }
    //val source=Source.fromFile("C:\\UserBehavior.csv")
    //for (line<-source.getLines()){
    // val record=new ProducerRecord[String,String](topic,line)
 
    //print(ts)
    producer.close()
 
 
 
  }
 
}
```

1.启动ZooKeeper

./zkServer.sh start

.2.启动Kafka

./kafka-server-start.sh -daemon $KAFKA_HOME/config/server.properties

3.创建topic

./kafka-topics.sh --create --zookeeper 192.168.226.10:2181 --replication-factor 1 --partitions 1 --topic test

查看topic是否创建成功

./kafka-topics.sh --list --zookeeper 192.168.226.10:2181

4.在IDEA运行KafkaProducer，可以看到每隔2秒产生一个消费

![image](https://upload-images.jianshu.io/upload_images/9716180-8c267737eaa4c589?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

启动监听

./kafka-console-consumer.sh --topic test --from-beginning --bootstrap-server 192.168.226.10:9092

测试成功，说明可以被消费

![image](https://upload-images.jianshu.io/upload_images/9716180-b03ccc07aec29a7f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**二、数据写入Clickhouse **

  Clickhouse可以直接作为Kafka的Consumer，这个是[官网](https://clickhouse.tech/docs/zh/engines/table-engines/integrations/kafka/)介绍，[格式](https://clickhouse.tech/docs/en/interfaces/formats/)这里查看,但是直接消费，没有ETL过程，我们还是用flink来消费，方便其他处理。

Flink 在 1.11.0 版本对其 JDBC connector 进行了一次较大的重构,包的名字也不一样：

二者对 Flink 中以不同方式写入 ClickHouse Sink 的支持情况如下：

<caption style="box-sizing: border-box; outline: 0px; font-weight: normal; overflow-wrap: break-word;"> </caption>
| API名称 | flink-jdbc | flink-connector-jdbc |
| --- | --- | --- |
| DataStream | 不支持 | 支持 |
| Table API (Legecy) | 支持 | 不支持 |
| Table API (DDL) | 不支持 | 不支持 |

本次使用flink 1.11.2版本，所以采用的方式为flink-connector-jdbc+DataStream的方式写入数据到ClickHouse

先添加依赖

```
<dependency>
			<groupId>org.apache.flink</groupId>
			<artifactId>flink-connector-jdbc_2.11</artifactId>
			<version>1.11.2</version>
		</dependency>
		<dependency>
			<groupId>ru.yandex.clickhouse</groupId>
			<artifactId>clickhouse-jdbc</artifactId>
			<version>0.2.4</version>
		</dependency>
		<!-- 添加 Flink Table API 相关的依赖 -->
		<dependency>
			<groupId>org.apache.flink</groupId>
			<artifactId>flink-table-planner-blink_${scala.binary.version}</artifactId>
			<version>${flink.version}</version>
			<scope>provided</scope>
		</dependency>
		<dependency>
			<groupId>org.apache.flink</groupId>
			<artifactId>flink-table-api-scala-bridge_${scala.binary.version}</artifactId>
			<version>${flink.version}</version>
		</dependency>
		<dependency>
			<groupId>org.apache.flink</groupId>
			<artifactId>flink-table-common</artifactId>
			<version>${flink.version}</version>
			<scope>provided</scope>
		</dependency>
```

代码如下，这里采用jdbc的方式写入，每5条批量写入一次

```
package TopNitems
 
import java.sql.PreparedStatement
import java.text.SimpleDateFormat
import java.util.{Date, Properties}
 
import org.apache.flink.api.common.serialization.SimpleStringSchema
import org.apache.flink.connector.jdbc.{JdbcConnectionOptions, JdbcExecutionOptions, JdbcSink, JdbcStatementBuilder}
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.{CheckpointingMode, TimeCharacteristic}
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.table.api.bridge.scala.StreamTableEnvironment
import org.apache.flink.api.scala._
import org.apache.flink.table.descriptors.Kafka
 
 
//当前版本的 flink-connector-jdbc，使用 Scala API 调用 JdbcSink 时会出现 lambda 函数的序列化问题。我们只能采用手动实现 interface 的方式来传入相关 JDBC Statement build 函数
class CkSinkBuilder extends JdbcStatementBuilder[(Int, String, String)] {
  def accept(ps: PreparedStatement, v: (Int, String, String)): Unit = {
    ps.setInt(1, v._1)
    ps.setString(2, v._2)
    ps.setString(3, v._3)
  }
}
 
object To_CK {
  def main(args: Array[String]): Unit = {
  
    //获得环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1) //设置并发为1，防止打印控制台乱序
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) //Flink 默认使用 ProcessingTime 处理,设置成event time
    val tEnv = StreamTableEnvironment.create(env) //Table Env 环境
   //从Kafka读取数据
    val pros = new Properties()
    pros.setProperty("bootstrap.servers", "192.168.226.10:9092")
    pros.setProperty("group.id", "test")
    pros.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
    pros.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
    pros.setProperty("auto.offset.reset", "latest")
    import org.apache.flink.api.scala._
    val dataSource = env.addSource(new FlinkKafkaConsumer[String]("test", new SimpleStringSchema(), pros))
    val sql="insert into ChinaDW.testken(userid,items,create_date)values(?,?,?)"
    val result = dataSource.map(line => {
      val x = line.split("\t")
      //print("收到数据",x(0),x(1),x(2),"\n")
      val member_id = x(0).trim.toLong
      val item = x(1).trim
      val times = x(2).trim
      var time = 0l
      try{time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse(times).getTime} //时间戳类型
      catch {case e: Exception => {print( e.getMessage)}}
      (member_id.toInt, item.toString ,time.toLong)
    }).assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[(Int, String, Long)](Time.seconds(2)) {
        override def extractTimestamp(t: (Int, String, Long)): Long = t._3
      }).map(x=>{(x._1,x._2,new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(x._3))}) //时间还原成datetime类型
        //result.print()
        result.addSink(JdbcSink.sink[(Int,String,String)](sql,new CkSinkBuilder,new JdbcExecutionOptions.Builder().withBatchSize(5).build(),
          new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
            .withUrl("jdbc:clickhouse://XX.XX.XX.XX:8123")
            .withDriverName("ru.yandex.clickhouse.ClickHouseDriver")
            .withUsername("default")
            .build()
        ))
 
 
 
      env.execute("To_CK")
  }
 
 
}
 
```

到Clickhouse查询，数据已经成功写入

![image](https://upload-images.jianshu.io/upload_images/9716180-6f08a865b9520c73?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**三、利用Tableau进行可视化**

可视化环节就比较简单了，这里选择了Tableau连接Clickhouse，因为简单方便，下面这个图大概就用了2分钟就搞定了，这里要说明一下，tableau必须2020版本以上，不然连接clickhouse可能发生字段被截取的情况。。首先安装好clickhouse的ODBC驱动，我安装的是clickhouse-odbc-1.1.7-win64.msi，然后在控制面板设置好ODBC的连接，如图

![image](https://upload-images.jianshu.io/upload_images/9716180-cd731b7b9d3816d7?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后tableau配置clickhouse的ODBC，具体可以百度一下 Tableau如何连接Clickhouse

![image](https://upload-images.jianshu.io/upload_images/9716180-7808a8a08bafda7f?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

简单拖拉做成下面这个表，现在还剩一个问题，Tableau如何作为大屏，自动刷新？ 强大的tableau当然有解决方法:

方法一:发布到Tableau server，然后利用浏览器自带的网页刷新功能，例如QQ浏览器，网址加&: refresh=yes,可以参考这个[博客](https://blog.csdn.net/weixin_45588393/article/details/107780188)

方法二:安装[Tableau拓展程序](https://extensiongallery.tableau.com/extensions?version=2020.3&per-page=50) ，到官网找到Auto Refresh这个插件，然后拖进去就可以直接用了，可以看到右下角有一个刷新的倒计时。

![image](https://upload-images.jianshu.io/upload_images/9716180-07dfd5725eb2ab75?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

到此，整个项目结束了。
