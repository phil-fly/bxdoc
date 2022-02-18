

### 先看规则:

```yml
id: d2d642d7-b393-43fe-bae4-e81ed5915c11
meta:
  author: phil
  description: 被控端发起请求到该端口，并将其命令行的输入输出转到控制端.
  title: reverse shell
and: true
enabled: true
rules:
  Image:
    data:
      - '/bash'
    type: endswith
  EventID:
    data:
      - '3001'
    type: string
: killpid(currentPid)
system: linux
level: high
```

规则描述：
发现bash进程外联(EventID=3001)事件时,认为发生reverse shell事件，通过threat_disposition指定方法进行威胁处置.

- currentPid: 不同namespace下进程映射在宿主机以后的pid
- killpid: 执行进程查杀的func

### 测试过程：

1、外部服务器nc监听.本地环境
```
nc -lvp 8888
```
2、靶机环境宿主机安装"云影-agent"，同时安装docker容器
3、加载规则
4、进入docker，执行bash反弹
5、触发云影规则进行执行killpid(currentPid)查杀进程.


云影日志
```
time="2021-12-20 17:27:13" level=info msg="{\"DockerName\":\"taoyi\",\"EventID\":\"3001\",\"GroupName\":\"root\",\"Image\":\"/bin/bash\",\"LocalIp\":\"172.17.0.3\",\"LocalPort\":\"2570\",\"Namespace\":\"4026532387\",\"RawPid\":\"9997\",\"RemoteIp\":\"10.10.27.66\",\"RemotePort\":\"8888\",\"UserName\":\"root\",\"euid\":\"0\",\"gid\":\"0\",\"pid\":\"22\",\"ppid\":\"9973\",\"ptgid\":\"9973\",\"time\":\"223206408632-01-28T08:51:12+08:00\",\"uid\":\"0\"}"
```


reverse shell阻断：

[进程查杀](images/killbash1.png)