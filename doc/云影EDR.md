###  云影 EDR👋
<p>
  <img src="https://img.shields.io/github/stars/bx-sec/cloudshadow.svg?style=flat-square" />
  <img src="https://img.shields.io/github/license/bx-sec/cloudshadow?style=flat-square" />
</p>

云影是由百晓安全团队开发维护的云环境端点威胁检测与响应系统。整个系统集异常检测、监控管理为一体，拥有容器监控、异常行为发现、内核级恶意行为阻断、高级分析等功能，可从多个维度行为信息中发现入侵行为。

## akserver
通过分析识别终端行为,产出对应的威胁事件与情报,上报威胁事件至本地威胁情报中心。

## control
主要功能为agent控制、升级更新、规则与策略同步、上线授管、keepalive等.

## 客户端agent

主要负责采集监控数据，数据降噪以及数据上报，同时黑名单和基于事件规则命中来阻止已知威胁，支持一些异常行为的内核级阻断操作：例如恶意进程创建，非法外联.

## kernel_driver

基于ftrace开发(3.x版本:ftrace, 2.x版本:inlinehook, 4.x版本:ebpf)，并根据进程行为，文件变动以及主机网络事件分别实现三个独立driver模块，三大模块基于akfs自研文件系统运行.

process_driver:  采集进程fork、exec、exit、kill等信息.
file_driver: 监控输出文件创建、写入、权限变更、删除、移动、映射、chroot等事件.
net_driver: 监控tcp v4/6请求建连、tcp v4/6接受建连、非法监听、DNS请求等主机网络事件.
akfs_driver：山竹开发实现的一个文件系统，提供基础能力支持.



## 云影架构
![](images/about.png)

## 输出示例
![](images/out.png)


## 降噪规则说明
```json
[
    {
        "id":"xxxx-xxxx-xxxx-xxxx"
        "meta": {
            "author": "xxx",
            "description": "docker 进程访问日志过滤",
            "name": "docker 进程写日志"
        },
        "and": true, // rules的逻辑，true为全部符合，false则只需要符合一个
        "enabled": true, // 规则启用开关

        "rules": {
            "Image":{ 
              "data": ["/usr/local/bin/dockerd"],
              "type": "string"
            },
            "Tarfile":{ 
              "data": [".log",".log1"], // 判断值,数组方式
              "type": "endswith", // 判断方式
              "notIs": true         //指定判断逻辑，true时，该规则表示chg_file后缀不为.log 则命中.
            }
        },
        "priority": 1, // 规则优先级
        "system": "linux" // 操作系统类型 (windows,linux)
    }
]
```

### 规则数据匹配类型
- string： 字符串完全命中
- startswith：前缀 
- endswith：后缀
- contains：包含/关键字
- regexp：正则

### 初始默认规则
```json
[
  {
    "meta": {
      "author": "phil",
      "description": "优先级最高,尽量不删除该规则,使用enabled控制.",
      "name": "仅监控docker事件,优先级最高"
    },
    "and": true,
    "enabled": true,

    "rules": {
      "DockerName": {
        "data": ["localhost"],
        "type": "string"
      }
    },
    "priority": 1,
    "system": "linux"
  },
  {
    "meta": {
      "author": "phil",
      "description": "优先级最高,尽量不删除该规则,使用enabled控制.",
      "name": "根据事件类型直接过滤"
    },
    "and": true,
    "enabled": true,

    "rules": {
      "EventID": {
        "data": ["1001","1003","1004","2001","2003","2004","2005","2006","2007","2008"],
        "type": "string"
      }
    },
    "priority": 1,
    "system": "linux"
  },
  {
    "meta": {
      "author": "phil",
      "description": "频繁写入会导致事件过多",
      "name": "日志以及记录类文件过滤"
    },
    "and": true,
    "enabled": true,

    "rules": {
      "TargetFile": {
        "data": ["nohup.out",".log","-log",".pcap",".db"],
        "type": "endswith"
      }
    },
    "priority": 1,
    "system": "linux"
  },
  {
    "meta": {
      "author": "phil",
      "description": "允许范围内的网络访问或者外联地址过滤.",
      "name": "允许范围内的网络访问或者外联地址过滤"
    },
    "and": true,
    "enabled": true,

    "rules": {
      "RemoteIp": {
        "data": ["127.0.0.1"],
        "type": "string"
      }
    },
    "priority": 1,
    "system": "linux"
  }
]
```

## 服务端事件规则示例
支持yml与json两种格式编写事件规则
```yml
id: 42df45e7-e6e9-43b5-8f26-bec5b39cc239
meta:
  author: phil
  description: Detects system information discovery commands
  title: System Information Discovery
  references:
    - https://github.com/redcanaryco/atomic-red-team/blob/master/atomics/T1082/T1082.md
and: true
enabled: true
rules:
  EventID:
    data:
      - '1002'
    type: string
  Image:
    data:
      - /uname
      - /hostname
      - /uptime
      - /lspci
      - /dmidecode
      - /lscpu
      - /lsmod
    type: endswith
source: process
system: linux
level: informational
tags:
  - attack.discovery
  - attack.t1082
```

## 捕获数据说明
### EventID:
进程相关
- 1001: 进程创建
- 1002: 进程执行/命令执行
- 1003: 进程退出
- 1004: kill

文件相关
- 2001:  文件创建
- 2002:  文件写入
- 2003:  权限变更
- 2004:  文件删除
- 2005:  文件移动
- 2006:  文件映射
- 2007:  chroot
- 2008:  文件所属

网络相关
- 3001: tcp4请求建连
- 3002: tcp接受建连
- 3004: tcp6请求建连
- 3005: tcp监听
- 3006: udp监听
- 3010: dns查询


### 进程类事件字段内容
- DockerName: 关联docker 容器名称
- EventID: 事件ID
- Image: 进程
- hash: 进程文件hash
- CommandLine: 命令行参数
- ParentName: 父进程名
- uid: 所属用户ID
- UserName: 用户名
- gid: 所属用户组ID
- GroupName: 组名
- Euid: 执行权限
- time: 时间
- Namespace: 进程命名空间ID
- ppid: 父进程ID
- ptgid: 父进程TGID
- sig: Sig
- TargetPid: 接受kill信号进程pid
- ActiveName: 发送kill信号进程名
- TargetProcessName: 接受kill信号进程pid

### 文件类字段内容

- DockerName: 关联docker 容器名称
- Namespace: 进程命名空间ID
- EventID: 事件ID
- Image: 进程
- hash: 进程文件hash
- TargetFile: 目标文件
- ParentName: 父进程名
- uid: 所属用户ID
- UserName: 用户名
- Gid: 所属用户组ID
- GroupName: 组名
- Euid: 执行权限
- time: 时间
- ppid: 父进程ID
- ptgid: 父进程TGID


### 网络类字段内容
- DockerName: 关联docker 容器名称
- Namespace: 进程命名空间ID
- EventID: 事件ID
- Image: 进程
- uid: 所属用户ID
- UserName: 用户名
- Gid: 所属用户组ID
- GroupName: 组名
- Euid: 执行权限
- time: 时间
- ppid: 父进程ID
- ptgid: 父进程TGID
- LocalIp: 本地地址
- RemoteIp: 远端地址
- LocalPort: 本地端口
- RemotePort: 远端端口
- QueryName: DNS查询域名