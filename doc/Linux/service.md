# 常见服务架设

* [NTP](#ntp)
	* [简介](#简介)
	* [NTP Server 安装配置](#ntp-server-安装配置)
		* [配置选项说明](#配置选项说明)
	* [相关命令](#相关命令)
* [Cron](#cron)
	* [Cron 基础](#cron-基础)
		* [什么是 cron, crond, crontab](#什么是-cron-crond-crontab)
		* [crontab 选项](#crontab-选项)
		* [crontab 格式](#crontab-格式)
	* [使用举例](#使用举例)
* [rsync](#rsync)
	* [rsync 基本介绍](#rsync-基本介绍)
	* [rsync 选项](#rsync-选项)
		* [常用选项](#常用选项)
	* [远程 Shell 方式](#远程-shell-方式)
	* [rsync C/S 方式](#rsync-cs-方式)
	* [一些命令](#一些命令)
		* [常用命令](#常用命令)
		* [ssh 端口非默认 22 同步](#ssh-端口非默认-22-同步)
		* [查看服务器同步资源](#查看服务器同步资源)
# NTP
## 简介

Network Time Protocol（NTP） 是用来使计算机时间同步化的一种协议，它可以使计算机对其服务器或时钟源（如石英钟，GPS 等等)做同步化，它可以提供高精准度的时间校正（LAN 上与标准间差小于1毫秒，WAN上几十毫秒），且可使用加密确认的方式来防止恶毒的协议攻击。默认使用 `UDP 123端口`

NTP 提供准确时间，首先需要一个准确的 UTC 时间来源，NTP 获得 UTC 的时间来源可以从原子钟、天文台、卫星，也可从 Internet 上获取。时间服务器按照 NTP 服务器的等级传播，根据离外部 UTC 源的远近将所有服务器归入不用的层 (Stratum) 中。Stratum-1 在顶层由外部 UTC 接入，stratum-1 的时间服务器为整个系统的基础，Stratum 的总数限制在 15 以内。下图为 NTP 层次图：

![Screenshot](../../images/linux_service/Network_Time_Protocol_servers_and_clients.png)

## NTP Server 安装配置

关于 NTP 服务器的安装，根据不同版本安装方法也不同。REDHAT 系统则可以使用 yum 安装，Ubuntu 系列可以使用 `apt-get` 安装，这里不做具体的介绍，主要详细介绍配置文件的信息。

对于 Centos 过滤注释和空行后，ntp 配置文件内容如下

```
# grep -vE '^#|^$' /etc/ntp.conf 
driftfile /var/lib/ntp/drift
restrict default kod nomodify notrap nopeer noquery 
restrict -6 default kod nomodify notrap nopeer noquery
restrict 127.0.0.1 
restrict -6 ::1
server 0.centos.pool.ntp.org
server 1.centos.pool.ntp.org
server 2.centos.pool.ntp.org
includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
```

### 配置选项说明

* `driftfile` 选项， 则指定了用来保存系统时钟频率偏差的文件。 ntpd 程序使用它来自动地补偿时钟的自然漂移， 从而使时钟即使在切断了外来时源的情况下， 仍能保持相当的准确度。另外，driftfile 选项也保存上一次响应所使用的 NTP 服务器的信息。 这个文件包含了 NTP 的内部信息， 它不应被任何其他进程修改。`无需更改`
* `restrict default kod nomodify notrap nopeer noquery`  默认拒绝所有NTP客户端的操作 [ restrict <IP 地址> <子网掩码>|<网段> [ignore|nomodiy|notrap|notrust|nknod ] 指定可以通信的IP地址和网段。如果没有指定选项，表示客户端访问NTP服务器没有任何限制
	* `ignore`:     关闭所有 NTP 服务
	* `nomodiy`:    表示客户端不能更改 NTP 服务器的时间参数，但可以通过 NTP 服务器进行时间同步
	* `notrust`:    拒绝没有通过认证的客户端
	* `knod`:       kod 技术科阻止 "Kiss of Death" 包（一种 DOS 攻击）对服务器的破坏，使用 knod 开启功能
	* `nopeer`:     不与其它同一层的 NTP 服务器进行同步
* `server [IP|FQDN|prefer]`指该服务器上层 NTP Server，使用 prefer 的优先级最高，没有使用 prefer 则按照配置文件顺序由高到低，默认情况下至少 15min 和上层 NTP 服务器进行时间校对
* `fudge`:          可以指定本地 NTP Server 层，如 `fudge 127.0.0.1 stratum 9`
* `broadcast 网段 子网掩码`:    指定 NTP 进行时间广播的网段，如`broadcast 192.168.1.255`
* `logfile`:        可以指定 NTP Server 日志文件

几个与NTP相关的配置文件:` /usr/share/zoneinfo/`、`/etc/sysconfig/clock`、`/etc/localtime`

* `/usr/share/zoneinfo/`:  存放时区文件目录
* `/etc/sysconfig/clock`:  指定当前系统时区信息
* `/etc/localtime`:        相应的时区文件

如果需要修改当前时区，则可以从 /usr/share/zoneinfo/ 目录拷贝相应时区文件覆盖 /etc/localtime 并修改 /etc/sysconfig/clock 即可

```
cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
sed -i 's:ZONE=.*:ZONE="Asia/Shanghai":g' /etc/sysconfig/clock
```

## 相关命令

`ntpstat` 查看同步状态

```
# ntpstat 
synchronised to NTP server (192.168.0.18) at stratum 4 
   time correct to within 88 ms  	# 表面时间校正 88ms
   polling server every 1024 s		# 每隔 1024s 更新一次
```

`ntpq` 列出上层状态

```
# ntpq -np
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
*192.168.0.18       202.112.31.197   3 u  101 1024  377   14.268    0.998   0.143
```

输出说明：

* `remote`:  NTP Server
* `refid` :  参考的上层 ntp 地址
* `st`    :  层次
* `when`  :  上次更新时间距离现在时常
* `poll`  :  下次更新时间
* `reach` :  更新次数
* `delay` :  延迟
* `offset`:  时间补偿结果
* `jitter`:  与 BIOS 硬件时间差异

`ntpdate` 同步当前时间: `ntpdate NTP服务器地址`

# Cron
## Cron 基础

### 什么是 cron, crond, crontab

> **cron** is the general name for the service that runs scheduled actions. **crond** is the name of the daemon that runs in the background and reads **crontab** files. 

简单理解：cron 是服务，crond 是守护进程， crontab 的 crond 的配置文件。

### crontab 选项

+ `crontab -e` : Edit your crontab file, or create one if it doesn't already exist. # 推荐使用命令新增计划任务--语法检查
+ `crontab -l` : Display your crontab file.
+ `crontab -r` : Remove your crontab file. # 慎用
+ `crontab -u user` : Used in conjunction with other options, this option allows you to modify or view the crontab file of user. When available, only administrators can use this option.

### crontab 格式

    minute(s) hour(s) day(s) month(s) weekday(s) command(s)

``` 
# Use the hash sign to prefix a comment
# +—————- minute (0 – 59)
# |  +————- hour (0 – 23)
# |  |  +———- day of month (1 – 31)
# |  |  |  +——- month (1 – 12)
# |  |  |  |  +—- day of week (0 – 7) (Sunday=0 or 7)
# |  |  |  |  |
# *  *  *  *  *  command to be executed
```

## 使用举例

使用命令 `crontab -e` 编辑 crontab 文件。

(1) 在每天的 7 点同步服务器时间

    0 7 * * * ntpdate 192.168.1.112


(2) 每两个小时执行一次

    0 */2 * * * echo "2 minutes later" >> /tmp/output.txt

(3) 每周五早上十点写周报

    0 10 * * * 5 /home/jerryzhang/update_weekly.py

(4) 每天 6, 12, 18 点执行一次命令

    0 6,12,18 * * *  /bin/echo hello

(5) 每天 13, 14, 15, 16, 17 点执行一次命令

    0 13-17 * * *  /bin/echo hello

__注:__

* 程序执行完毕，系统会给对应用户发送邮件，显示该程序执行内容，如果不想收到，可以重定向内容 `> /dev/null 2>&1`
* 如果执行语句中有 `%` 号，需要使用反斜杠 '\' 转义

# rsync

## rsync 基本介绍

`rsync` 是类 unix 系统下的数据镜像备份工具，从软件的命名上就可以看出来了—— remote sync。它的特性如下：


* 1、可以镜像保存整个目录树和文件系统
* 2、可以很容易做到保持原来文件的权限、时间、软硬链接等等
* 3、无须特殊权限即可安装
* 4、优化的流程，文件传输效率高
* 5、可以使用 rsh、ssh 等方式来传输文件，当然也可以通过直接的socket连接
* 6、支持匿名传输

在使用 rsync 进行远程同步时，可以使用两种方式：__远程 Shell 方式__（用户验证由 ssh 负责）和 __C/S 方式__（即客户连接远程 rsync 服务器，用户验证由 rsync 服务器负责）。

无论本地同步目录还是远程同步数据，首次运行时将会把全部文件拷贝一次，以后再运行时将只拷贝有变化的文件（对于新文件）或文件的变化部分（对于原有文件）。

## rsync 选项

```
Usage: rsync [OPTION]... SRC [SRC]... DEST
  or   rsync [OPTION]... SRC [SRC]... [USER@]HOST:DEST
  or   rsync [OPTION]... SRC [SRC]... [USER@]HOST::DEST
  or   rsync [OPTION]... SRC [SRC]... rsync://[USER@]HOST[:PORT]/DEST
  or   rsync [OPTION]... [USER@]HOST:SRC [DEST]
  or   rsync [OPTION]... [USER@]HOST::SRC [DEST]
  or   rsync [OPTION]... rsync://[USER@]HOST[:PORT]/SRC [DEST]
The ':' usages connect via remote shell, while '::' & 'rsync://' usages connect
to an rsync daemon, and require SRC or DEST to start with a module name.
```
  
__注:__ 在指定复制源时，路径是否有最后的 “/” 有不同的含义，例如：

* /data ：表示将整个 /data 目录复制到目标目录
* /data/ ：表示将 /data/ 目录中的所有内容复制到目标目录

### 常用选项

* `-v` : Verbose (try -vv for more detailed information)            # 详细模式显示
* `-e` "ssh options" : specify the ssh as remote shell              # 指定 ssh 作为远程 shell
* `-a` : archive mode   # 归档模式，表示以递归方式传输文件，并保持所有文件属性，等于 -rlptgoD
    * `-r`(--recursive) : 目录递归
    * `-l`(--links) ：保留软链接
    * `-p`(--perms) ：保留文件权限
    * `-t`(--times) ：保留文件时间信息
    * `-g`(--group) ：保留属组信息
    * `-o`(--owner) ：保留文件属主信息
    * `-D`(--devices) ：保留设备文件信息
* `-z` : 压缩文件
* `-h` : 以可读方式输出
* `-H` : 复制硬链接
* `-X` : 保留扩展属性
* `-A` : 保留ACL属性
* `-n` : 只测试输出而不正真执行命令，推荐使用，特别防止 `--delete` 误删除！
* `--stats` : 输出文件传输的状态
* `--progress` : 输出文件传输的进度
* `––exclude=PATTERN` : 指定排除一个不需要传输的文件匹配模式
* `––exclude-from=FILE` : 从 FILE 中读取排除规则
* `––include=PATTERN` : 指定需要传输的文件匹配模式
* `––include-from=FILE` : 从 FILE 中读取包含规则
* `--numeric-ids` : 不映射 uid/gid 到 user/group 的名字
* `-S, --sparse` : 对稀疏文件进行特殊处理以节省DST的空间
* `--delete` : 删除 DST 中 SRC 没有的文件，也就是所谓的镜像 [mirror] 备份

## 远程 Shell 方式

```
rsync [OPTION]... SRC [SRC]... [USER@]HOST:DEST # 执行“推”操作
or   rsync [OPTION]... [USER@]HOST:SRC [DEST]   # 执行“拉”操作
```

## rsync C/S 方式

```
rsync [OPTION]... SRC [SRC]... [USER@]HOST::DEST                    # 执行“推”操作
or   rsync [OPTION]... SRC [SRC]... rsync://[USER@]HOST[:PORT]/DEST # 执行“推”操作
or   rsync [OPTION]... [USER@]HOST::SRC [DEST]                      # 执行“拉”操作
or   rsync [OPTION]... rsync://[USER@]HOST[:PORT]/SRC [DEST]        # 执行“拉”操作
```

C/S 方式需要配置服务端，下面是一个配置文件示例：

```
# /etc/rsyncd.conf

uid = root
gid = root
use chroot = yes

[bak-data]
    path = /data/
    comment = data backup
    numeric ids = yes
    read only = yes
    list = no
    auth users = data
    filter = merge /etc/.data-filter  # 过滤规则
    secrets file = /etc/rsync-secret
    hosts allow = 192.168.80.0/24 172.16.0.10

[bak-home]
    path = /home/
    comment = home backup
    numeric ids = yes
    read only = yes
    list = no
    auth users = home,test
    exclude = .svn .git
    secrets file = /etc/rsync-secret
    hosts allow = 192.168.80.0/24 172.16.0.10
```

密码文件和 filter 文件内容如下：

```
# cat /etc/rsync-secret
data:123321
home:123456
test:654321
# chmod 600 /etc/rsync-secret
# cat /etc/.data-filter     # 关于 filter 的规则文件需要多测试才能彻底明白
+ mysql56/***
- *
# 以上规则表示匹配所有 mysql56 目录下的内容，其它都不同步
```

关于filter的匹配规则可以参考[man手册](http://www.samba.org/ftp/rsync/rsyncd.conf.html)：

      filter
      The daemon has its own filter chain that determines what files it will let the client access. This chain is not sent to the client and is independent of any filters the client may have specified. Files excluded by the daemon filter chain (daemon-excluded files) are treated as non-existent if the client tries to pull them, are skipped with an error message if the client tries to push them (triggering exit code 23), and are never deleted from the module. You can use daemon filters to prevent clients from downloading or tampering with private administrative files, such as files you may add to support uid/gid name translations.

      The daemon filter chain is built from the "filter", "include from", "include", "exclude from", and "exclude" parameters, in that order of priority. Anchored patterns are anchored at the root of the module. To prevent access to an entire subtree, for example, "/secret", you must exclude everything in the subtree; the easiest way to do this is with a triple-star pattern like "/secret/***".

      The "filter" parameter takes a space-separated list of daemon filter rules, though it is smart enough to know not to split a token at an internal space in a rule (e.g. "- /foo - /bar" is parsed as two rules). You may specify one or more merge-file rules using the normal syntax. Only one "filter" parameter can apply to a given module in the config file, so put all the rules you want in a single parameter. Note that per-directory merge-file rules do not provide as much protection as global rules, but they can be used to make --delete work better during a client download operation if the per-dir merge files are included in the transfer and the client requests that they be used. 

## 一些命令

### 常用命令

```
RSYNC_PASSWORD=123321 rsync -havAEHXi -n --numeric-ids --delete --stats --progress [SRC] [DEST]
```
  
__注：__ 如果有稀疏文件，则添加 `-S` 选项可以提升传输性能。

### ssh 端口非默认 22 同步

使用 ssh 方式传输时如果连接服务器 ssh 端口非标准，则需要通过 `-e` 选项指定：

```
RSYNC_PASSWORD=123321 rsync -havAEHXi -n --numeric-ids --delete --stats --progress -e "ssh -p 22222" [USER@]HOST:SRC [DEST]
```

### 查看服务器同步资源

```
RSYNC_PASSWORD=123321 rsync --list-only data@192.168.80.150::bak-data
或
RSYNC_PASSWORD=123321 rsync --list-only rsync://data@192.168.80.150/bak-data
```