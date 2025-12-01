目前国行iPhone AIR已经发售，但是由于国行版本阉割eSim，AI，无线充电功率等功能，博主还是入手了一台日版的iPhone AIR。但是由于外版手机写入不了国内三大运营商的SIM卡，导致绑了一些国内手机号的服务用起来比较麻烦。近期搜罗了目前网上主流的几种解决方案，看下来有以下几个缺点：

1. [CMLink一卡多号](https://www.cmlink.com/jp/zh/1-card-multi-number/)：申请海外sim卡同时可以申请一个国内号码，可以用于办理国内业务。但是这样解决不了保留国内老号的问题，老号绑了太多国内服务，换号成本太高。
2. IKOS苹果皮：这是一个可以插SIM卡放家里联网的设备，需要在手机上下载APP连接该设备，APP可以把SIM卡的短信和来电转接过来。设备将近800块，有点小贵，而且和他们客服聊过之后得知，应该是把短信和来电先上传到他们的服务器再push到APP，隐私和安全性也没法保证。

后来找到这几篇文章：

- [**EC20 模块+Issabel 实现网络电话**](https://blog.sparktour.me/posts/2022/10/08/quectel-ec20-asterisk-freepbx-gsm-gateway/)
- [**使用EC20模块配合asterisk及freepbx实现短信转发和网络电话**](https://blog.sparktour.me/posts/2022/10/08/quectel-ec20-asterisk-freepbx-gsm-gateway/)

大概原理就是通过4G模块将SIM卡桥接到自建VOIP服务上去，在手机上下在voip通话软件实现接打电话，而短信则通过telegram bot转发。成本很低，而且自建的服务隐私问题也可以保障。以下教程通过上面两篇文章改良而来。

前期准备

1. 移远EC20CEFAG-512-SGNS MINI-PCIE接口全网通4G模块，某鱼购入40块。
2. MINI-PCIE转usb 4G模块转接板，某鱼购入20块。
3. 4G/LTE/GSM内置PCB天线（1代IPEX头），某宝2块。
4. 虚拟机安装Debian11系统。
5. 公网IPV4地址+ddns域名解析。（大部分人应该没有v4公网地址，v6地址应该也同样可以，待验证）
6. 手机端下载APP：（美区Apple ID）我用的是Groundwire，软件自带的push notification功能比zoiper好用
7. 准备iPhone Air可写入的海外流量esim卡，博主用的是RedteaGO，邀请码ZHAN0839

## 虚拟机安装Debian11系统

这部分网上教程资源很多，这里不展开讲了，只列举容易出错的几个步骤。博主是在自组黑群晖上VMM安装的，其他环境同理。

1. [下载Debian11官方镜像](https://saimei.ftp.acc.umu.se/cdimage/archive/11.11.0/amd64/iso-cd/debian-11.11.0-amd64-netinst.iso)
2. 虚拟机CPU配置要选CPU兼容，不然GRUB引导安装容易出问题
3. 安装只选SSH和标准系统

## 配置模块

将模块组装好后，插入SIM卡，连接进Debian11，看到4个ttyUSB端口,说明模块识别成功。

```bash
root@Issabel:~$ ls /dev/ | grep ttyUSB
ttyUSB0
ttyUSB1    #PCM 语音，GPS 信号   
ttyUSB2    #AT 控制命令
ttyUSB3
```

[这里](https://www.rhydolabz.com/documents/30/Quectel_EC20_R2.1_AT_Commands_Manual_V1.0.pdf) 提供了移远 EC20 系列的 AT 命令文档。

### 激活模块

1. 安装依赖

```bash
apt update
apt install minicom libasound2-dev alsa-utils adb libsqlite3-dev asterisk-dev make
```

1. 重置模块 —— 防止前主人错误配置

```bash
#连接模块串口
minicom -D /dev/ttyUSB2

#重置模块
at+qprtpara=3

#重启模块
AT+CFUN=1,1
```

模块重启之后可能需要重新插拔一下USB，如果还是没反应，建议换一个USB口尝试

1. 启动VoLTE —— 4G入网

```bash
#重新连接串口
minicom -D /dev/ttyUSB2

#启用VoLTE
AT+QCFG="ims",1 #打开IMS
AT+QMBNCFG="AutoSel",0 #关闭自动选择mbn文件
at+qmbncfg="deactivate" #反激活当前的mbn
AT+QMBNCFG="select","ROW_Generic_3GPP" #强制选择3gpp
AT+CFUN=1,1 #重启模块
```

重启完了以后，可以在minicom里使用 `AT+QCFG="ims"` 查询 VoLTE 激活状态。 如果是 `+QCFG: "ims",1,1`，就代表 VoLTE 已经启用并激活， 如果是 `+QCFG: "ims",1,0` 就代表 VoLTE 启用了但没有激活， 如果是 `+QCFG: "ims",0,0` 则代表 VoLTE 没有启用。

1. 启动UAC —— 优化通话质量

```bash
#重新连接串口
minicom -D /dev/ttyUSB2

#启动UAC
AT+QCFG="usbcfg",0x2C7C,0x0125,1,1,1,1,1,0,1
```

启动之后可以通过adb devices查看有没有一个android设备, 如果有no serial number的设备就说明激活UAC成功了

```bash
root@Issabel:~# adb devices
* daemon not running; starting now at tcp:5037
* daemon started successfully
List of devices attached
(no serial number)      device
```

## 安装Issabel

### 安装本体

官方提供了安装脚本，一行命令解决：

```bash
curl http://repo.issabel.org/install-debian-install_amp | bash
```

该脚本会安装 Issabel 和 Asterisk 16 及其相关依赖，Issabel 默认占用 80 端口和 443 端口，请保持畅通，安装后可修改。

### 安装EC20驱动

该驱动用于桥接EC20模块和Asterisk 16，让Asterisk 16能像控制一个组件的控制EC20模块

```bash
git clone https://github.com/IchthysMaranatha/asterisk-chan-quectel
cd asterisk-chan-quectel

./bootstrap
./configure --with-astversion=16
make
make install

cp uac/quectel.conf /etc/asterisk
```

编辑 `/etc/asterisk/quectel.conf`，将最后四行的注释去掉：

```bash
[quectel0]
audio=/dev/ttyUSB1                       ; tty port for Audio, set as ttyUSB4 for Simcom if no other dev present
data=/dev/ttyUSB2                        ; tty port for AT commands;no default value
quec_uac=1                              ; Uncomment line if using UAC mode
alsadev=hw:CARD=Android,DEV=0           ; Uncomment if using UAC, set device name or index as reqd
```

编辑 `/etc/asterisk/extensions_custom.conf` ，添加以下内容：

```bash
[incoming-mobile]

; ---------------- SMS ----------------
exten => sms,1,NoOp(Incoming SMS from ${CALLERID(num)})
 same => n,Set(DEC_MSG=${BASE64_DECODE(${SMS_BASE64})})
 ; 输出日志
 same => n,Verbose("Incoming SMS from ${CALLERID(num)}: ${DEC_MSG}")
 ; 追加到主日志文件
 same => n,Set(LOG_FILE=/var/log/asterisk/sms.txt)
 same => n,System(echo "${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - ${QUECTELNAME} - ${CALLERID(num)}: ${DEC_MSG}" >> ${LOG_FILE})
 ; 生成单条未读 SMS 文件
 same => n,Set(UNREAD_DIR=/var/log/asterisk/unread_sms)
 same => n,Set(UNREAD_FILE=${UNREAD_DIR}/${STRFTIME(${EPOCH},,%Y%m%d%H%M%S)}-${CALLERID(num)}.txt)
 same => n,System(echo "${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - ${QUECTELNAME} - ${CALLERID(num)}\n${DEC_MSG}" >> ${UNREAD_FILE})
 same => n,Hangup()

; ---------------- USSD ----------------
exten => ussd,1,NoOp(Incoming USSD)
 same => n,Set(DEC_USSD=${BASE64_DECODE(${USSD_BASE64})})
 same => n,Verbose("Incoming USSD: ${DEC_USSD}")
 same => n,Set(LOG_FILE=/var/log/asterisk/ussd.txt)
 same => n,System(echo "${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - ${QUECTELNAME}: ${DEC_USSD}" >> ${LOG_FILE})
 same => n,Hangup()

; ---------------- Blocked fake call ----------------
exten => report,1,Verbose("Incoming report from ${CALLERID(num)}")
 same => n,Set(UNREAD_FILE=/var/log/asterisk/unread_sms/blocked-calls-${STRFTIME(${EPOCH},,%Y%m%d%H%M%S)}-${CALLERID(num)}.txt)
 same => n,System(echo "${STRFTIME(${EPOCH},,%Y-%m-%d %H:%M:%S)} - BLOCKED REPORT CALL EXTEN=${EXTEN} CHANNEL=${CHANNEL(name)} CALLERID=${CALLERID(num)}" >> ${UNREAD_FILE})
 same => n,Hangup()

; ---------------- Default: pass to from-trunk ----------------
exten => _.,1,Set(CALLERID(name)=${CALLERID(num)})
 same => n,Goto(from-trunk,${EXTEN},1)

```

短信默认保存到 `/var/log/asterisk/sms.txt`；如果需要转发到 Telegram，就把 ;for tg bot use 下方的一行的注释去掉，并创建`/var/log/asterisk/unread_sms/` 文件夹。

## **设备权限配置**

asterisk 用户默认没有访问 `/dev/ttyUSB*` 的权限。为了后续配置方便，博主直接把asterisk 加入root组

```bash
root@Issabel:~# usermod -g 0 asterisk
```

大部分linux的发行版会有个**`ModemManager.service`** 占用`/dev/ttyUSB2` ，这里一并给他禁用掉

```bash
root@Issabel:~# systemctl stop ModemManager.service
root@Issabel:~# systemctl disable ModemManager.service
```

然后重启asterisk：

```bash
root@Issabel:~# systemctl restart asterisk
```

进入asterisk控制台`asterisk -rvvv` 执行`quectel show devices` 能显示出设备则说明模块加载成功

```bash
root@Issabel:~# asterisk -rvvv
Asterisk 16.16.1~dfsg-1+deb11u1, Copyright (C) 1999 - 2018, Digium, Inc. and others.
Created by Mark Spencer <markster@digium.com>
Asterisk comes with ABSOLUTELY NO WARRANTY; type 'core show warranty' for details.
This is free software, with components licensed under the GNU General Public
License version 2 and other licenses; you are welcome to redistribute it under
certain conditions. Type 'core show license' for details.
=========================================================================
Connected to Asterisk 16.16.1~dfsg-1+deb11u1 currently running on Issabel (pid = 520)
Issabel*CLI> quectel show devices
ID           Group State      RSSI Mode Submode Provider Name  Model      Firmware          IMEI             IMSI             Number        
quectel0     0     Free       20   0    0       CHN-UNICOM     EC20F      EC20CEFAGR06A10M4 1234567890123   123456789123034  Unknown       
Issabel*CLI> 
```

## 配置Issabel

浏览器输入虚拟机所在的 IP 地址打开 Issabel 控制台，点击中央的 `IssabelPBX Administration` 输入用户名 `admin` 和安装过程中设置的密码登录。

### 添加分机

`Applications-Extensions` 添加分机号。

选择`Generic PJSIP Device` 。

纯数字填写User Extension，secret可以改成好记的字符串，该配置作为你在手机APP端登陆的用户名和密码，Display Name随便填。`remove_existing = Yes`，`nat = Yes`填完记得点右下角的`Submit`，最后点左上角的`Apply`。

![image](https://github.com/AmorXxx/iPhone_air_esim_tutorial/blob/main/IMG/%E6%B7%BB%E5%8A%A0%E5%88%86%E6%9C%BA.png)

### 添加中继

`Connectivity-Trunks`，选择 `Add Custom Trunk` 。

`Trunk Name` 随便填，Custom Dial String 填 `Quectel/quectel0/$OUTNUM$`，直接 Submit，Apply。

记得删掉除了刚刚添加的 Trunk 外其它自动生成的 Trunk。

![image](https://github.com/AmorXxx/iPhone_air_esim_tutorial/blob/main/IMG/%E6%B7%BB%E5%8A%A0%E4%B8%AD%E7%BB%A7.png)

### 添加入站路由

`Connectivity-Inbound Routes` 添加入站路由。

`Description` 随便填，`Set Destination` 选择 `Extension`，然后在下方的出口里选择刚添加的。

![image](https://github.com/AmorXxx/iPhone_air_esim_tutorial/blob/main/IMG/%E6%B7%BB%E5%8A%A0%E5%85%A5%E7%AB%99%E8%B7%AF%E7%94%B1.png)

### 添加出站路由

`Connectivity-Outbound Routes` 添加出站路由。

`Route Name` 随便填，`Trunk Sequence for Matched Routes` 选刚刚添加的 Trunk，设置一个 `X.` 作为 Pattern 以匹配所有号码。

![image](https://github.com/AmorXxx/iPhone_air_esim_tutorial/blob/main/IMG/%E6%B7%BB%E5%8A%A0%E5%87%BA%E7%AB%99%E8%B7%AF%E7%94%B1.png)

### Issabel网络配置

`Settings-PJSIP Settings`，

`NAT = Yes`

`IP Configuration = Dynamic IP`

`Dynamic Host` =  你的域名

`Local Networks` = 你的局域网段和子网掩码

建议修改一下TCP/UDP Transport中的Bind Port，我实操下来发现某些运营商可能会封禁默认端口，我这里是改成了5160

![image](https://github.com/AmorXxx/iPhone_air_esim_tutorial/blob/main/IMG/%E7%BD%91%E7%BB%9C%E9%85%8D%E7%BD%AE.png)

`Settings-PJSIP Settings`

`RTP Port Ranges = 10000 - 10010`

`RTP Timers = 3 (rtptimeout)`

博主这里只连了一部手机，基本没什么并发场景，所以把`RTP Port Ranges`调小了许多。

测试的时候发现在外网通话时偶尔会收不到Hang up信号，不会自动挂断。所以改了`rtptimeout = 3` ，3s没有收到任何音频信号直接挂断。

## 测试

### 测试呼叫

我使用的APP是GroundWire，点击右上角小齿轮，Incoming Calls设置Push Notifications。然后添加一个Generic SIP Account，Title随便写，Username是刚刚分机设置里的User Extension，password是分机设置里的secret，Domain设置为你的域名 + port完成后点击save。

然后随便打一个电话就可以测试了

### 测试Push Notification

下面给出两种测试方法

1. APP自带Push Notification的测试，点击测试，等待几秒如果提示Push Test来电则说明功能开启成功
2. 关闭APP，进入asterisk控制台： `asterisk -rvvv`, 打开APP，再关闭APP，如果关闭APP的同时控制台输出`Endpoint 200 is now Reachable` 则说明GroundWire的服务器已经和asterisk建立了连接，这样的话即使APP在后台也不怕电话漏接了

## 短信转发

### 创建telegram bot

tg搜索 @BotFather创建机器人，记得保存`API Token`

### 安装依赖

```bash
pip install python-telegram-bot==13.7 watchdog
```

### 编写脚本

博主部署的环境不能直连telegram，所以用了proxy，这部分代码按需修改吧。

```bash
import os
import time
import logging
import subprocess
from telegram.ext import Updater, CommandHandler, MessageHandler, Filters
from telegram import Update
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

# 配置
TOKEN = "Bot Token"  # 替换为你的Bot Token
MESSAGE_DIR = "/var/log/asterisk/unread_sms/"  # 替换为你的消息文件夹路径
ALLOWED_IDS = [123456789]  # 替换为允许的Telegram用户ID列表

# 代理配置（根据你的代理类型选择一种）
PROXY = {
    # HTTP代理示例
    # 'proxy_url': 'http://proxy.example.com:8080',
    # 如果需要认证：
    # 'proxy_url': 'http://username:password@proxy.example.com:8080',
    'proxy_url': 'http://username:password@proxy.example.com:8080'
    # SOCKS5代理示例
    #'proxy_url': 'socks5://proxy.example.com:1080',
    # 如果需要认证：
    # 'proxy_url': 'socks5://username:password@proxy.example.com:1080',
}

# 日志配置
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class MessageHandler(FileSystemEventHandler):
    """文件夹变化处理器"""
    def __init__(self, bot, allowed_ids):
        self.bot = bot
        self.allowed_ids = allowed_ids

    def on_created(self, event):
        """当新文件创建时触发"""
        if not event.is_directory and event.src_path.endswith('.txt'):
            logger.info(f"检测到新文件: {event.src_path}")
            time.sleep(0.5)  # 确保文件写入完成
            try:
                if not os.path.exists(event.src_path):
                    logger.error(f"文件 {event.src_path} 不存在")
                    return
                if os.path.getsize(event.src_path) == 0:
                    logger.warning(f"文件 {event.src_path} 为空")
                    for chat_id in self.allowed_ids:
                        self.bot.send_message(chat_id=chat_id, text=f"New message: 文件 {os.path.basename(event.src_path)} 为空")
                    os.remove(event.src_path)
                    return

                with open(event.src_path, 'r', encoding='utf-8') as f:
                    content = f.read().strip()
                logger.info(f"读取到文件内容: {content}")
                
                if not content:
                    logger.warning(f"文件 {event.src_path} 内容为空")
                    for chat_id in self.allowed_ids:
                        self.bot.send_message(chat_id=chat_id, text=f"New message: 文件 {os.path.basename(event.src_path)} 内容为空")
                else:
                    for chat_id in self.allowed_ids:
                        self.bot.send_message(chat_id=chat_id, text=f"New message:\n{content}")
                        logger.info(f"消息已发送给用户 {chat_id}")
                
                os.remove(event.src_path)
                logger.info(f"删除文件: {event.src_path}")
            except Exception as e:
                logger.error(f"处理文件 {event.src_path} 失败: {str(e)}")
                for chat_id in self.allowed_ids:
                    self.bot.send_message(chat_id=chat_id, text=f"读取 {os.path.basename(event.src_path)} 失败: {str(e)}")

def get_user_id(update: Update, context):
    """处理 /myid 命令，获取用户ID"""
    user_id = update.message.from_user.id
    update.message.reply_text(f"你的Telegram ID是: {user_id}")
    logger.info(f"用户 {user_id} 查询了其ID")

def send_sms(update: Update, context):
    """处理 /send 命令，执行asterisk发送短信"""
    user_id = update.message.from_user.id
    if user_id not in ALLOWED_IDS:
        update.message.reply_text("无权限使用此Bot！")
        logger.info(f"用户 {user_id} 尝试使用/send，无权限")
        return

    args = context.args
    if len(args) < 2:
        update.message.reply_text("用法: /send <phone_number> <message>")
        logger.warning(f"用户 {user_id} 输入无效/send命令: {args}")
        return

    phone_number = args[0]
    message = ' '.join(args[1:])  # 支持多词消息

    # 验证phone_number（简单检查，确保是数字）
    if not phone_number.isdigit():
        update.message.reply_text("电话号码必须是数字！")
        logger.warning(f"用户 {user_id} 输入无效电话号码: {phone_number}")
        return

    # 构建asterisk命令
    command = ['asterisk', '-rx', f'quectel sms quectel0 {phone_number} "{message}"']
    
    try:
        # 如果需要sudo，修改为：command = ['sudo', 'asterisk', '-rx', f'quectel sms quectel0 {phone_number} "{message}"']
        result = subprocess.run(command, capture_output=True, text=True, check=True)
        update.message.reply_text(f"短信发送成功: {phone_number}\n输出: {result.stdout}")
        logger.info(f"用户 {user_id} 发送短信到 {phone_number}: {message}")
    except subprocess.CalledProcessError as e:
        update.message.reply_text(f"短信发送失败: {str(e)}\n错误: {e.stderr}")
        logger.error(f"用户 {user_id} 发送短信失败: {str(e)}")
    except Exception as e:
        update.message.reply_text(f"执行命令失败: {str(e)}")
        logger.error(f"用户 {user_id} 执行命令失败: {str(e)}")

def start_watching(bot, allowed_ids):
    """启动文件夹监控"""
    event_handler = MessageHandler(bot, allowed_ids)
    observer = Observer()
    observer.schedule(event_handler, MESSAGE_DIR, recursive=False)
    observer.start()
    logger.info("文件夹监控已启动")

def main():
    """主函数，启动Bot"""
    updater = Updater(TOKEN, use_context=True, request_kwargs=PROXY)
    dp = updater.dispatcher

    # 添加命令处理器
    dp.add_handler(CommandHandler("myid", get_user_id))
    dp.add_handler(CommandHandler("send", send_sms))

    # 错误处理
    def error(update, context):
        logger.error(f"更新 {update} 引发错误: {context.error}")

    dp.add_error_handler(error)

    # 启动文件夹监控
    start_watching(updater.bot, ALLOWED_IDS)

    # 启动Bot
    updater.start_polling()
    logger.info("Bot已启动")
    updater.idle()

if __name__ == '__main__':
    main() 
```

### 常驻服务

1. 创建`systemd`服务文件

```bash
vi /etc/systemd/system/telegram-bot.service
```

1. 添加以下内容

```bash
[Unit]
Description=Telegram Message Bot
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/python3 /path/to/bot.py
WorkingDirectory=/path/to
User=root
Group=root
Restart=always
RestartSec=10
StartLimitIntervalSec=60
StartLimitBurst=3
TimeoutStartSec=30
TimeoutStopSec=10
StandardOutput=journal
StandardError=journal
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

1. 启动并测试服务

```bash
# 重新加载systemd配置
systemctl daemon-reload
# 启用服务-- 开机自启
systemctl enable telegram-bot
#启动服务
systemctl start telegram-bot
#检查服务状态
systemctl status telegram-bot # 输出应显示active (running)
#查看日志
journalctl -u telegram-bot -f
```

### 测试收发短信

服务启动后新收到的短信会先保存到`/var/log/asterisk/unread_sms/` 目录下，脚本监测到有新文件产生就会把文件转发到tg bot上。

发信息指令为/send，给tg bot发送消息`/send 10010 1`,会触发指令`asterisk -rx quectel sms quectel0 10010 "1”`，也就是给10010发送内容为1的短信。

![image](https://github.com/AmorXxx/iPhone_air_esim_tutorial/blob/main/IMG/%E7%9F%AD%E4%BF%A1%E6%B5%8B%E8%AF%95.png)

All done！！

## 写在最后

随着公网v4地址的收紧，目前能申请到v4地址的地区已经越来越少，所以博主也在做内网穿透和ipv6相关的测试。对于家里没有能24小时开机的服务器的小伙伴，理论上用树莓派也可以代替虚拟机，这些问题都需要测试和优化，欢迎各路大神共同讨论交流。

联系方式 

tg：@Amor_Paul

## 常见问题汇总

1. 一直报permission denied连接不上模块试试看 `vim /lib/systemd/system/asterisk.service` asterisk启动的时候有没有指定Group。
2. `snd_pcm_open failed: No such device`检查`asterisk`有没有`audio`的权限

## 该方案博主已在树莓派3B上验证完美运行，其他树莓派同理。
