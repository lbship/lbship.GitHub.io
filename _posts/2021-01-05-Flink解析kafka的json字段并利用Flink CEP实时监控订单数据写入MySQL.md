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

**一、pattern**

pattern包含单个模式、组合模式和模式组，常用的Flink CEP API如下,来自官网:

单个模式API:

| **where(condition)** | 为当前模式定义一个条件。为了匹配这个模式，一个事件必须满足某些条件。 多个连续的where()语句取与组成判断条件：pattern.where(event => ... /* 一些判断条件 */) |
| **or(condition)** | 增加一个新的判断，和当前的判断取或。一个事件只要满足至少一个判断条件就匹配到模式：pattern.where(event => ... /* 一些判断条件 */).or(event => ... /* 替代条件 */) |
| **until(condition)** | 为循环模式指定一个停止条件。意思是满足了给定的条件的事件出现后，就不会再有事件被接受进入模式了。只适用于和oneOrMore()同时使用。提示： 在基于事件的条件中，它可用于清理对应模式的状态。pattern.oneOrMore().until(event => ... /* 替代条件 */) |
| **subtype(subClass)** | 为当前模式定义一个子类型条件。一个事件只有是这个子类型的时候才能匹配到模式：pattern.subtype(classOf[SubEvent]) |
| **oneOrMore()** | 指定模式期望匹配到的事件至少出现一次。.默认（在子事件间）使用松散的内部连续性。 关于内部连续性的更多信息可以参考连续性。提示： 推荐使用until()或者within()来清理状态。pattern.oneOrMore() |
| **timesOrMore(#times)** | 指定模式期望匹配到的事件至少出现#times次。.默认（在子事件间）使用松散的内部连续性。 关于内部连续性的更多信息可以参考连续性。pattern.timesOrMore(2) |
| **times(#ofTimes)** | 指定模式期望匹配到的事件正好出现的次数。默认（在子事件间）使用松散的内部连续性。 关于内部连续性的更多信息可以参考连续性。pattern.times(2) |
| **times(#fromTimes, #toTimes)** | 指定模式期望匹配到的事件出现次数在#fromTimes和#toTimes之间。默认（在子事件间）使用松散的内部连续性。 关于内部连续性的更多信息可以参考连续性。pattern.times(2, 4) |
| **optional()** | 指定这个模式是可选的，也就是说，它可能根本不出现。这对所有之前提到的量词都适用。pattern.oneOrMore().optional() |
| **greedy()** | 指定这个模式是贪心的，也就是说，它会重复尽可能多的次数。这只对量词适用，现在还不支持模式组。pattern.oneOrMore().greedy() |

组合模式API：

| **consecutive()** | 与oneOrMore()和times()一起使用， 在匹配的事件之间施加严格的连续性， 也就是说，任何不匹配的事件都会终止匹配（和next()一样）。如果不使用它，那么就是松散连续（和followedBy()一样）。例如，一个如下的模式：Pattern.begin("start").where(_.getName().equals("c"))  .followedBy("middle").where(_.getName().equals("a"))                       .oneOrMore().consecutive()  .followedBy("end1").where(_.getName().equals("b"))输入：C D A1 A2 A3 D A4 B，会产生下面的输出：如果施加严格连续性： {C A1 B}，{C A1 A2 B}，{C A1 A2 A3 B}不施加严格连续性： {C A1 B}，{C A1 A2 B}，{C A1 A2 A3 B}，{C A1 A2 A3 A4 B} |
| **allowCombinations()** | 与oneOrMore()和times()一起使用， 在匹配的事件中间施加不确定松散连续性（和followedByAny()一样）。如果不使用，就是松散连续（和followedBy()一样）。例如，一个如下的模式：Pattern.begin("start").where(_.getName().equals("c"))  .followedBy("middle").where(_.getName().equals("a"))                       .oneOrMore().allowCombinations()  .followedBy("end1").where(_.getName().equals("b"))输入：C D A1 A2 A3 D A4 B，会产生如下的输出：如果使用不确定松散连续： {C A1 B}，{C A1 A2 B}，{C A1 A3 B}，{C A1 A4 B}，{C A1 A2 A3 B}，{C A1 A2 A4 B}，{C A1 A3 A4 B}，{C A1 A2 A3 A4 B}如果不使用：{C A1 B}，{C A1 A2 B}，{C A1 A2 A3 B}，{C A1 A2 A3 A4 B} |

模式组API：

| **begin(#name)** | 定一个开始模式：val start = Pattern.begin[Event]("start") |
| **begin(#pattern_sequence)** | 定一个开始模式：val start = Pattern.begin(    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)) |
| **next(#name)** | 增加一个新的模式，匹配的事件必须是直接跟在前面匹配到的事件后面（严格连续）：val next = start.next("middle") |
| **next(#pattern_sequence)** | 增加一个新的模式。匹配的事件序列必须是直接跟在前面匹配到的事件后面（严格连续）：val next = start.next(    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)) |
| **followedBy(#name)** | 增加一个新的模式。可以有其他事件出现在匹配的事件和之前匹配到的事件中间（松散连续）：val followedBy = start.followedBy("middle") |
| **followedBy(#pattern_sequence)** | 增加一个新的模式。可以有其他事件出现在匹配的事件和之前匹配到的事件中间（松散连续）：val followedBy = start.followedBy(    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)) |
| **followedByAny(#name)** | 增加一个新的模式。可以有其他事件出现在匹配的事件和之前匹配到的事件中间， 每个可选的匹配事件都会作为可选的匹配结果输出（不确定的松散连续）：val followedByAny = start.followedByAny("middle") |
| **followedByAny(#pattern_sequence)** | 增加一个新的模式。可以有其他事件出现在匹配的事件序列和之前匹配到的事件中间， 每个可选的匹配事件序列都会作为可选的匹配结果输出（不确定的松散连续）：val followedByAny = start.followedByAny(    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)) |
| **notNext()** | 增加一个新的否定模式。匹配的（否定）事件必须直接跟在前面匹配到的事件之后 （严格连续）来丢弃这些部分匹配：val notNext = start.notNext("not") |
| **notFollowedBy()** | 增加一个新的否定模式。即使有其他事件在匹配的（否定）事件和之前匹配的事件之间发生， 部分匹配的事件序列也会被丢弃（松散连续）：val notFollowedBy = start.notFollowedBy("not") |
| **within(time)** | 定义匹配模式的事件序列出现的最大时间间隔。如果未完成的事件序列超过了这个事件，就会被丢弃：pattern.within(Time.seconds(10)) |

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
