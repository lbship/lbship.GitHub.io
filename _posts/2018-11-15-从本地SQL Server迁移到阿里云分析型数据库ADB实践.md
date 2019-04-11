# 一、数据库官网介绍：  

  分析型数据库（AnalyticDB），是阿里巴巴自主研发的海量数据实时高并发在线分析（Realtime OLAP）云计算服务，使得您可以在毫秒级针对千亿级数据进行即时的多维分析透视和业务探索。分析型数据库对海量数据的自由计算和极速响应能力，能让用户在瞬息之间进行灵活的数据探索，快速发现数据价值，并可直接嵌入业务系统为终端客户提供分析服务

整套架构如图所示:
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads1.png) 


阿里云ADB提供了多种数据迁移方式，可以通过DataWorks调度OSS，Local DB、Hive等多种资源。

本次实践分为两大部分，历史数据同步（大约10亿条）和增量数据导入。同时有一部分数据源是Excel形式存在的，直接利用Kettle进行传输。

# 二、历史数据同步

- 1.在ADB建表

有两种方式建表，一种是可视化，一种是命令的。语法有点像HQL的语法，默认设置128个分区，需要设置组名，把表进行分组。更新方式设置为实时更新
```  
 CREATE TABLE 
库名.表名 (
unknown1 varchar , 
unknown2 varchar , 
unknown3 varchar , 
unknown4 varchar , 
unknown5 varchar , 
primary key (字段1，字段2)
), 
PARTITION BY HASH KEY(字段1) PARTITION NUM 128
TABLEGROUP 组名
OPTIONS(UPDATETYPE='realtime'),
```  
- 2.开通VPC专线

一种是可视化操作，另一种是命令，先登录数据库：

在安装MySQL服务的电脑上，输入CMD命令登录数据库
Mysql： -h域名 -P端口 -uAccessKey -pAccessKeySecret -D数据库名 -A –c
域名和端口在ADB控制台查看。 AccesKey和secret找管理员要。
然后输入命令,即可开通vpc，不同区域的id不一样，可以联系阿里云人员要：

ALTER DATABASE 库名 
SET zone_id='cn-xxxxx' 
    vpc_id='vpc-xxxxxx' 
vswitch_id='vsw-xxxxx';
- 3.通过DataWorks导入历史数据：

a.开通Dataworks（由管理员操作）

开通方法：https://help.aliyun.com/document_detail/65311.html?spm=a2c4g.11186623.6.549.644460efPdBvME

添加权限：https://help.aliyun.com/document_detail/58185.html?spm=a2c4g.11186623.6.551.4424352ed3cpkT

账号信息更新：https://help.aliyun.com/document_detail/30262.html?spm=a2c4g.11186623.6.552.34b6352e4CUxnB

配置子账号：https://help.aliyun.com/document_detail/55316.html?spm=a2c4g.11186623.6.553.696d591a72ZD3i

b.配置OSS数据源：

进入Dataworks的数据集成，新建数据源
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads2.png) 


填入endpoint，

Endpoint配置参考：

https://help.aliyun.com/document_detail/31837.html?spm=a2c4g.11186623.6.572.49661c62kbQsIM 

Endpoint分内网和外网，使用内网速度会比较快。不用测试连通性，专用网络无法测试。bucket就是OSS的bucket名称。  

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads3.png) 


c.配置ADB数据源

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads4.png) 

d.配置同步任务

参考网址:

https://help.aliyun.com/document_detail/100640.html?spm=a2c4g.11186623.2.22.220a7793M8B9Af

进入DataWorks控制台，单击对应项目操作栏中的数据开发, 新建业务流程

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads5.png) 
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads6.png) 


创建完成之后，双击中创建的节点，配置数据同步任务的数据来源（Reader）、数据去向（Writer）、字段映射、通道控制信息，然后保存和提交，然后运行，就完成了历史数据的同步，下方有日志可以查看。
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads7.png) 


# 三、增量数据同步

- 1.新增本地SQL server数据源

这里我是使用了有公网ip的数据源，如果3306端口有限制，需要对阿里云ip进行授权访问。如果没有公网ip的数据源，就是纯内网的，可以找一台有公网和内网访问的机器作为桥接，设置为共享资源组就行了。

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads8.png) 

- 2.配置同步任务

操作方法类似oss同步，可以添加过滤调节，下面的切分键可以分批查询本地数据源，提高数据传输效率。

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/ads/ads9.png) 

提交之后，可以到运行中心调度任务，设置每天同步时间



这样，整套历史数据传输和增量任务传输就完成了。总体感觉还是比较流畅的
