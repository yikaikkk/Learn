# Flux

## 背景

最近在用langchain4j做一些项目，其中涉及到使用调用大模型生成输出，其中就有两种输出模式，一种是一直阻塞等大模型完全生成完毕以后输出，另一种是非阻塞模式的输出，即让大模型生成一个token以后，就输出一个token。

## Flux

Flux有两种实现模式，一种是在SpringMVC中的实现方式，另一种是netty的实现

### springMVC中的FLux

在SpringMVC中，FLux的响应式是模拟的响应式

在SpringMVC中

请求线程（Tomcat 工作线程）

   ↓
   
Controller

   ↓

业务逻辑

   ↓

返回结果

   ↓

线程释放

所以在SpringMVC中实际是，当请求到达服务器，Controller会直接返回一个Flux，然后使用在线程池中注册一个Subscriber线程，然后这个线程开始emit数据


### netty中的FLux

之前看过muduo库的源码，一直觉得muduo的实现逻辑和netty的实现逻辑非常相似，都是使用IO多路复用

EventLoop（少量线程）

   ↓

Channel

   ↓

Pipeline

   ↓

Handler

在netty中使用的真正的响应式模式，利用EventLoop来监听事件，来处理。