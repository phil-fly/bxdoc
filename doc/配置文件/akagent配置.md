## 配置样例：

```
uuid: 31ef3e81-d646-5bc4-5fc2-07353c05289f   #上报事件设备标识ID
processMonitor:                              #进程监控配置
  enable: true
fileMonitor:                                 #文件监控配置
  enable: true
  monitorPath:                               #指定监控路径
    '/': default
  fileResave:                                #文件另存配置
    enable: false
    savePath: /opt/file/monitor/
    maxSizeMB: 100
netMonitor:                                  #网络监控配置
  enable: true
report:                                      #事件上报配置
  enable: true
  type: logfile                              #上报方式: logfile,http,kafka,stdout
filterDir: /opt/bxsec/filter                 #指定降噪规则存储目录
```



## 说明：
### filterDir
降噪规则目录

### process_cfg配置

- enable：是否打开进程监控.



### file_cfg配置

- enable：是否打开文件监控.

- monitor_path_map：监控路径列表.
  - key：监控路径
  - value：说明
  
- file_resave：监控文件另存.

  - enable：是否打开文件另存.

  - save_path：文件保存路径.

  - maxSizeMB：文件大小限制.

    

### net_cfg配置

- enable：是否打开网络监控.

### report配置

- enable：是否打开上报.

- type：输出方式

  - logfile ：日志文件输出，服务端filebeat采集
  - stdout：终端打印
  - kafka：
  - http：http长连接输出.

- http_report：(type = http)

  - target_url_prefix：目的url前缀。例如配置：https://127.0.0.1:12345/hostmonitor/  （上报地址如下）
    - process：https://127.0.0.1:12345/hostmonitor/process
    - file：https://127.0.0.1:12345/hostmonitor/file
    - net: https://127.0.0.1:12345/hostmonitor/net

- kafka_report:
  - tls_enblse：是否打开tls.
- Brokers



## 其它
监控主机指定目录存储（存储目录与监控目录不允许存在逻辑包含关系，避免存在事件回环）

文件另存后新的文件名格式：
```
指定路径/容器名/{文件名}-{unix时间戳}
```