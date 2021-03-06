---
layout: post
title: 趣店集团-研发线上操作军规
author: 徐章健
tag: 规范,线上操作
digest: 为了保证生产环境的持续、稳定、高效地运转，并且使新同学更快的掌握线上操作的基本方法，本文从禁忌，强制点出发，整理出”操作手册”，并加入一些平时遇到的问题，总结成操作条款。
---

> 如有违反，请自行认领各类惩罚吧。

### 1. 线上变更操作
* 条款01:禁止流量高峰进行影响cache的升级
    * 内容：对影响cache的升级操作禁止在流量高峰进行.
    * 正确：应该在服务流量低峰期进行上线或操作.
    * 说明：减少上线或操作对用户的影响，在异常时候减少损失.
    
* 条款02:禁止程序线上“裸奔”
    * 内容：禁止程序在线上"裸奔".
    * 正确：应该在程序上线前增加相应的"监控","统计"等.
    * 说明：防止服务异常时,op无法知晓,不能及时处理.
    
* 条款03:禁止上线程序没有“回滚方案”
    * 内容：禁止程序上线时没有"回滚方案".
    * 正确：所有上线必须有回滚方案.
    * 说明：防止因回滚准备不足，导致回滚时间变长或出错.
    
* 条款04:禁止不进行备份
    * 内容：禁止不进行备份.
    * 正确：上线前，至少备份其变更部分.
    * 说明：避免回滚时无法快速进行有效回滚.
    
* 条款05:禁止无依据操作
    * 内容：禁止无依据的操作.
    * 正确：
        * 依据已有的checklist.
        * 依据op和rd讨论的结论进行.
    * 说明：避免由于个人单点，对服务处理不当，导致上线操作出现非预期的结果.
    
* 条款06:禁止以ruser账号在线上进行非必要操作
    * 内容：禁止无依据的操作.例如ruser账号下添加crontab任务。
    * 正确：
        * 紧急操作可以找op申请suser权限，非紧急发工单由op执行。

* 条款07:机器间拷贝数据应限速
    * 内容：进行线上机器间数据拷贝之前，要查看系统性能、负载，分析和判断后进行.
    * 正确：数据传输必须进行限速传输.
    * 说明：避免大数据操作对系统影响.

* 条款8:禁止程序上线后不复查
    * 内容：程序上线后，应检查服务资源消耗、服务错误日志.

* 条款09:禁止RD不按流程上线
    * 正确：需要经过审核后，通过deploy上线.
    * 说明：防止RD误更新导致服务异常.

* 条款10:禁止RD修改服务资源配置
    * 内容：修改涉及机器nginx等配置，op要review后才能上线。
    * 正确：评估RD对配置的修改，然后做出上线判断.
    * 说明：一个服务的资源配置修改会影响到其他服务，甚至整个集群，对配置修改上线需要严格控制.
    
* 条款12:建议在特定时间上线
    * 内容：下午6点后，尽量不上线，尽量不在周六上线
    * 正确：尽量在周一至周五的11点-12点、17– 18点进行上线操作
    * 说明：在工作时间进行上线，出现问题，能够及时找到相关人员进行解决.
    
### 2.定时任务
* 条款13:不建议整点运行任务
    * 内容：不建议在整点时间运行任务,如 00:00.
    * 正确：应该跳过整点运行,如:00:01.
    * 说明：避免因系统时钟提前运行或不运行导致任务执行异常.
    
* 条款14:禁止程序屏幕打印日志
    * 内容：不允许程序屏幕打印日志.
    * 正确：应该将输出重定向到其他文件，或以 &>/dev/null 结尾.
    * 说明：避免系统将标准输出，打到 /var/spool/clientmqueue/下，引起根分区空间不足，导致硬盘报警.
    
* 条款15:禁止定时任务“裸奔”
    * 内容：不允许定时任务“裸奔”.
    * 正确：应该对定时任务程序进行监控.
    * 说明：避免因程序异常退出，不能及时处理，导致任务不按时执行.
    
* 条款16:禁止crontab中直接写shell命令
    * 内容：不准在crontab中直接写shell命令（cd命令除外）.
    * 正确：如果有需求，必须用脚本封装来实现.
    * 说明：避免由于crontab 解析问题引起运行异常.
    
### 3.日志方面
* 条款17:日志文件须切割
    * 内容：日志文件不能不切割.
    * 正确：日志文件必须切割处理。常用的切割方法包括cronlog/定期mv等.
    * 说明：防止服务程序日志过大引起服务故障,磁盘空间不足.
    
* 条款18:禁止Vi查看日志
    * 内容：禁止用vi(vim)查看日志.
    * 正确：应该用 tail more less命令查看日志.
    * 说明：防止改变文件和用vim打开比较大的日志文件，系统资源消耗过多.

* 条款19:禁止线上进行资源高消耗的操作
    * 内容：线上分析日志时，如果预期要消耗大量资源，不直接在线上操作。
    * 正确：
        * 将日志限速拖到线下再分析。
        * 联系op将机器流量摘掉再进行操作。
    * 说明：系统资源消耗过多，影响线上服务。
    
* 条款20:禁止对IO敏感服务使用重定向方式文件覆盖
    * 内容：对于IO敏感的服务，禁止使用重定向的方式进行文件覆盖.
    * 正确：
        * mv后，低优先级删除的方式进行. （nice -n **）
        * 或者停服务删.
    * 说明：防止IO抢占影响服务.
    
### 4.文件修改
* 条款21:禁止直接修改正在运行的程序的配置，数据
    * 内容：对于正在运行的程序，其配置，数据，程序文件禁止直接修改.
    * 正确：应该修改其副本，diff确认正确后，再进行替换.
    * 说明：防止程序重启或重载时加载正在修改的文件.
    
* 条款22:禁止线上直接修改程序代码
    * 内容：禁止在线上直接修改程序代码.
    * 正确：应该走程序的上线流程来进行上线操作.

### 5.文件权限
* 条款23:禁止文件权限设置为777,755
    * 内容：文件权限不能设为777,755.
    * 正确：可执行文件应该设为744. 非可执行文件应该设为644. 系统文件权限为安装时的默认权限.
    
* 条款24:禁止目录权限设为777
    * 内容：目录权限不能设为777.
    * 正确：
        * 日志、配置及非关键数据的目录权限为755。
        * 含有密码及敏感信息的文件/目录，不允许同组帐号和其他用户帐号有读、写和执行权限.
    
### 6.数据传输
* 条款24:禁止不限速数据传输
    * 内容：禁止不限速的数据传输.
    * 正确：应该在有效通报后，进行数据传输，建议低于5MB/s.
    * 说明：减少数据传输给系统的资源消耗.
    
* 条款25:大规模数据传输前应先通报
    * 内容：大规模数据传输启动前钉钉通报组内，避免对其他服务造成影响.
    * 说明：例如重灌数据等。
    
### 7.Root用户
* 条款26:慎用关机重启命令
    * 内容：慎用 `poweroff,reboot,shutdown,init` 等关机重启命令.
    * 正确：应该在确认操作服务器名,后再进行操作.
    * 说明：防止误操作服务器.
    
* 条款27:禁止未授权修改iptables规则
    * 内容：禁止未授权修改 `iptables` 规则.
    * 正确：需要找op申请
    
* 条款28:禁止密码设为初始密码或弱口令
    * 内容：禁止将 root,工作账户密码设置为弱口令或者安装时候的初始密码.如:123456等.
    * 正确：最好用 ranpwd工具生成随机密码，该工具生成的密码符合密码管理的要求.
    
* 条款30:先扫描硬盘再重启服务
    * 内容1：重启服务器后应该先进行扫描硬盘后,再启动服务.
    * 内容2：除特定的技术方案外（如程序逻辑需要格式化硬盘分区），禁止op修改sudoer 配置
    * 命令使用
    
* 条款31:谨慎使用killall“程序名”方式停止服务
    * 内容：停止服务，谨慎使用 `killall` "程序名"方式.
    * 正确：使用相应程序的控制接口进行启停。如果程序暂时没有控制接口，需要严格防止误杀.
    
* 条款32:慎用kill -g 命令
    * 内容：慎用 `killall -g`命令.
    * 正确：同源程序会被错误杀掉，如a和b都由同一个supervise调用，就会出问题.

* 条款33:慎用rm –rf* 进行文件删除
    * 内容：慎用`rm -rf *`进行文件删除.
    * 正确：
        * 确认目录包含文件，确认操作的目标路径
        * 批量的删除应该先进行一台，确认无误后再进行，每台都需要确认.
        * 严格避免 cd 错误导致的误删除
        
* 条款34:禁止cd后不检查当前路径
    * 内容：禁止 cd 后不检查当前路径.
    * 正确：cd 命令后，必须判断是否已经在预期的路径下，再进行相关操作
    
* 条款35:禁止使用skill root
    * 内容：禁用skill root
 
### 8.故障处理
* 条款36:当发生故障时，首先保证服务可用
    * 内容：当服务出现故障，一定先保证线上服务可用，再进行定位。
    * 正确：可以采取的方案包括将部分机器摘掉以保存现场，手动备份日志等。