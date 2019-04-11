#一、POM配置

因为使用windows的IDEA连接虚拟机中的Spark，所有要配置一下依赖  
```  

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.imooc.spark</groupId>
  <artifactId>sparktrain</artifactId>
  <version>1.0</version>
  <inceptionYear>2008</inceptionYear>
  <properties>
    <scala.version>2.11.4</scala.version>
    <kafka.version>1.0.0</kafka.version>
    <spark.version>2.4.0</spark.version>
    <hadoop.version>2.6.1</hadoop.version>
    <hbase.version>1.2.6</hbase.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>org.scala-lang</groupId>
      <artifactId>scala-library</artifactId>
      <version>${scala.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka_2.11</artifactId>
      <version>1.0.0</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hadoop</groupId>
      <artifactId>hadoop-client</artifactId>
      <version>${hadoop.version}</version>
    </dependency>
    <dependency>
      <groupId>io.netty</groupId>
      <artifactId>netty-all</artifactId>
      <version>4.1.17.Final</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.module</groupId>
      <artifactId>jackson-module-scala_2.11</artifactId>
      <version>2.9.1</version>
    </dependency>
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.31</version>
    </dependency>
    <dependency>
      <groupId>org.apache.hbase</groupId>
      <artifactId>hbase-client</artifactId>
      <version>${hbase.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.spark</groupId>
      <artifactId>spark-streaming_2.11</artifactId>
      <version>2.4.0</version>
    </dependency>
  </dependencies>
 
  <build>
    <sourceDirectory>src/main/scala</sourceDirectory>
    <testSourceDirectory>src/test/scala</testSourceDirectory>
    <plugins>
      <plugin>
        <groupId>org.scala-tools</groupId>
        <artifactId>maven-scala-plugin</artifactId>
        <executions>
          <execution>
            <goals>
              <goal>compile</goal>
              <goal>testCompile</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <scalaVersion>${scala.version}</scalaVersion>
          <args>
            <arg>-target:jvm-1.5</arg>
          </args>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-eclipse-plugin</artifactId>
        <configuration>
          <downloadSources>true</downloadSources>
          <buildcommands>
            <buildcommand>ch.epfl.lamp.sdt.core.scalabuilder</buildcommand>
          </buildcommands>
          <additionalProjectnatures>
            <projectnature>ch.epfl.lamp.sdt.core.scalanature</projectnature>
          </additionalProjectnatures>
          <classpathContainers>
            <classpathContainer>org.eclipse.jdt.launching.JRE_CONTAINER</classpathContainer>
            <classpathContainer>ch.epfl.lamp.sdt.launching.SCALA_CONTAINER</classpathContainer>
          </classpathContainers>
        </configuration>
      </plugin>
    </plugins>
  </build>
  <reporting>
    <plugins>
      <plugin>
        <groupId>org.scala-tools</groupId>
        <artifactId>maven-scala-plugin</artifactId>
        <configuration>
          <scalaVersion>${scala.version}</scalaVersion>
        </configuration>
      </plugin>
    </plugins>
  </reporting>
</project>
 
 ```  

二、实时接收网络数据

1.代码
```  

package Sparkstreaming
 
import java.sql.DriverManager
 
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
 
 
object NetWorkCount {
  def main(args: Array[String]): Unit = {
    val conf=new SparkConf().setAppName("NerWorkCount").setMaster("local[2]")
    val ssc =new StreamingContext(conf,Seconds(5))
    val lines=ssc.socketTextStream("192.168.116.10",9999)
    val result=lines.flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_)
    //System.setProperty("hadoop.home.dir","E:\\MVN\\hadoop-common-2.2.0-bin-master")
    //相当于这个写法 reduceByKey((x,y) => x+y)当找到key相同的两条记录时会对其value(分别记为x,y)做(x,y) => x+y
    result.print()
    result.foreachRDD { rdd =>
      rdd.foreachPartition { partitionOfRecords =>
        val connection = createconnection()
        partitionOfRecords.foreach(record =>{
          val sql="insert into wordcount values('"+record._1+"',"+record._2+")"
          connection.createStatement().execute(sql)
        }
         )
        connection.close()
      }
    }
    ssc.start()
    ssc.awaitTermination()
    //awaitTermination用于等待子线程结束，再继续执行下面的代码
 
  }
  def createconnection()={
    Class.forName("com.mysql.jdbc.Driver")
    DriverManager.getConnection("jdbc:mysql://192.168.116.10:3306/test","root","123456")
  }
 
}  
```  
2.测试

在虚拟机中开启新开一个窗口，输入nc -lk 6789

然后运行IDEA的spark源码，随便输入几个单词，可以发现IDEA已经显示出来了。





3.报错处理。

在Windows运行的时候，可能会报错 Failed to locate the winutils binary in the hadoop binary path，可以到GitHub下载整个bin目录，然后修改本机的环境变量。在cmd中测试一下输入hadoop看看环境是否设置成功。

三、实时接收本地hdfs数据

1.代码
```  
package Sparkstreaming
 
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkConf, SparkContext}
 
object Filewordcount {
  def main(args: Array[String]): Unit = {
    val conf=new SparkConf().setAppName("Filewordcont").setMaster("local")
    val ssc=new StreamingContext(conf,Seconds(5))
    val lines=ssc.textFileStream("hdfs://192.168.116.10:9000/sqoop/hdfs/")
    val result=lines.flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_)
    result.print()
    ssc.start()
    ssc.awaitTermination()
  }
 
}
```  
四、做名单过滤

1.代码
```  
package Sparkstreaming
 
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
 
object Filtername {
  def main(args: Array[String]): Unit = {
 
    val conf = new SparkConf().setAppName("Filewordcont").setMaster("local[2]")
    val ssc = new StreamingContext(conf, Seconds(5))
    //名单过滤之创建名单
    val names=List("good","better")
    val namesRDD=ssc.sparkContext.parallelize(names).map((_,true))
    val lines = ssc.socketTextStream("192.168.116.10", 6789)
    val result = lines.map(x=>(x.split(",")(1),x)).transform(rdd=>rdd.leftOuterJoin(namesRDD)).
      filter(x=>x._2._2.getOrElse("good")!="good").map(x=>x._2._1)
    result.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```   
2.测试

在控制台输入2017,now  2018,good可以发现只有2018，good打印出来而已
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume2.png)  
