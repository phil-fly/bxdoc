## 配置样例：

```
{
  "process_cfg": {
    "enable": true,	
    "driver_file":"/opt/mount/process"
  },
  "file_cfg": {
    "enable": true,
    "driver_file":"/opt/mount/file",
    "monitor_path_map": {
      "/": "default"
    },
    "file_resave": {
      "enable": true,
      "save_path": "/opt/file/monitor/",
      "maxSizeMB": 100
    }
  },
  "net_cfg": {
    "enable": true,
    "driver_file":"/opt/mount/net"
  },
  "report": {
    "enable": true,
    "type": "logfile"
  },
  "filterDir": "/opt/bxsec/filter"
}
```



## 说明：

### process_cfg配置

- enable：是否打开进程监控.
- driver_file：驱动接口文件.



### file_cfg配置

- enable：是否打开文件监控.

- driver_file：驱动接口文件.

- monitor_path_map：监控路径列表.
  - key：监控路径
  - value：说明
  
- file_resave：监控文件另存.

  - enable：是否打开文件另存.

  - save_path：文件保存路径.

  - maxSizeMB：文件大小限制.

    

### net_cfg配置

- enable：是否打开网络监控.
- driver_file：驱动接口文件.

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