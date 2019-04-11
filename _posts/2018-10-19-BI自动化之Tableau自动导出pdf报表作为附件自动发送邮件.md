# 一、目的：

1.每天定时从tableau导出pdf报表

2.每天自动定时发送邮件



# 二、实现的过程：

1.首先利用windows自带的记事本写好bat批处理文件，更多tableau可以看tableau cmd命令，本地新建记事本，贴代码如下，然后改后缀bat：
```  
set path=C:\Program Files\Tableau\Tableau Server\10.5\bin
tabcmd login -s http://服务器ip:服务器端口 -u 用户名 -p 密码
del C:\pdf\*.pdf -y
del C:\pdf\*.png -y
tabcmd export "Deli/DailySales" --png -f "C:\pdf\DailySales.png"
tabcmd export "Deli/DailySales" --pdf -f "C:\pdf\DailySales.pdf"
```  

 
这样就可以在本地c盘的pdf文件夹看到生成的pdf和png附件了。

检查数据库数量是否正确
```  
@echo off
osql -U 用户明  -P 密码 -S 17.服务器ip -d 数据库 -Q "select * from S where *" >1.txt
for /f "skip=2 skip=1 eol=(" %%i in (1.txt) do set b= %%i
set /a num1=1860
if %num1% LSS %b%  echo.>>C:\pdf\output.txt
del 1.txt

```  

2.利用vbs文件发送邮件。关于vbs，还有很好玩的，可以看看如何恶搞朋友的电脑。进入正题，本地新建记事本，贴下面的代码，改后缀vbs：


```  
Set objShell = CreateObject("Wscript.Shell")
SET objFSO = CreateObject("Scripting.FileSystemObject")


SDate = DATE()-1

StrArray = SPLIT(CStr(SDate),"/")
SYear = StrArray(0)
SMonth = StrArray(1)
SDay = StrArray(2)

IF SMonth < 10 THEN SMonth = "0"&SMonth
IF SDay < 10 THEN SDay = "0"&SDay
StrDate = SYear&SMonth&SDay

EXPORTPath = "c:\pdf\"
SENDPath = """C:\pdf\his\"""

objShell.Run("CMD /c COPY /V /Y "&EXPORTPath&"DailySales.png "&SENDPath&"DailySales_"&StrDate&".png")
objShell.Run("CMD /c COPY /V /Y "&EXPORTPath&"DailySales.pdf "&SENDPath&"DailySales_"&StrDate&".pdf")
objShell.Run("CMD /c COPY /V /Y "&EXPORTPath&"Store.pdf "&SENDPath&"Store_"&StrDate&".pdf")
objShell.Run("CMD /c COPY /V /Y "&EXPORTPath&"StorePercentage.pdf "&SENDPath&"StorePercentage_"&StrDate&".pdf")

Wscript.Sleep 3000

Dim strHTML

strHTML = "<HTML>"
strHTML = strHTML & "<HEAD>"
strHTML = strHTML & "<BODY>"
strHTML = strHTML & "<span style=font-size:9pt;font-family:Tahoma,Calibri,sans-serif;color:black>"
strHTML = strHTML & "===Please do not reply this mail directly, it is a system generated mail=== <p>"
strHTML = strHTML & "<span style=font-size:10pt;font-family:Tahoma,Calibri,sans-serif;color:black>"
strHTML = strHTML & "Dear all ,</span><p>"
strHTML = strHTML & "<span style=font-size:10pt;font-family:Tahoma,Calibri,sans-serif;color:black>"
strHTML = strHTML & "Below " & SDate & " Thanks! </span>"
strHTML = strHTML & "<span style=font-size:10pt;font-family:Tahoma,Calibri,sans-serif;color:black>"
strHTML = strHTML & "<img src=http://xxxxx/mail/DailySales_"&StrDate&".png />"
strHTML = strHTML & "</BODY>"
strHTML = strHTML & "</HTML>"

Set myfile = objFSO.openTextFile("c:\pdf\mail.txt", 1, false)

While Not myfile.AtEndOfLine                                   
mailto=myfile.readline


If objFSO.FileExists("c:\pdf\etl_output_delivery.txt") Then sendrpt()

On Error Resume Next 

Wend


Sub sendrpt()
  Set objEmail = CreateObject("CDO.Message")
  objEmail.From ="report@xxx.com"
  objEmail.To = mailto
  objEmail.Subject = "testemail  report " & StrDate
  objEmail.HTMLbody = strHTML

  objEmail.AddAttachment "C:\pdf\his\DailySalesbyRegionDelivery_" & StrDate & ".pdf"
  objEmail.AddAttachment "C:\pdf\his\StoreDetailsDelivery_" & StrDate & ".pdf"
  objEmail.AddAttachment "C:\pdf\his\StoreDetailDeliveryPercentage_" & StrDate & ".pdf"




  Const schema = "http://schemas.microsoft.com/cdo/configuration/"  

  With objEmail.Configuration.Fields  
   .Item(schema & "sendusing") = 2  
   .Item(schema & "smtpserver") = "smtp.xxx.cn" 
   .Item(schema & "smtpauthenticate") = 1  
   .Item(schema & "sendusername") = "report@xxx.xx" 
   .Item(schema & "sendpassword") = "密码" 
   .Item(schema & "smtpserverport") = 25  
   .Item(schema & "smtpusessl") = False
   .Item(schema & "smtpconnectiontimeout") = 60  
   .Update  
  End With 

  objEmail.Send 

End Sub
```  

3.利用windows自带的server managers发送邮件。

我的电脑-右键-管理，找到任务计划程序-任务计划程序库
右键，新建任务，可以在触发器这里设置自动更新的频率：
![image](https://raw.githubusercontent.com/lbship/lbship.github.io/master/img/hadoop2.6/flume8.png) 
在操作这里可以设置我们刚才创建的bat和vbs文件自启动，到这里就大功告成了。
接下里就每天喝喝茶，看看邮件就行，双手全解放了。
