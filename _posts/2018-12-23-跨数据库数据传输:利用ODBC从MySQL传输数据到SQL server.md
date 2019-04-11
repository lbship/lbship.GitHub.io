要做数据库迁移和增量备份，把MySQL数据每天移动到SQL server中

1.设置ODBC工具

首先电脑要先安装好 MySQL的ODBC connector，百度一下就可以了。安装完成之后，在控制面板的ODBC数据源管理里面就可以看到了。

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/O1.png) 

填好相关的连接信息。记住这个datasource name，等会儿要用
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/O2.png)
2.在SQL server中设置link

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/O3.png)
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/O4.png)


3.测试

SELECT * into xxx..temp1 FROM OPENQUERY(MYSQL, 'select * from channel' ) 

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/O5.png)

测试成功
