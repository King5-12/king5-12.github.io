---
nav:
  title: go
---

一个mysql服务包含
- connection Pool 用户鉴权、安全管理（用户执行操作权限校验）、连接处理（响应客户端连接请求、线程池资源管理）
- Services & utilities 管理服务&工具集。 备份恢复、安全管理、集群管理服务&工具。
- SQL interface ，SQL接口。接受用户sql命令并处理。
- parser ，SQL解析器。解析查询语句，生成语法树，解析器会查询缓存。
- Optimizer ，查询优化器。对查询语句进行优化，选择合适索引
- Caches 缓存。全局缓存和session缓存
- Pluggable Storage Rngines , 存储引擎层，一种文件访问机制，与文件交互，插件时存储引擎。
- 物理文件存储层，包括错误日志、慢查寻日志等。