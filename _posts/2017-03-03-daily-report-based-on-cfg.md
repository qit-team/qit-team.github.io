---
layout: post
title: Set Us Free：趣店BI基于配置的自动化报表工具简单实现
author: 仰宗强
tag: 报表,工具,配置
digest: BI团队成长路上的自我拯救：平时一大堆又乱又杂的报表需求，即便是招再多的工程师，也都是被动的；真正的出路，是实现报表需求从「多乱难 → 有序 → 简单化 → 流程化 → 配置化 → 自动化」的蜕变！
---


## 一、背景

- 公司对内的报表需求太多
- 要求快速响应
- 需要追溯历史
- 需要周期性例行（同种需求需要日、周、月都要）

之前的状况是，每次的数据报表需求都要进行评估、开发、测试、数据校准、再上线推送，周期较长，遇上紧急的数据需求，就必须得大波儿人马投入，加班加点搞了。
还总是出现「数据不能及时输出，业务着急、产品着急、老板着急，研发更着急」！

时间长了，BI同学可能会对自己产生一个**错误**的定位：我的工作职责就是接需求、开发报表！

所以，一个能够自动化构建邮件报表的工具，保证这类的报表需求能够快速、准确的响应并输出，同时还能把研发的日常工作，进行`多乱难 → 有序 → 简单化 → 流程化 → 配置化 → 自动化`的蜕变，是多么必要且重要的事情啊！

## 二、目标

- 当接到一个新的报表需求时，不用写代码,只要 *配置参数* 就可以了。
- 当一个老的报表需要优化时，不用修改代码,只要修改 *配置参数* 就可以了。
- 当一个提数需求需要追溯时，不用拿sql一个个去执行，直接修改配置就可以了。

## 三、技术方案

### 3.1 技术架构图

![技术架构图](/public/images/report/技术架构图.png)

> 由于目前模型指标体系尚未完善，所以目前操作用户主要是BI成员，需求方对应为公司各业务线。

主要思路：用户能够在配置页面上完成报表的所需配置，然后在后台启动服务，对报表配置进行解析，执行并生成报表，发送给对应的用户。


### 3.2 编写配置页面，保存配置数据

#### 3.2.1 需求登记

> 主要是指这个报表是干啥的（产品日报），其中发布状态主要将该需求从测试状态改到发布，正式运行。

![需求报表配置](/public/images/report/需求报表配置.jpeg)

#### 3.2.2 指标定义

> 配置需求指标名称、指标逻辑以及执行时间；一个需求支持多个需求指标。

![需求报表指标配置](/public/images/report/需求报表指标配置.jpeg)

#### 3.2.3 依赖管理

> 对于一个需求，它的数据源一般都不是直接去取线上业务库的数据，中间会通过同步、清洗、模型建立等操作加工成多个数据模型；所以在配需求时，一定要配置其模型依赖。

![指标依赖配置](/public/images/report/指标依赖配置.jpeg)

### 3.3 后台服务，主要是后台常驻进程，采用队列消费的方式进行自动发送报表邮件

- 使用Redis队列，保存每次需要执行的需求指标信息

- 后台常驻进程，使用Crontab配合守护的方式，保证进程被kill掉或者假死

- 配置一个crontab，每1分钟检查一次，保证程序被kill死掉

- 配置一个crontab，每天执行一次，kill掉进程，保证常驻进程假死

- 代码中增加一段代码，实现脚本自守护

```javascript
$cmd = "ps -ef | grep -v grep |grep php | grep 'artisan' | grep 'execute:service' | wc -l";
$res = array();
exec($cmd, $res);
if ($res[0] > 2) {
   echo date('Y-m-d H:i:s') . " | " . "进程已存在，正常退出 \n";
   exit();
}
echo date('Y-m-d H:i:s') . " | " . "进程已死，正常启动 \n";
```

- 生产队列数据的常驻进程，该脚本的主要作用是从配置库中取出该时间点以及之前需要执行的需求指标信息，并插入各自队列（进行多队列并行）

```javascript
public static function pushToTaskQueue(){
    $taskInfo = self::getCurrentData();//获取当前需要执行的指标信息
    foreach ($taskInfo as $task) {
        $queque = self::getRedisQueueList($task['model']);//获取队列名称
        if(empty($queque)){
            continue;
        }
        self::pushToRedis($task, $queque);
        switch ($task['is_routine']) {
            case 1: //天例行
                $execute_time = date('Y-m-d H:i:s',strtotime($task['execute_time']) + 246060);
                break;
            case 2: //周例行
                $execute_time = date('Y-m-d H:i:s',strtotime($task['execute_time']) + 246060*7);
                break;
            case 3: //月例行
                $m_days = date('t',strtotime($task['execute_time']));
                $execute_time = date('Y-m-d H:i:s',strtotime($task['execute_time']) + 246060*$m_days);
                break;
            default:
                $execute_time = date('Y-m-d H:i:s',strtotime('2037-12-01'));
                break;
        }
        DB::connection(self::$bi_db)->table('ls_execute_info')->where('id', $task['id'])
            ->update(array('execute_time' => $execute_time));
    }
}
```

- 消费队列数据的常驻进程，该脚本的主要作用是从任务队列中获取需要执行的需求指标信息，进行占位符替换，并执行获取报表数据，并插入log日志

```javascript
$exec_start_at = date('Y-m-d H:i:s',time());
$end_at = date('Y-m-d',strtotime($execute['execute_time']));
$execute_at = date('Y-m-d', strtotime("-1 days", strtotime($execute['execute_time'])));
switch ($execute['is_routine']) {
    case 1: //天例行
        $start_at = $execute_at;
        break;
    case 2: //周例行
        $start_at = self::getFirstOfWeek($execute_at);
        break;
    case 3: //月例行
        $start_at = self::getFirstOfMonth($execute_at);
        break;
    default:
        $start_at = date('Y-m-d', strtotime("-1 days", strtotime($execute['execute_time'])));
        break;

}
$partition_at = date('Y-m-d', strtotime("-1 days", strtotime($execute['execute_time'])));
$execute_sql = preg_replace('/#start_at/',$start_at,$execute['execute_sql']);
$execute_sql = preg_replace('/#end_at/',$end_at,$execute_sql);
$execute_sql = preg_replace('/#partition_at/',$partition_at,$execute_sql);
```

- 消费队列数据的常驻进程，另一个作用是每执行完一次需求指标信息后，会判断该需求的所有指标是否都已经完成；
若已完成，会自动调起邮件服务，对数据进行模版拼接，并自动根据相应的配置将数据内容报表可视化的推送给需求方。

![效果展示](/public/images/report/效果展示.png)

## 总结

* 减少技术成本，提升效率和数据准确性，是报表设计的核心；
* 自动化报表不是万能的，不是所有的报表都可以通过配置化就可以生成；但是只要能解决大部分定制化的需求，那就是成功的；
* 繁琐且重复低效的工作，只要深度思考，一定能找到简化的办法，事情简化了，再继续流程化也不是不可以，既然都流程化了，离自动化也就不远了！