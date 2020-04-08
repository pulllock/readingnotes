# 整体架构

- 调用方，手机端、接入方等等一系列调用方
- LVS负载均衡
- Nginx反向代理
- APIGateway
  - 流控，控制流量，针对同一个ip在指定的时间段内访问次数做限制
  - 验签解密，校验参数、验证签名信息、将加密的信息解密
  - 接口验证，验证接口是否存在、接口信息是否是当前调用者的接口
  - 接口权限验证，ip黑名单校验、调用的ip是否在白名单内
  - 业务参数验证，校验业务接口参数是否正确
  - 调用业务接口，可以使用dubbo泛化调用
  - 熔断降级，业务方接口不可用的时候或者业务方处理速度变慢，考虑进行熔断降级
  - 加密返回，将调用结果封装、加密、返回
- 注册中心，dubbo服务注册到注册中心zookeeper
- 业务服务，各个业务提供的dubbo服务，服务注册到注册中心
- 存储
  - 本地缓存
  - 分布式缓存
  - MySQL

# 数据库设计

- agw_api，接口信息
- agw_api_param，接口对应的参数信息
- agw_sys，接口所属业务系统
- agw_out，外部调用方
- agw_out_api，外部调用方拥有的api
- agw_out_ip，外部调用方的白名单配置
- agw_black_ip，黑名单ip

## agw_api

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注                     |
| ------------- | ------------ | -------- | ------- | ------ | ------------------------ |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID                   |
| created_time  | datetime     | N        |         |        | 创建时间                 |
| modified_time | datetime     | N        |         |        | 修改时间                 |
| version       | smallint(6)  | N        |         |        | 版本号                   |
| code          | varchar(255) | N        |         |        | api唯一标识              |
| name          | varchar(255) | N        |         |        | api接口名                |
| method        | varchar(255) | N        |         |        | api方法名                |
| alias         | varchar()    | Y        |         |        | api方法别名              |
| sys_id        | bigint(20)   | N        |         |        | 所属业务系统id           |
| timeout       | int(6)       | N        |         | 1000   | 超时时间，毫秒           |
| ip_control    | tinyint(4)   | Y        |         | 0      | 是否白名单控制 0-否 1-是 |

## agw_api_param

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注     |
| ------------- | ------------ | -------- | ------- | ------ | -------- |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID   |
| created_time  | datetime     | N        |         |        | 创建时间 |
| modified_time | datetime     | N        |         |        | 修改时间 |
| version       | smallint(6)  | N        |         |        | 版本号   |
| api_id        | varchar(255) | N        |         |        | api id   |
| name          | varchar(255) | N        |         |        | 参数名   |
| type          | varchar(255) | N        |         |        | 参数类型 |
| order         | smallint(6)  | N        |         |        | 参数顺序 |

## agw_sys

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注         |
| ------------- | ------------ | -------- | ------- | ------ | ------------ |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID       |
| created_time  | datetime     | N        |         |        | 创建时间     |
| modified_time | datetime     | N        |         |        | 修改时间     |
| version       | smallint(6)  | N        |         |        | 版本号       |
| name          | varchar(255) | N        |         |        | 业务系统名   |
| desc          | varchar(255) | N        |         |        | 业务系统描述 |

## agw_out

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注             |
| ------------- | ------------ | -------- | ------- | ------ | ---------------- |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID           |
| created_time  | datetime     | N        |         |        | 创建时间         |
| modified_time | datetime     | N        |         |        | 修改时间         |
| version       | smallint(6)  | N        |         |        | 版本号           |
| name          | varchar(255) | N        |         |        | 外部系统名       |
| desc          | varchar(255) | N        |         |        | 外部系统描述     |
| code          | varchar(255) | N        |         |        | 外部系统唯一标识 |

## agw_out_api

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注         |
| ------------- | ------------ | -------- | ------- | ------ | ------------ |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID       |
| created_time  | datetime     | N        |         |        | 创建时间     |
| modified_time | datetime     | N        |         |        | 修改时间     |
| version       | smallint(6)  | N        |         |        | 版本号       |
| out_id        | bigint(20)   | N        |         |        | 外部系统id   |
| api_id        | varchar(255) | N        |         |        | 外部系统描述 |

## agw_out_ip

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注           |
| ------------- | ------------ | -------- | ------- | ------ | -------------- |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID         |
| created_time  | datetime     | N        |         |        | 创建时间       |
| modified_time | datetime     | N        |         |        | 修改时间       |
| version       | smallint(6)  | N        |         |        | 版本号         |
| out_id        | bigint(20)   | N        |         |        | 外部系统id     |
| ip            | varchar(255) | N        |         |        | 外部系统白名单 |

## agw_black_ip

| 名称          | 类型         | 是否为空 | 索引    | 默认值 | 备注       |
| ------------- | ------------ | -------- | ------- | ------ | ---------- |
| id            | bigint(20)   | N        | PRIMARY |        | 主键ID     |
| created_time  | datetime     | N        |         |        | 创建时间   |
| modified_time | datetime     | N        |         |        | 修改时间   |
| version       | smallint(6)  | N        |         |        | 版本号     |
| out_id        | bigint(20)   | Y        |         |        | 外部系统id |
| ip            | varchar(255) | N        |         |        | 黑名单ip   |