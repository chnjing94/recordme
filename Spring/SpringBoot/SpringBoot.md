### 用 java -jar启动jar包形式的SpringBoot应用，具体执行流程

java解压后有三个子文件夹BOOT-INF/，META-INF/，org/。根据JAR规范，会首先执行META-INF/MANIFEST.MF里指定的Main-Class，SpringBoot指定的Main-Class是/org/springframework/boot/loader/JarLauncher，JarLauncher.class里面指定了ClassPath，以及lib文件目录，在它的main方法里执行launch方法来启动应用程序。具体执行的入口方法是在MANIFEST.MF里指定的Start-Class，也就是我们用@SpringBootApplication标注的类的main函数(如果该类里没有main函数，就会扫描项目所有类中的main函数，如果有多个main函数就会报错)。

### SpringBoot怎么读取注解

通过AnnotationMetadata API获取，该API有两种实现方式，一种是通过Java反射实现，一种通过ASM实现。

ASM的性能会比反射方式好（反射需要排除Java标准注解）。反射要求类被ClassLoader加载，会导致package下所有类被加载，而ASM是按需加载。

### SpringApplication初始化阶段

- 构造阶段：在使用SpringApplication.run()时，相当于调用new SpringApplication(Class, args)构造函数。Class参数作为引导类，也称作primarySource。之后做以下三件事：
  - 推断Web应用类型，有非Web类型和Web类型，Web类型又包括Reactive和Servlet。
  - 加载Spring应用上下文初始化器，加载spring.factories里配置的ApplicationContexInitializer。
  - 加载Spring应用事件监听器，加载spring.factories里配置的ApplicationListenser。
  - 推导应用引导类，也就是运行该main方法的类。
- 配置阶段：
  - 调整SpringApplication设置，例如设置WebApplicationType。
  - 增加SpringApplication配置源，配置源可以是主配置类，@Configuration Class，XML配置文件和package。
  - 调整SpringApplication外部化配置，可以调整application.properties文件的搜索路径。

### SpringApplication运行阶段

#### SpringApplication准备阶段

准备阶段要准备以下一些对象：

- SpringApplicationRunListeners, 作为集合容器，装载多个SpringApplicationRunListener。

- SpringApplicationRunListener, 作为SpringBoot运行时监听器，用户可以定义在spring.factories中，在SpringBoot应用运行的各个阶段执行相应的回调。

- ApplicationArguments，将SpringApplication启动参数封装到一个对象中。

- ConfigurableEnvironment

- ConfigurableApplicationContext, 作为Spring应用上下文。

  - 根据WebApplicationType创建Spring应用上下文，实际对象为AnnotationConfigurableApplicationContext。 

  - 也可使用SpringApplicationBuilder#contextClass指定ConfigurableApplicationContext。

    

#### Spring应用上下文运行前准备

1. Spring应用上下文准备阶段

   1. 设置ConfigurableEnvironment.
   2. 运行Spring应用上下文后置处理-SpringApplication#postProcessApplicationContext。用于对ConfigurableApplicationContext的后置处理，可以允许子类覆盖其实现增加额外功能，例如改变BeanName的生成规则。
   3. 运行Spring应用上下文初始化器（ApplicationContextInitializer），也就是在SpringApplication构造阶段加载的初始化器。
   4. 执行SpringApplicationRunListenser#contextPrepared方法回调

2. Spring应用上下文装载阶段

   1. 注册Spring Boot Bean，将ApplicationArguments对象以及可能存在的Banner实例注册为Spring单体Bean。
   2. 合并Spring应用上下文配置源，合并primarySource，Configuration Class，XML配置资源路径等成为一个Set。
   3. 加载Spring应用上下文配置源。调用多种BeanDefinitionLoader，将来自注解，XML的配置源解析成为BeanDefinition。
   4. 执行SpringApplicationRunListenser#contextLoaded。

3. Spring应用上下文启动阶段

   本阶段调用refreshContext(ConfigurableApplicationContext)。执行ApplicationContext的启动，注册shutdownHook线程，实现Spring Bean销毁生命周期回调。随着refreshContext，Spring应用上下文正式进入Spring生命周期，SprintBoot核心特性也随之启动，如自动装配，嵌入式容器启动，生产就绪等。同时触发ContextRefreshEvent，并进入ApplicationContext启动后阶段。

4. Spring应用上下文启动后阶段：没有具体实现，全部交给开发人员实现。

### SpringApplication 结束阶段

SpringApplication#run方法正常结束时，会触发SpringApplicationListenser#running回调函数，EventPublishingRunListenser作为SpringApplicationListenser唯一内建实现。会在running中触发ApplicationReadyEvent。

SpringApplication异常结束时，会触发SpringApplicationListenser#failed回调函数，

### SpringBoot应用退出

SpringApplication会注册shutdownhook线程，当JVM退出时，保证所有Spring上下文管理的Bean都能在标准生命周期中回调。

通过定义ExitCodeGenerator Bean，来更改SpringBoot退出码，并通过ApplicationListenser来监听ExitCodeEvent，得到退出码。

当SpringBoot异常结束时，提供两种退出码与异常类型的关联方式

1. 让Throwable对象实现ExitCodeGenerator接口，不依赖ConfigurableApplicationContext活跃。
2. ExitCodeExceptionMapper实现退出码与Throwable的映射，依赖ConfigurableApplicationContext活跃。













### Spring事件/监听机制

Spring事件：ApplicationEvent，继承自java.util.EventObject，Spring Framework内建五种事件，如ContextRefreshedEvent。事件源为ApplicationContext。

Spring事件监听手段：1. 面向接口ApplicationListenser编程 2.注解@EventListenser。

Spring事件广播器：1. ApplicationEventPublisher#publishEvent 2. ApplicationEventMulticaster#multicastEvent

### SpringBoot事件/监听机制

SpringBoot包含一些特有的事件（与Spring相比），如ApplicationStartingEvent, ApplicationPreparedEvent。

SpringBoot在启动过程中也能监听到Spring事件。

Spring事件是由ApplicationContext对象触发的，而SpringBoot事件发布者则是SpringApplication.initialMulticaster。

SpringBoot事件：继承ApplicationEvent和SpringApplicationEvent。事件源为SpringApplication。发布顺序依次为：

- ApplicationStartingEvent
- ApplicationEnvironmentPreparedEvent
- ApplicationPreparedEvent
- ApplicationStartedEvent
- ApplicationReadyEvent
- ApplicationFailedEvent

SpringBoot事件监听手段：

1. 加载META-INF/spring.factories资源中的ApplicationListenser，并将其关联到SpringApplication。
2. 通过SpringApplication#addListensers(ApplicationListensers...)或SpringApplicationBuilder#listensers(ApplicationListensers...)方法显示添加。

SpringBoot事件广播器：与Spring事件广播器无异，都是ApplicationEventMulticaster，SpringBoot发布的事件是特定的。

