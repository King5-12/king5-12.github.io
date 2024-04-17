---
nav:
  title: mysql
---

一个 mysql 服务包含

- connection Pool 用户鉴权、安全管理（用户执行操作权限校验）、连接处理（响应客户端连接请求、线程池资源管理）
- Services & utilities 管理服务&工具集。 备份恢复、安全管理、集群管理服务&工具。
- SQL interface ，SQL 接口。接受用户 sql 命令并处理。
- parser ，SQL 解析器。解析查询语句，生成语法树，解析器会查询缓存。
- Optimizer ，查询优化器。对查询语句进行优化，选择合适索引
- Caches 缓存。全局缓存和 session 缓存
- Pluggable Storage Rngines , 存储引擎层，一种文件访问机制，与文件交互，插件时存储引擎。
- 物理文件存储层，包括错误日志、慢查寻日志等。

### 创建数据库

创建数据表时的列参数

- PK：primary key 主键
- NN：not null 非空
- UQ：unique 唯一索引
- BIN：binary 二进制数据(比 text 更大的二进制数据)
- UN：unsigned 无符号 整数（非负数）
- ZF：zero fill 填充 0 例如字段内容是 1 int(4), 则内容显示为 0001
- AI：auto increment 自增
- G：generated column 生成列
