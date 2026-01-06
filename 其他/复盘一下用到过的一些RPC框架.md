# 复盘一下用到的一些RPC框架

## 一些概念
RPC：RPC 是 Remote Procedure Call（远程过程调用） 的缩写。RPC 是 Remote Procedure Call（远程过程调用） 的缩写。

RPC解决了什么问题：
在分布式系统里：

	•	服务 A 想调用服务 B 的能力
	•	又不想关心：
	•	HTTP 请求怎么拼
	•	参数怎么编码
	•	网络异常怎么处理
    
RPC 把这些复杂性都封装掉了

## GRPC
在之前的一个项目中，一个服务在调用了一个服务的一些函数时使用了GRPC

在使用Grpc框架是与dubbo的最大区别就是需要自定义protobuf文件

gRPC = RPC 框架 + 协议规范

它主要做了三件事：

	1.	用 IDL 定义接口（Protocol Buffers）
	2.	自动生成客户端 / 服务端代码
	3.	高效的网络通信（HTTP/2）

grpc有几个优点：

	1.	Protobuf（二进制）
	•	比 JSON 小、快
	2.	HTTP/2 多路复用
	•	减少 TCP 连接
	3.	长连接
	•	降低建连成本
	4.	代码直调
	•	无需手写解析逻辑

## DUBBO

在这个博客项目拆分的微服务版中，不同微服务间的调用使用的就是nacos+dubbo的模式

相对于grpc

Dubbo = RPC + 服务治理

Dubbo采用的是hessian2的序列化方式，和grpc的protobuf的序列化模式不同


## McPacketProxy

这是baidu内部的一个rpc框架，和其他两个不同的是采用的McPacket的序列化方式



## 一些对比
| 维度 | Dubbo | gRPC |
|------|------|-------|
|生态	|Java 为主|	多语言|
|接口定义|	Java Interface|	Protobuf IDL|
|性能	|高	|很高|

## 性能对比
### 序列化：Protobuf 更稳定、更轻

gRPC

	•	强制 Protobuf
	•	二进制、紧凑
	•	字段编号 + Varint
	•	跨语言一致

Dubbo

	•	可选序列化
	•	Hessian
	•	Kryo
	•	Fastjson
	•	Java 对象模型复杂
	•	反射 / 类信息参与


### 传输协议：HTTP/2 vs 自定义 RPC

gRPC

	•	标准 HTTP/2
	•	天生支持：
	•	多路复用
	•	流控
	•	Header 压缩
	•	一个 TCP 连接并发多个 RPC

Dubbo

	•	自定义 Dubbo 协议（或 tri）
	•	多路复用能力依赖实现
	•	老版本：
	•	请求-响应模型更重


## Todo
细化一下