## Spring Framework总览

### 什么是Spring Framework?

Spring是一个企业级JAVA应用开发框架，提供非常全面的基础设施，让开发人员更好的专注于业务开发。

### Spring Framework有哪些核心模块？

spring-core：Spring基础API模块，如资源管理，泛型处理

spring-beans：Spring Bean相关，如依赖注入，依赖查找

spring-aop：如动态代理，AOP字节码提升

spring-context：事件驱动，注解驱动，模块驱动等

spring-expression：Spring表达式语言模块

### Spring Framework有什么优势和不足？

// **FIX**

优势：

不足：开发人员不理解底层原理，出了问题不易调试。

## 反转控制IoC

###什么是IoC？

IoC是反转控制，主要实现有依赖注入和依赖查找。

### 什么是Spring IoC容器

是spring对于IoC的具体实现，也是依赖注入的实现方式。

### 依赖查找和依赖注入的区别？

依赖查找是一种主动或手动的依赖获取方式，通常需要容器的标准API实现。依赖注入是一种自动获取依赖的方式，无需依赖特定容器和API。

### Spring作为IoC容器的优势？

1. 提供依赖查找和依赖注入两种IoC实现
2. AOP抽象
3. 事务抽象
4. 事件机制
5. 强大的第三方整合
6. 易测试性

### BeanFactory和FactoryBean区别？

BeanFactory是IoC底层容器。

FactoryBean是创建Bean的一种方式，帮助实现复杂的初始化逻辑。

### Spring IoC容器启动时做了哪些准备？

// **FIX**

IoC配置元信息读取与解析，IoC容器生命周期，Spring事件发布，国际化等

### 如何注册一个Spring Bean？

通过BeanDefinition和外部单体对象来注册。

### 什么是Spring BeanDefinition？

用来定义Spring Bean，包含了Bean元信息（scope, role, parent, classname, lazy）。

### Spring 容器怎样管理注册Bean?

// **FIX**

IoC配置元信息读取和解析、依赖查找和注入以及Bean生命周期等。

### ObjectFactory和BeanFactory的区别？

ObjectFactory和BeanFactory均提供依赖查找的能力。ObjectFactory仅关注一个或一种类型的Bean依赖查找，并且自身不具备依赖查找的能力，能力由BeanFactory输出。BeanFactory则提供了单一类型，集合类型以及层次性等多种依赖查找。

### BeanFactory.getBean操作是否线程安全？

是线程安全的，操作过程会增加互斥锁。

### Spring依赖查找与注入在来源上的区别？

依赖查找使用getBean的方式获取依赖，来源包括BeanDefinition以及单例对象，依赖注入使用ResolverDependency来获取依赖，来源包括BeanDefinition、单例对象、Resolvable Dependency（BeanFactory, ResourceLoader, ApplicationEventPublisher, ApplicationContext）以及@Value所标注的外部化配置。

### 有多少种依赖注入的方式

- 构造器注入，用于少依赖，强制依赖场景
- Setter注入，用于多依赖，非强制依赖场景
- 字段注入，开发比较便利
- 方法注入，通常用来做Bean的声明
- 接口回调注入

### 偏好构造器注入还是Setter注入？

两种依赖注入方式均可使用，如果是必须依赖，推荐使用构造器注入，Setter注入用于可选依赖。

### 单例对象能在IoC容器启动后注册吗？

可以的，BeanDefinition会被BeanFactory#freezeConfiguration()方法影响，从而冻结注册，单例对象则没有这个限制。 

