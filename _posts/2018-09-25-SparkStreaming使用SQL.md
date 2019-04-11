例子来源于官网的wordcount例子
```  

package Sparkstreaming
 
import org.apache.spark.SparkConf
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.SparkSession
import org.apache.spark.storage.StorageLevel
import org.apache.spark.streaming.{Seconds, StreamingContext, Time}
 
object SQLtest {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("SQLtest").setMaster("local[2]")
    val ssc = new StreamingContext(conf, Seconds(5))
    val lines = ssc.socketTextStream("192.168.116.10", 9999)
    val words = lines.flatMap(_.split(" "))
    words.foreachRDD { (rdd: RDD[String], time: Time) =>
      // Get the singleton instance of SparkSession
      val spark = SparkSessionSingleton.getInstance(rdd.sparkContext.getConf)
      import spark.implicits._
 
      // Convert RDD[String] to RDD[case class] to DataFrame
      val wordsDataFrame = rdd.map(w => Record(w)).toDF()
 
      // Creates a temporary view using the DataFrame
      wordsDataFrame.createOrReplaceTempView("words")
 
      // Do word count on table using SQL and print it
      val wordCountsDataFrame =
        spark.sql("select word, count(*) as total from words group by word")
      println(s"========= $time =========")
      wordCountsDataFrame.show()
    }
    ssc.start()
    ssc.awaitTermination()
 
  }
 
  case class Record(word: String)
 
 
  /** Lazily instantiated singleton instance of SparkSession */
  object SparkSessionSingleton {
 
    @transient private var instance: SparkSession = _
 
    def getInstance(sparkConf: SparkConf): SparkSession = {
      if (instance == null) {
        instance = SparkSession
          .builder
          .config(sparkConf)
          .getOrCreate()
      }
      instance
    }
  }
 
}
```  
测试

在Linux新建一个窗口，输入aaaa bbbb aaaa

可以发现IDEA控制台已经输出结果了
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume3.png) 
