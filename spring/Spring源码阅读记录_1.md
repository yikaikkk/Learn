# bean的获取 & 三级缓存 & 解决循环依赖
目前项目开发一直在用Springboot，之前一直没有系统性的看过Spring的源码。今天打算开始学习spring的源码，打算先从spring的核心功能部分代码看起。

## bean的获取
在spring的源码中，对于获取bean的代码在 `protected <T> T doGetBean` 中。

在代码的第一部分
``` java
String beanName = transformedBeanName(name);
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
    ...
    beanInstance = getObjectForBeanInstance(sharedInstance, requiredType, name, beanName, null);
}
```
函数先对传入的name进行了处理，去掉别名、去掉 & 前缀等，得到规范化的 beanName。同时，先去获取一下是否已经存在单例的bean对象。并且只有在 args == null 时才直接使用缓存，因为传入构造参数时意味着请求特殊构造——不能直接返回已有缓存。

如果命中缓存，使用 getObjectForBeanInstance(...) 进一步处理（例如：如果缓存的是 FactoryBean、要不要解包等），然后直接返回（后续还有 adapt 步骤）。


在第二部分中
```java
if (isPrototypeCurrentlyInCreation(beanName)) {
    throw new BeanCurrentlyInCreationException(beanName);
}
```
如果正在创建同名 prototype，直接抛异常。


接下来
```java
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
   // delegate to parent
}
```
如果当前的bean没有BeanFactory，并且有父BeanFactory，就使用父容器来创建。

然后
```java
if (!typeCheckOnly) {
	markBeanAsCreated(beanName);
}
```
标记 bean 为“已创建”（除非只是做类型检查）
