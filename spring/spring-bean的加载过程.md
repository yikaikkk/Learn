# Bean加载过程

在spring中，refresh函数是一个很重要的函数,bean的加载过程就是通过refresh函数实现的

Spring 容器加载 Bean 的大致过程如下 ：

```
refresh()
 ├─> obtainFreshBeanFactory()
 │     └─> loadBeanDefinitions()      ← 加载 Bean 定义（解析 XML / 注解）
 ├─> prepareBeanFactory()
 ├─> postProcessBeanFactory()
 ├─> invokeBeanFactoryPostProcessors()
 ├─> registerBeanPostProcessors()
 ├─> initMessageSource() / initEventMulticaster()
 ├─> finishBeanFactoryInitialization() ← 实例化所有非懒加载单例 Bean
 └─> finishRefresh()
```

整个过程主要分为 **两大阶段**：

| 阶段 | 关键操作 |
|------|-----------|
| BeanDefinition 阶段 | 读取并注册 Bean 元信息（配置信息） |
| Bean 实例化阶段 | 创建 Bean 对象，执行依赖注入、初始化、后置处理等 |

---

refresh()的具体代码如下

```java
try {
			this.startupShutdownThread = Thread.currentThread();

			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");

			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);
				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);
				beanPostProcess.end();

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}
```

## BeanDefinition 阶段（加载配置信息）

这一阶段的目标是把配置（XML / 注解 / @Bean 方法）解析成 `BeanDefinition` 并注册到容器。

### 1️⃣ 从 XML 读取的入口

```java
AbstractApplicationContext#obtainFreshBeanFactory()
   ↓
AbstractRefreshableApplicationContext#refreshBeanFactory()
   ↓
AbstractXmlApplicationContext#loadBeanDefinitions()
   ↓
XmlBeanDefinitionReader#doLoadBeanDefinitions()
```

核心逻辑：

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    InputStream inputStream = encodedResource.getInputStream();
    Document doc = doLoadDocument(inputSource, resource);
    return registerBeanDefinitions(doc, resource);
}
```

然后会调用 `finishBeanFactoryInitialization(beanFactory)`创建对象

## Bean 实例化阶段（创建对象）

触发点在：
```java
finishBeanFactoryInitialization(beanFactory)
```

内部调用：

```java
beanFactory.preInstantiateSingletons()
```

---

### 执行链概览：

```
DefaultListableBeanFactory.preInstantiateSingletons()
   ↓
AbstractBeanFactory.getBean(beanName)
   ↓
doGetBean()
   ↓
createBean()
   ↓
doCreateBean()
   ↓
populateBean()      ← 属性注入
   ↓
initializeBean()    ← 初始化、后置处理
```

---

## `doGetBean()` 核心源码分析

```java
protected <T> T doGetBean(String name, @Nullable Class<T> requiredType,
                          @Nullable Object[] args, boolean typeCheckOnly) {

    // 1️⃣ 检查是否已创建过（单例缓存）
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null) {
        return (T) getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }

    // 2️⃣ 获取 BeanDefinition
    RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

    // 3️⃣ 确保依赖的 Bean 已加载
    String[] dependsOn = mbd.getDependsOn();
    for (String dep : dependsOn) getBean(dep);

    // 4️⃣ 创建 Bean
    Object beanInstance = createBean(beanName, mbd, args);

    // 5️⃣ 如果是单例，放入缓存
    registerSingleton(beanName, beanInstance);

    return (T) getObjectForBeanInstance(beanInstance, name, beanName, mbd);
}
```

---

## Bean 的创建过程（`createBean()`）

真正创建 Bean 的地方在：

```java
AbstractAutowireCapableBeanFactory#doCreateBean()
```

关键步骤如下：

```java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, Object[] args) {
    // 1️⃣ 实例化 (Constructor / FactoryMethod)
    Object beanInstance = createBeanInstance(beanName, mbd, args);

    // 2️⃣ 属性注入 (依赖注入)
    populateBean(beanName, mbd, beanInstance);

    // 3️⃣ 初始化（Aware 回调 + init-method）
    Object exposedObject = initializeBean(beanName, beanInstance, mbd);

    // 4️⃣ 注册到单例池
    registerSingleton(beanName, exposedObject);

    return exposedObject;
}
```

---

###  实例化阶段

```java
protected Object createBeanInstance(...) {
    return instantiateBean(beanName, mbd);
}
```

底层调用：
```java
BeanUtils.instantiateClass(Constructor)
```
→ 反射创建实例。

如果是 `@Configuration` + `@Bean`，则会通过 `CGLIB` 增强生成代理类。

---

###  属性注入阶段

```java
populateBean(beanName, mbd, instanceWrapper)
```

核心逻辑：
- 解析依赖（`@Autowired`, `@Value`, XML `<property>`）
- 从容器中获取依赖 Bean
- 反射注入到目标属性

---

###  初始化阶段

```java
initializeBean(beanName, exposedObject, mbd)
```

执行流程：

1. **Aware 接口回调**
   - `BeanNameAware`
   - `BeanFactoryAware`
   - `ApplicationContextAware`

2. **BeanPostProcessor.beforeInitialization()**

3. **调用初始化方法**
   - `@PostConstruct`
   - `InitializingBean#afterPropertiesSet()`
   - `<bean init-method="">`

4. **BeanPostProcessor.afterInitialization()**
   - 比如 `AOP` 的代理创建就是在这里完成的！

---

## AOP 代理的时机

Spring 的 AOP 是在 Bean 初始化阶段通过 `BeanPostProcessor` 实现的。  
关键类是：
```java
AbstractAutoProxyCreator#postProcessAfterInitialization()
```

在这个时机，会判断：
- 该 Bean 是否匹配切面；
- 如果匹配，就用 CGLIB 或 JDK Proxy 包装成代理对象；
- 最终放入单例池的是 **代理对象**。

---

## 三级缓存防循环依赖

Spring 为了解决单例 Bean 的循环依赖问题，在创建 Bean 时使用了三级缓存机制：

| 缓存 | 含义 |
|------|------|
| `singletonObjects` | 一级缓存：已创建好的单例 |
| `earlySingletonObjects` | 二级缓存：提前暴露的半成品 Bean |
| `singletonFactories` | 三级缓存：创建代理对象的工厂 |

在 `doCreateBean()` 中，Spring 会先把 Bean 工厂放入三级缓存，让依赖的 Bean 可以提前拿到引用，从而解决循环依赖。

```
调用方
   │
   ▼
getBean("A") 
   │
   ▼
doGetBean("A")
   │
   ▼
getSingleton("A")
   │
   ├─► 一级缓存 singletonObjects 有吗？ ──► 有 → 返回，结束
   │
   ├─► 二级缓存 earlySingletonObjects 有吗？ ──► 有 → 返回早期对象
   │
   ├─► 三级缓存 singletonFactories 有吗？ 
   │       │
   │       ├─► 有 → 调用 ObjectFactory.getObject()
   │       │       ├─► 生成早期对象（可能是代理）
   │       │       ├─► 放入二级缓存 earlySingletonObjects
   │       │       └─► 从三级缓存删除
   │       │
   │       └─► 无 → 第一次创建 Bean → 调用 createBean("A")
   │
   ▼
createBean("A")
   │
   ├─► 实例化对象（构造函数）
   │
   ├─► addSingletonFactory(beanName, ObjectFactory) 
   │       └─► 把创建早期对象的工厂放入三级缓存
   │
   ├─► 属性注入 / 依赖注入（可能触发循环依赖）
   │
   ├─► 初始化后置处理 / AOP 代理生成
   │
   └─► addSingleton(beanName, fullyInitializedBean)
           ├─► 放入一级缓存 singletonObjects
           ├─► 清理二级缓存 earlySingletonObjects
           └─► 清理三级缓存 singletonFactories
```

---

## 总结：Spring Bean 加载的完整生命周期

```
BeanDefinition阶段：
   ↓
读取配置 → 解析成 BeanDefinition → 注册到 BeanFactory
   ↓
Bean实例化阶段：
   ↓
创建实例 → 属性注入 → 初始化 → AOP代理 → 放入单例池
```

最终，容器中缓存的 Bean 就可以被应用程序正常使用。

---

## 总结

> Spring Bean 的加载过程：
> - **先加载配置元信息（BeanDefinition）**  
> - **再通过反射创建对象（实例化）**  
> - **再注入依赖（DI）**  
> - **再执行初始化方法与 AOP 增强（生命周期）**  
> - **最后缓存到单例池中，供容器复用**

---





