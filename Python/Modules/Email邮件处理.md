[TOC]

# Python之Email邮件处理

SMTP是一组用于由源地址到目的地址传送邮件的规则，用于控制邮件信息的中转方式。为此，python提供了smtplib包来发送电子邮件，它对smtp协议进行了简单封装。

## 本地SMTP

通过本地SMTP服务可以自定义一个邮件发送者来发送邮件，这种情况无需提供邮箱服务器，但可能被拦截或认定为垃圾邮件

```
import smtplib
from email.mime.text import MIMEText
from email.header import Header

# 自定义邮箱发送者
sender = "test_user@test.com"
# 邮箱接收者，多个邮箱用逗号分隔
receivers = ['abc@test.com']

# 邮件内容
context = MIMEText('该邮件为测试邮件，请忽略','plain','utf-8')
context['From'] = Header("发送者", 'utf-8')
context['To'] =  Header("接收者", 'utf-8')
# 设置邮件标题
context['subject'] = Header("测试邮件",'utf-8')

# 发送邮件
try:
    smtpObj = smtplib.SMTP('localhost')
    smtpObj.sendmail(sender, receivers, context.as_string())
    print "邮件发送成功"
except smtplib.SMTPException:
    print "Error: 无法发送邮件"
```

## 第三方邮件服务器

当我们想要通过第三方邮件服务商的服务来发送邮件，需要先获取到smtp服务地址，用户，口令

```
import smtplib
from email.mime.text import MIMEText
from email.header import Header
from email.mime.multipart import MIMEMultipart

# 设置SMTP服务配置
mail = smtplib.SMTP_SSL('smtphm.qiye.163.com',465)
mail.login('username','password')
# 邮箱接收者，多个邮箱用逗号分隔
receivers = ['abc@test.com']
message = MIMEText('该邮件为测试邮件，请忽略','plain','utf-8')
message['Subject'] = Header('slowlog check info','utf-8').encode()
message['From'] = Header("发送者", 'utf-8')
message['To'] = Header("接收者", 'utf-8')
try:
    mail.sendmail(sender,receivers,message.as_string())
    print("邮件发送成功")
except smtplib.SMTPException:
    print ("Error: 无法发送邮件")
```

## 带附件的邮件

有时我们需要发送带附件的邮件，可以利用MIMEMultipart来实现

```
def sendMail():
    mail = smtplib.SMTP_SSL('smtphm.qiye.163.com',465)
    mail.login('username','password')
    # 自定义邮箱发送者
    sender = "test_user@test.com"
    # 邮箱接收者，多个邮箱用逗号分隔
    receivers = ['abc@test.com']
    message=MIMEMultipart()
    message['Subject'] = Header('slowlog check info','utf-8').encode()
    message['From'] = 'test_user@test.com'
    message['To'] = 'abc@test.com'
    # 正文内容
    message.attach(MIMEText('附件为slowlog check info，请查收!', 'html', 'utf-8'))
    # 构建附件
    files = MIMEApplication(open('/data/luhx/DBCheck/result.html', 'rb').read())
    # 邮件中显示的附件名
    files.add_header('Content-Disposition', 'attachment', filename="result.html")
    message.attach(files)
    try:
        mail.sendmail(sender,receivers,message.as_string())
        print("邮件发送成功")
    except smtplib.SMTPException:
        print ("Error: 无法发送邮件")
```
