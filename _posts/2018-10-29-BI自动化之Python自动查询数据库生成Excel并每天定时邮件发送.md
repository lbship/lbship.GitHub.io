# 一、目的

1.每天自动查询SQL数据

2.生成Excel并作为附件邮件发送

3.每天定时自动执行

# 二、开发环境

1.Python3.5

2.SQL server2014

# 三、代码

一两个小时弄的，代码可能有点乱，凑合着看吧
```  
 
import pymssql
import pandas as pd
import time,datetime
import smtplib
import traceback
from email.mime.text import MIMEText
from email.mime.application import  MIMEApplication
from email.mime.multipart import MIMEMultipart
 
 
 
#连接数据库
def get_db_connection():
    host = 'xxx.xxxx.xxx'
    port = 1433
    user = 'xxxx'
    password = 'xxxxx'
    database = 'xxxx'
    conn = pymssql.connect(host=host,user=user,password=password,database=database,charset="utf8")
    print('数据库连接成功')
    return conn
#查询SQL结果并生成execl
def get_execl(filepath):
    sql= '''select xxxx,sum(xx) as SALES 
     from xxx a 	left join xxxx b	
     on a.xxx=b.xxxx	
     where (Date=convert(date,GETDATE()-1)) 
     group by xxxx'''
    conn=get_db_connection()
    #直接使用pandas的read_sql即可转化成DataFrame
    df=pd.read_sql(sql,conn)
    #使用解析的方法（列表+header）
    # column=cur.description
    # columns=[column[i][0] for i in range(len(column))] #利用光标获得列名
    # df=pd.DataFrame([list(i)for i in data],columns=columns)
    df.to_excel(filepath)
    print('已经生成Excel文件')
#发送邮件
def sendmail(subject, msg, to_addrs, from_addr, smtp_addr, password,filepath):
    mail_msg = MIMEMultipart()
    mail_msg['Subject'] = subject
    mail_msg['From'] = from_addr
    mail_msg['To'] = ','.join(to_addrs)
    msg['Cc'] = 'xxx@qq.com'
    mail_msg.attach(MIMEText(msg, 'html', 'utf-8'))
    part1 = MIMEApplication(open(filepath, 'rb').read())
    part1.add_header('Content-Disposition', 'attachment', filename=('bychannel.xlsx'))
    mail_msg.attach(part1)
    try:
        s = smtplib.SMTP()
        s.connect(smtp_addr)
        s.login(from_addr, password)
        s.sendmail(from_addr, to_addrs, mail_msg.as_string())
        s.quit()
    except Exception:
        print('error')
        print(traceback.format_exc())
if __name__ == '__main__':
    curdate = time.strftime('%Y%m%d',time.localtime(time.time()))
    filepath = 'D:/bychannel{}.xlsx'.format(curdate)
    get_execl(filepath)
    from_addr = 'xxx@163.com'
    smtp_addr = 'smtp.163.com'
    to_addrs = 'xxxx@xxxx','xxx@qq.com'
    subject = (curdate) + 'report  by channel'
    password = 'xxxxxxx'
    msg = '===\nPlease do not reply this mail directly,\n it is a system generated mail\n===\n '
    sendmail(subject, msg, to_addrs, from_addr, smtp_addr, password,filepath)
 
 
 ```  
 
# 四、测试效果

![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume7.png) 



# 五、设置每天定时执行

1.新建一个bat文件，输入调用脚本

python mssql.py

2.利用Windows自带的计划任务每天定时调用这个脚本

我的电脑邮件-管理-任务计划程序，设置一下操作和触发器等，这样就可以每天定时执行了
