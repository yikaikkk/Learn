# 博客集成ai

## 背景

打算在博客中集成一个ai模块，初期打算集成一个对话助手，后续打算增加rag和mcp，目前调研向量数据库chroma合适，相对于其他的向量数据库不占太多的内存

## 进展

增加了一个主要负责集成第三方AI大语言模型的模块blog-ai模块，目前集成了智谱AI

### 技术选项

- LangChain4j : Java版的LangChain框架，用于简化大语言模型的集成
- 智谱AI SDK : 用于接入智谱AI的GLM系列模型
- Spring Boot : 微服务基础框架
- Dubbo : RPC远程调用框架，向其他模块提供AI服务
- Reactor : 响应式编程框架，支持流式响应

### 一些其他考虑

增加了一个策略层，使用策略模式来选择使用的模型类型，同时后续扩展方便    

- AiModelStrategy类 : 核心AI模型管理策略类
  - 使用@PostConstruct注解在Bean初始化时自动加载和初始化AI模型
  - 维护两个模型映射表：普通同步模型(map)和流式模型(streamingModelMap)
  - 提供chat()方法进行同步对话
  - 提供chatStream()方法进行流式对话，返回Flux
    支持Server-Sent Events

### 一些问题

流式响应在Dubbo调用时可能遇到序列化问题，需要特殊处理。目前直接直接采用整体输出的方式

### 一些考虑
后续考虑在迭代流式输出时使用回调的方式来处理单个token输出的问题