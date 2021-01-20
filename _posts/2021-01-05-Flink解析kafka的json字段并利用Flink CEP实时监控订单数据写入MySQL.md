上篇博客写了Flink接入Kafka数据并实时写入数据库实时展示，这次利用Flink CEP进行实时监控。

整体架构图如下:

![image](https://upload-images.jianshu.io/upload_images/9716180-47a1e18ad35b7e53?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

实现目标如下：

1.查看顾客是否点击之后立即购买，是的话输出用户id，购买商品，时间

2.如果同一个顾客买了5次牛奶，输出用户id，时间

后面有时间的话，再研究把监控数据写入MySQL或者ES等

先了解一下Flink CEP 开发过程，大概分为三步:

a.定义Pattern

b.把pattern应用于输入流 CEP.pattern(inputstream, pattern) 变成patternstream

c.通过select或process算子筛选出符合pattern的流变成Datastream

实际上就是从普通流中输出符合匹配模式的流出来。



**二、开发过程**

**1.模拟产生消费数据**

上篇博客模拟产生结构化数据，因为生产中很多数据都是json类型的，这次就采用json格式的数据，通过kafka发送给flink进行解析，代码如下:

```
package TopNitems
 
import java.text.SimpleDateFormat
import java.util
import java.util.{Date, Properties}
 
import TopNitems.KafkaProducers.SendtoKafka
import org.apache.kafka.clients.producer.{KafkaProducer, ProducerRecord}
 
import scala.Array.range
import scala.util.Random.shuffle
import com.alibaba.fastjson
import com.alibaba.fastjson.JSON
import com.alibaba.fastjson.serializer.SerializerFeature
 
object KafkaProducerJson {
  def main(args: Array[String]): Unit = {
    SendtoKafka("testken")
  }
 
  def SendtoKafka(topic: String): Unit = {
    val pro = new Properties()
    pro.put("bootstrap.servers", "192.168.226.10:9092")
    //pro.put("bootstrap.servers", "40.73.75.70:9092")
    pro.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    pro.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    val producer = new KafkaProducer[String, String](pro)
    var member_id = range(1, 10)
    var goods = Array("Milk", "Bread", "Rice") //为了尽快显示效果，减少商品品相
    var action = Array("click", "buy")
    //var ts=DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss",Locale.CHINA).format( ZonedDateTime.now())
 
    while (true) {
 
      var ts = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date())
      //var msg = shuffle(member_id.toList).head + "\t" + shuffle(goods.toList).head + "\t" + shuffle(action.toList).head + "\t" + ts + "\t" + "\n"
      var map=new util.HashMap[String,Any]()
      map.put("userid",shuffle(member_id.toList).head)
      map.put("item",shuffle(goods.toList).head)
      map.put("action",shuffle(action.toList).head)
      map.put("times",ts)
 
      val jsons = JSON.toJSONString(map, SerializerFeature.BeanToArray)
      print(jsons+"\n")
      var record = new ProducerRecord[String, String](topic, jsons)
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

效果如下:

可以看到下图方框的顾客1和3已经满足了我们的需求一:

**查看顾客是否点击之后立即购买，是的话输出用户id，购买商品，时间**

![image](https://upload-images.jianshu.io/upload_images/9716180-f7284dba3348f16c?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**2.编码**

**需求一：查看顾客是否点击之后立即购买，是的话输出用户id，购买商品，时间**

首先引入CEP依赖

```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep_2.11</artifactId>
  <version>1.11.2</version>
</dependency>
```

代码：

```
package TopNitems
 
import java.text.SimpleDateFormat
import java.util.Properties
 
import com.alibaba.fastjson.JSON
import org.apache.flink.api.common.serialization.SimpleStringSchema
import org.apache.flink.api.java.tuple.Tuple
import org.apache.flink.cep.PatternSelectFunction
import org.apache.flink.cep.pattern.conditions.SimpleCondition
import org.apache.flink.cep.scala.{CEP, PatternStream}
import org.apache.flink.cep.scala.pattern.Pattern
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.streaming.api.scala.function.{ProcessWindowFunction, WindowFunction}
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.windowing.windows.TimeWindow
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer
import org.apache.flink.util.Collector
 
object CEPTestJson {
  def main(args: Array[String]): Unit = {
    //获得环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1) //设置并发为1，防止打印控制台乱序
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) //Flink 默认使用 ProcessingTime 处理,设置成event time
    //从Kafka读取数据
    val pros = new Properties()
    pros.setProperty("bootstrap.servers", "192.168.226.10:9092")
    //pros.setProperty("bootstrap.servers", "40.73.75.70:6666")
    pros.setProperty("group.id", "testken")
    pros.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
    pros.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
    pros.setProperty("auto.offset.reset", "latest")
    import org.apache.flink.api.scala._
    val dataSource = env.addSource(new FlinkKafkaConsumer[String]("testken", new SimpleStringSchema(), pros))
    val result = dataSource.map(line => {
      val userid=JSON.parseObject(line).getString("userid")
      val item=JSON.parseObject(line).getString("item")
      val action=JSON.parseObject(line).getString("action")
      val times=JSON.parseObject(line).getString("times")
      var time=0L
      try{time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse(times).getTime} //时间戳类型
      catch {case e: Exception => {print( e.getMessage)}}
      (userid.toInt, item.toString,action.toString ,time.toLong)
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[(Int,String, String, Long)](Time.seconds(10)) {
      override def extractTimestamp(t: (Int,String, String, Long)): Long = t._4
    }).map(x=>{(x._1,x._2,x._3,new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(x._4))}) //时间还原成datetime类型
      .keyBy(k=>(k._1,k._2))//把userid和item组合通过做keyby
 
 
    val patterns=Pattern.begin[(Int,String,String,String)]("start").where(new SimpleCondition[(Int, String, String, String)] {
      override def filter(t: (Int, String, String, String)): Boolean = t._3.equals("click")
    }).next("middle").where(new SimpleCondition[(Int, String, String, String)] {
      override def filter(t: (Int, String, String, String)): Boolean = t._3.equals("buy")
    }).within(Time.seconds(20))
    //把pattern应用于普通流
 
    val ptstream: PatternStream[(Int,String,String,String)]=CEP.pattern(result,patterns)
    ptstream.select(new PatternSelectFunction[(Int,String,String,String),String] {
      override def select(map: java.util.Map[String, java.util.List[(Int, String, String, String)]]): String = {
        val click: (Int, String, String, String) = map.get("start").iterator().next()
        val buy: (Int, String, String, String) = map.get("middle").iterator().next()
        // 打印用户的名称，点击和购买的时间
        s"name: ${click._1}, item: ${click._2}, clicktime: ${click._4}, buytime: ${buy._4}"
      }
    }).print()
 
    env.execute("CEPTestJson")
    }
}
```

效果如下:

可以看到顾客1和3的数据已经被打印出来了，满足我们的需求

![image](https://upload-images.jianshu.io/upload_images/9716180-6e391feebea12a34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**需求二:如果同一个顾客买了5次牛奶，输出用户id，时间**

代码部分只需要更改一下pattern就可以了

```
 val patterns=Pattern.begin[(Int,String,String,String)]("start").where(new SimpleCondition[(Int, String, String, String)] {
      override def filter(t: (Int, String, String, String)): Boolean = t._2.equals("Milk")
    }).where(new SimpleCondition[(Int, String, String, String)] {
      override def filter(t: (Int, String, String, String)): Boolean = t._3.equals("buy")
    }).timesOrMore(3)
    //把pattern应用于普通流
 
    val ptstream: PatternStream[(Int,String,String,String)]=CEP.pattern(result,patterns)
    ptstream.select(new PatternSelectFunction[(Int,String,String,String),String] {
      override def select(map: java.util.Map[String, java.util.List[(Int, String, String, String)]]): String = {
        val info: (Int, String, String, String) = map.get("start").iterator().next()
        // 打印用户的名称，产品和购买的时间
        s"name: ${info._1}, item: ${info._2}, buytime: ${info._4}"
      }
    }).print()
```

效果如下

![image](https://upload-images.jianshu.io/upload_images/9716180-7ec63cf5c039f404.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**结果集存入MySQL**

首先在myql数据库建表

```
create table cep(name int,item char(10),clicktime datetime,buytime datetime);
```

代码部分其实就添加一个sink，把我上篇博客的Clickhouse改成mysql，理论上可以更改成任意支持JDBC的数据库。

完整代码如下:

```
package TopNitems
 
import java.sql.PreparedStatement
import java.text.SimpleDateFormat
import java.util.Properties
 
import com.alibaba.fastjson.JSON
import org.apache.flink.api.common.serialization.SimpleStringSchema
import org.apache.flink.api.java.tuple.Tuple
import org.apache.flink.cep.PatternSelectFunction
import org.apache.flink.cep.pattern.conditions.SimpleCondition
import org.apache.flink.cep.scala.{CEP, PatternStream}
import org.apache.flink.cep.scala.pattern.Pattern
import org.apache.flink.connector.jdbc.{JdbcConnectionOptions, JdbcExecutionOptions, JdbcSink, JdbcStatementBuilder}
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.streaming.api.scala.function.{ProcessWindowFunction, WindowFunction}
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.windowing.windows.TimeWindow
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer
import org.apache.flink.util.Collector
 
class MysqlSinkBuilder extends JdbcStatementBuilder[(Int, String, String,String)] {
  def accept(ps: PreparedStatement, v: (Int, String, String,String)): Unit = {
    ps.setInt(1, v._1)
    ps.setString(2, v._2)
    ps.setString(3, v._3)
    ps.setString(4, v._4)
  }
}
 
object CEPTestJson {
  def main(args: Array[String]): Unit = {
    //获得环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1) //设置并发为1，防止打印控制台乱序
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) //Flink 默认使用 ProcessingTime 处理,设置成event time
    //从Kafka读取数据
    val pros = new Properties()
    pros.setProperty("bootstrap.servers", "192.168.226.10:9092")
    //pros.setProperty("bootstrap.servers", "40.73.75.70:6666")
    pros.setProperty("group.id", "testken")
    pros.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
    pros.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")
    pros.setProperty("auto.offset.reset", "latest")
    import org.apache.flink.api.scala._
    val dataSource = env.addSource(new FlinkKafkaConsumer[String]("testken", new SimpleStringSchema(), pros))
    val result = dataSource.map(line => {
      val userid=JSON.parseObject(line).getString("userid")
      val item=JSON.parseObject(line).getString("item")
      val action=JSON.parseObject(line).getString("action")
      val times=JSON.parseObject(line).getString("times")
      var time=0L
      try{time = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").parse(times).getTime} //时间戳类型
      catch {case e: Exception => {print( e.getMessage)}}
      (userid.toInt, item.toString,action.toString ,time.toLong)
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[(Int,String, String, Long)](Time.seconds(10)) {
      override def extractTimestamp(t: (Int,String, String, Long)): Long = t._4
    }).map(x=>{(x._1,x._2,x._3,new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(x._4))}) //时间还原成datetime类型
      .keyBy(k=>(k._1,k._2))//把userid和item组合通过做keyby
 
 
    val patterns=Pattern.begin[(Int,String,String,String)]("start").where(new SimpleCondition[(Int, String, String, String)] {
      override def filter(t: (Int, String, String, String)): Boolean = t._3.equals("click")
    }).next("middle").where(new SimpleCondition[(Int, String, String, String)] {
      override def filter(t: (Int, String, String, String)): Boolean = t._3.equals("buy")
    }).within(Time.seconds(20))
    //把pattern应用于普通流
 
    val ptstream: PatternStream[(Int,String,String,String)]=CEP.pattern(result,patterns)
    val cepresult=ptstream.select(new PatternSelectFunction[(Int,String,String,String),String] {
      override def select(map: java.util.Map[String, java.util.List[(Int, String, String, String)]]): String = {
        val click: (Int, String, String, String) = map.get("start").iterator().next()
        val buy: (Int, String, String, String) = map.get("middle").iterator().next()
        // 打印用户的名称，点击和购买的时间
        s"${click._1},${click._2},${click._4},${buy._4}"
      }
    })
    cepresult.print()
    val cepout=cepresult.map(line=>{
      val x=line.split(",")
      val name=x(0)
      val item=x(1)
      val clicktime=x(2)
      val buytime=x(3)
      (name.toInt,item.toString,clicktime.toString,buytime.toString)
    })
      //cepout.print()
      cepout.addSink(JdbcSink.sink[(Int,String,String,String)]("insert into test.cep(name,item,clicktime,buytime)values(?,?,?,?)"
      ,new MysqlSinkBuilder,new JdbcExecutionOptions.Builder().withBatchSize(2).build(),
      new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
        .withUrl("jdbc:mysql://localhost:3306/test")
        .withDriverName("com.mysql.jdbc.Driver")
        .withUsername("root")
        .withPassword("123456")
        .build()
    ))
 
 
    env.execute("CEPTestJson")
    }
}
```

效果：

![image](https://upload-images.jianshu.io/upload_images/9716180-e25e7775b0d40554?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
