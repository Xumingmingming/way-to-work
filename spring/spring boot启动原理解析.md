我们开发任何一个Spring Boot项目，都会用到如下的启动类：

~~~java
 @SpringBootApplication
 public class Application {
     public static void main(String[] args) {
         SpringApplication.run(Application.class, args);
     }
 }
~~~

从上面代码可以看出，Annotation定义（==@SpringBootApplication==）和类定义（==SpringApplication.run==）最为耀眼，所以要揭开SpringBoot的神秘面纱，我们要从这两位开始就可以了。

# **1 SpringBootApplication背后的秘密**

@SpringBootApplication注解是Spring Boot的核心注解，它其实是一个组合注解：

~~~java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
...
}
~~~

虽然定义使用了多个Annotation进行了原信息标注，但实际上重要的只有三个Annotation：

- ==@SpringBootConfiguration==（@SpringBootConfiguration点开查看发现里面还是应用了@Configuration）
- ==@EnableAutoConfiguration==
- ==@ComponentScan==

即 @SpringBootApplication = (默认属性)@SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan。

所以，如果我们使用如下的SpringBoot启动类，整个SpringBoot应用依然可以与之前的启动类功能对等：

~~~java
@Configuration
@EnableAutoConfiguration
@ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
~~~

每次写这3个比较累，所以写一个**@SpringBootApplication**方便点。接下来分别介绍这3个Annotation。

## **@SpringBootConfiguration**

这里的@Configuration对我们来说不陌生，它就是==JavaConfig==形式的Spring Ioc容器的配置类使用的那个@Configuration，SpringBoot社区推荐使用基于JavaConfig的配置形式，所以，这里的启动类标注了@Configuration之后，本身其实也是一个IoC容器的配置类。

举几个简单例子回顾下，XML跟config配置方式的区别：

###**1）表达形式层面**

基于XML配置的方式是这样：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.0.xsd"
       default-lazy-init="true">
    <!--bean定义-->
</beans>
~~~

而基于JavaConfig的配置方式是这样：

~~~java
@Configuration
 public class MockConfiguration{
     //bean定义
 }
~~~

任何一个标注了@Configuration的Java类定义都是一个JavaConfig配置类。

###**（2）注册bean定义层面**

基于XML的配置形式是这样：

~~~xml
 <bean id="mockService" class="..MockServiceImpl">
     ...
 </bean>
~~~

而基于JavaConfig的配置形式是这样的：

~~~java
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl();
    }
}
~~~

任何一个标注了**@Bean**的方法，其返==回值将作为一个bean定义注册到Spring的IoC容器==，方法名将默认成该bean定义的id。

###**（3）表达依赖注入关系层面**

为了表达bean与bean之间的依赖关系，在XML形式中一般是这样：

~~~xml
<bean id="mockService" class="..MockServiceImpl">
     <propery name ="dependencyService" ref="dependencyService" />
 </bean>
 
 <bean id="dependencyService" class="DependencyServiceImpl"></bean>
~~~

而基于JavaConfig的配置形式是这样的：

~~~java
@Configuration
public class MockConfiguration{
    @Bean
    public MockService mockService(){
        return new MockServiceImpl(dependencyService());
    }
    
    @Bean
    public DependencyService dependencyService(){
        return new DependencyServiceImpl();
    }
}
~~~

如果**一个bean的定义依赖其他bean**，则直接调用对应的JavaConfig类中依赖bean的创建方法就可以了。

###补充：

@Configuration：提到@**Configuration**就要提到他的搭档**@Bean**。使用这两个注解就可以创建一个简单的spring配置类，可以用来替代相应的xml配置文件。

~~~xml
<beans> 
     <bean id = "car" class="com.test.Car"> 
         <property name="wheel" ref = "wheel"></property> 
     </bean> 
     <bean id = "wheel" class="com.test.Wheel"></bean> 
</beans>
~~~

相当于：

~~~java
@Configuration 
public class Conf { 
    @Bean 
    public Car car() { 
        Car car = new Car(); 
        car.setWheel(wheel()); 
        return car; 
    }
    
    @Bean 
    public Wheel wheel() { 
        return new Wheel(); 
    } 
}
~~~

==@Configuration==的注解类**==标识这个类可以使用Spring IoC容器作为bean定义的来源==**。

==@Bean==注解**==告诉Spring，一个带有@Bean的注解方法将返回一个对象，该对象应该被注册为在Spring应用程序上下文中的bean==**。

##**@ComponentScan**

@ComponentScan这个注解在Spring中很重要，它对应XML配置中的元素，**@ComponentScan的功能其实就是自动扫描并加载符合条件的组件**（比如@Component和@Repository等）**或者bean定义**，==最终将这些bean定义加载到IoC容器中==。

​     我们可以通过basePackages等属性来细粒度的定制@ComponentScan自动扫描的范围，如果不指定，则==默认Spring框架实现会从声明@ComponentScan所在类的package进行扫描==。

注：所以SpringBoot的启动类最好是放在root package下，因为默认不指定basePackages。

##@EnableAutoConfiguration

​    个人感觉@EnableAutoConfiguration这个Annotation最为重要，所以放在最后来解读，大家是否还记得Spring框架提供的各种名字为@Enable开头的Annotation定义？比如@EnableScheduling、@EnableCaching、@EnableMBeanExport等，@EnableAutoConfiguration的理念和做事方式其实一脉相承，简单概括一下就是，**借助@Import的支持，==收集和注册特定场景相关的bean定义==。**

- @EnableScheduling是通过@Import将Spring调度框架相关的bean定义都加载到IoC容器。
- @EnableMBeanExport是通过@Import将JMX相关的bean定义加载到IoC容器。

而==@EnableAutoConfiguration也是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器==，仅此而已！

​    @EnableAutoConfiguration会根据类路径中的jar依赖为项目进行自动配置，如：添加了spring-boot-starter-web依赖，会自动添加Tomcat和Spring MVC的依赖，Spring Boot会对Tomcat和Spring MVC进行自动配置。

spring-boot-starter-web：

~~~xml
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starters</artifactId>
		<version>1.5.19.RELEASE</version>
	</parent>
	<artifactId>spring-boot-starter-web</artifactId>
	<name>Spring Boot Web Starter</name>
	<properties>
		<main.basedir>${basedir}/../..</main.basedir>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</dependency>
		<dependency>
			<groupId>org.hibernate</groupId>
			<artifactId>hibernate-validator</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-web</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-webmvc</artifactId>
		</dependency>
	</dependencies>
</project>

~~~

@EnableAutoConfiguration作为一个复合Annotation，其自身定义关键信息如下：

~~~java
@SuppressWarnings("deprecation")
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(EnableAutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    ...
}
~~~

其中，最关键的要属==**@Import(EnableAutoConfigurationImportSelector.class)**==，借助**EnableAutoConfigurationImportSelector**，@EnableAutoConfiguration可以帮助SpringBoot应用将所有符合条件的@Configuration配置都加载到当前SpringBoot创建并使用的IoC容器。就像一只“八爪鱼”一样，借助于Spring框架原有的一个工具类：**SpringFactoriesLoader的支持**，@EnableAutoConfiguration可以智能的自动配置功效才得以大功告成！

![249993-20171207163759488-1739516792](assets/249993-20171207163759488-1739516792.png)

### **自动配置幕后英雄：SpringFactoriesLoader详解**

**SpringFactoriesLoader**属于Spring框架私有的一种扩展方案，其主要功能就是**从指定的配置文件META-INF/spring.factories加载配置**。

~~~java
public abstract class SpringFactoriesLoader {
    //...
    public static <T> List<T> loadFactories(Class<T> factoryClass, ClassLoader classLoader) {
        ...
    }


    public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        ....
    }
}
~~~

配合@EnableAutoConfiguration使用的话，它更多是提供一种配置查找的功能支持，即==根据@EnableAutoConfiguration的完整类名org.springframework.boot.autoconfigure.EnableAutoConfiguration作为查找的Key，获取对应的一组@Configuration类==。

![屏幕快照 2019-04-11 下午9.45.04](assets/屏幕快照 2019-04-11 下午9.45.04.png)

上图就是从SpringBoot的autoconfigure依赖包中的META-INF/spring.factories配置文件中摘录的一段内容，可以很好地说明问题。

所以，@EnableAutoConfiguration自动配置的魔法其实就变成了：**==从classpath中搜寻所有的META-INF/spring.factories配置文件==，并将其中==org.springframework.boot.autoconfigure.EnableutoConfiguration==对应的配置项通过==反射==（Java Refletion）实例化为对应的标注了@Configuration的JavaConfig形式的IoC容器配置类，然后汇总为一个并加载到IoC容器。**

#**2、深入探索SpringApplication执行流程**

##主要流程

SpringApplication的==run方法==的实现是我们本次旅程的主要线路，该方法的主要流程大体可以归纳如下：

1） 如果我们使用的是SpringApplication的静态run方法，那么，这个方法里面首先要创建一个SpringApplication对象实例，然后调用这个创建好的SpringApplication的实例方法。在SpringApplication实例初始化的时候，它会提前做几件事情：

- 根据classpath里面是否存在某个特征类（org.springframework.web.context.ConfigurableWebApplicationContext）来决定是否应该创建一个为Web应用使用的ApplicationContext类型。
- 使用SpringFactoriesLoader在应用的classpath中查找并加载所有可用的ApplicationContextInitializer。
- 使用SpringFactoriesLoader在应用的classpath中查找并加载所有可用的ApplicationListener。
- 推断并设置main方法的定义类。

2） SpringApplication实例初始化完成并且完成设置后，就开始执行run方法的逻辑了，方法执行开始，首先遍历执行所有通过SpringFactoriesLoader可以查找到并加载的==SpringApplicationRunListener==。调用它们的started()方法，告诉这些SpringApplicationRunListener，“嘿，SpringBoot应用要开始执行咯！”。

3） 创建并配置当前SpringBoot应用将要使用的==Environment==（包括配置要使用的**PropertySource**以及**Profile**）。

4） 遍历调用所有SpringApplicationRunListener的environmentPrepared()的方法，告诉他们：“当前SpringBoot应用使用的Environment准备好了咯！”。

5） 如果SpringApplication的showBanner属性被设置为true，则打印banner。

6） 根据用户是否明确设置了applicationContextClass类型以及初始化阶段的推断结果，决定该为当前SpringBoot应用创建什么类型的ApplicationContext并创建完成，然后根据条件决定是否添加ShutdownHook，决定是否使用自定义的BeanNameGenerator，决定是否使用自定义的ResourceLoader，当然，最重要的，将之前准备好的Environment设置给创建好的ApplicationContext使用。

7） ApplicationContext创建好之后，SpringApplication会再次借助SpringFactoriesLoader，查找并加载classpath中所有可用的ApplicationContextInitializer，然后遍历调用这些ApplicationContextInitializer的initialize（applicationContext）方法来对已经创建好的ApplicationContext进行进一步的处理。

8） 遍历调用所有SpringApplicationRunListener的contextPrepared()方法。

9） 最核心的一步，将之前通过@EnableAutoConfiguration获取的所有配置以及其他形式的IoC容器配置加载到已经准备完毕的ApplicationContext。

10） 遍历调用所有SpringApplicationRunListener的contextLoaded()方法。

11） 调用ApplicationContext的refresh()方法，完成IoC容器可用的最后一道工序。

12） 查找当前ApplicationContext中是否注册有CommandLineRunner，如果有，则遍历执行它们。

13） 正常情况下，遍历执行SpringApplicationRunListener的finished()方法、（如果整个过程出现异常，则依然调用所有SpringApplicationRunListener的finished()方法，只不过这种情况下会将异常信息一并传入处理）

去除事件通知点后，整个流程如下：

![249993-20171212151321051-993064506](assets/249993-20171212151321051-993064506.jpg)

本文以调试一个实际的SpringBoot启动程序为例，参考流程中主要类类图，来分析其启动逻辑和自动化配置原理。

![1112095-20181115161924186-928667393](assets/1112095-20181115161924186-928667393.png)

##**总览：**    

上图为SpringBoot启动结构图，我们发现启动流程主要分为三个部分，第一部分==进行SpringApplication的初始化模块==，配置一些基本的环境变量、资源、构造器、监听器，第二部分实现了应用==具体的启动方案==，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块，第三部分是==自动化配置模块==，该模块作为springboot自动配置核心，在后面的分析中会详细讨论。在下面的启动程序中我们会串联起结构中的主要功能。

##**启动：**

每个SpringBoot程序都有一个主入口，也就是==main==方法，main里面调用SpringApplication.run()启动整个springboot程序，该方法所在类需要使用@SpringBootApplication注解，以及@ImportResource注解(if need)，@SpringBootApplication包括三个注解，功能如下：

@EnableAutoConfiguration：SpringBoot根据应用所声明的依赖来对Spring框架进行自动配置。

@SpringBootConfiguration(内部为@Configuration)：被标注的类等于在spring的XML配置文件中(applicationContext.xml)，装配所有bean事务，提供了一个spring的上下文环境。

@ComponentScan：==组件扫描==，可自动发现和装配Bean，默认扫描SpringApplication的run方法里的ToutiaoApplication.class所在的包路径下文件，所以最好将该启动类放到根包路径下。

![屏幕快照 2019-04-11 下午10.04.47](assets/屏幕快照 2019-04-11 下午10.04.47.png)

###**SpringBoot启动类**

首先进入run方法

~~~java
    public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
        return (new SpringApplication(sources)).run(args);
    }
~~~

run方法中去创建了一个SpringApplication实例，在该构造方法内，我们可以发现其调用了一个初始化的initialize方法：

~~~java
public SpringApplication(Object... sources) {
        this.bannerMode = Mode.CONSOLE;
        this.logStartupInfo = true;
        this.addCommandLineProperties = true;
        this.headless = true;
        this.registerShutdownHook = true;
        this.additionalProfiles = new HashSet();
        this.initialize(sources);
    }
private void initialize(Object[] sources) {
        if (sources != null && sources.length > 0) {
            this.sources.addAll(Arrays.asList(sources));
        }

        this.webEnvironment = this.deduceWebEnvironment();
        this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
        this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
        this.mainApplicationClass = this.deduceMainApplicationClass();
    }
~~~

这里主要是为SpringApplication对象赋一些初值。构造函数执行完毕后，我们回到run方法

~~~java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        FailureAnalyzers analyzers = null;
        this.configureHeadlessProperty();
        //1
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
          //2,3
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            Banner printedBanner = this.printBanner(environment);
          //4
            context = this.createApplicationContext();
            new FailureAnalyzers(context);
          //5
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
          //6
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            listeners.finished(context, (Throwable)null);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, listeners, (FailureAnalyzers)analyzers, var9);
            throw new IllegalStateException(var9);
        }
    }
~~~

该方法中实现了如下几个关键步骤：

1.创建了应用的监听器SpringApplicationRunListeners并开始监听

2.加载SpringBoot配置环境(ConfigurableEnvironment)，如果是通过web容器发布，会加载StandardEnvironment，其最终也是继承了ConfigurableEnvironment，类图如下

![249993-20171213144808707-1335729370](assets/249993-20171213144808707-1335729370.png)

可以看出，**Environment**最终都实现了**PropertyResolver**接口，我们平时通过environment对象获取配置文件中指定Key对应的value方法时，就是调用了propertyResolver接口的getProperty方法

3.配置环境(Environment)加入到监听器对象中(SpringApplicationRunListeners)

~~~java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments) {
        //2
        ConfigurableEnvironment environment = this.getOrCreateEnvironment();
        this.configureEnvironment((ConfigurableEnvironment)environment, applicationArguments.getSourceArgs());
        //3
        listeners.environmentPrepared((ConfigurableEnvironment)environment);
        if (!this.webEnvironment) {
            environment = (new EnvironmentConverter(this.getClassLoader())).convertToStandardEnvironmentIfNecessary((ConfigurableEnvironment)environment);
        }

        return (ConfigurableEnvironment)environment;
    }
private ConfigurableEnvironment getOrCreateEnvironment() {
        if (this.environment != null) {
            return this.environment;
        } else {
          //如果是通过web容器发布，会加载StandardEnvironment
            return (ConfigurableEnvironment)(this.webEnvironment ? new StandardServletEnvironment() : new StandardEnvironment());
        }
    }
~~~



4.创建run方法的返回对象：ConfigurableApplicationContext(应用配置上下文)，我们可以看一下创建方法：

~~~java
protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            try {
                contextClass = Class.forName(this.webEnvironment ? "org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext" : "org.springframework.context.annotation.AnnotationConfigApplicationContext");
            } catch (ClassNotFoundException var3) {
                throw new IllegalStateException("Unable create a default ApplicationContext, please specify an ApplicationContextClass", var3);
            }
        }

        return (ConfigurableApplicationContext)BeanUtils.instantiate(contextClass);
    }
~~~

方法会**先获取显式设置的应用上下文(applicationContextClass)**，如果不存在，再加载默认的环境配置（通过是否是web environment判断），默认选择**AnnotationConfigApplicationContext**注解上下文（通过扫描所有注解类来加载bean），最后**通过BeanUtils实例化上下文对象**，并返回。

ConfigurableApplicationContext类图如下：

![249993-20171213154313488-1411301156](assets/249993-20171213154313488-1411301156.png)

主要看其继承的两个方向：

LifeCycle：生命周期类，定义了start启动、stop结束、isRunning是否运行中等生命周期空值方法

**ApplicationContext**：应用上下文类，其主要继承了beanFactory(bean的工厂类)

5.回到run方法内，prepareContext方法将listeners、environment、applicationArguments、banner等重要组件与上下文对象关联

6.接下来的refreshContext(context)方法(初始化方法如下)将是实现spring-boot-starter-*(mybatis、redis等)自动化配置的关键，包==括spring.factories的加载，bean的实例化等核心工作==。

最后底层调用的是AbstractApplicationContext类中的refresh()：

~~~java
    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
~~~

配置结束后，Springboot做了一些基本的收尾工作，返回了应用环境上下文。回顾整体流程，Springboot的启动，主要创建了配置环境(environment)、事件监听(listeners)、应用上下文(applicationContext)，并基于以上条件，在容器中开始实例化我们需要的Bean，至此，通过SpringBoot启动的程序已经构造完成，接下来我们来探讨自动化配置是如何实现。

##**自动化配置：**

之前的启动结构图中，我们注意到无论是应用初始化还是具体的执行过程，都调用了SpringBoot自动配置模块

![249993-20171219161544990-1859845219](assets/249993-20171219161544990-1859845219.png)

###SpringBoot自动配置模块

​    该配置模块的主要使用到了==SpringFactoriesLoader==，即**Spring工厂加载器**，该对象提供了loadFactoryNames方法，入参为factoryClass和classLoader，即需要传入上图中的**工厂类名称**和**对应的类加载器**，==方法会根据指定的classLoader，加载该类加器搜索路径下的指定文件，即spring.factories文件==，传入的工厂类为接口，而文件中对应的类则是接口的实现类，或最终作为实现类，所以文件中一般为如下图这种一对多的类名集合，获取到这些实现类的类名后，loadFactoryNames方法返回类名集合，方法调用方得到这些集合后，再通过==反射==获取这些类的类对象、构造方法，**最终生成实例**。

![249993-20171219162725615-751500087](assets/249993-20171219162725615-751500087.png)

工厂接口与其若干实现类接口名称。

下图有助于我们形象理解自动配置流程：

![249993-20171219162844787-1104034109](assets/249993-20171219162844787-1104034109.png)

SpringBoot自动化配置关键组件关系图 

mybatis-spring-boot-starter、spring-boot-starter-web等组件的META-INF文件下均含有spring.factories文件，自动配置模块中，==SpringFactoriesLoader收集到文件中的类全名并返回一个类全名的数组==，返回的类全名通过**反射被实例化**，就形成了具体的工厂实例，工厂实例来生成组件具体需要的bean。

之前我们提到了EnableAutoConfiguration注解，其类图如下：

![249993-20171219163356100-1697141132](assets/249993-20171219163356100-1697141132.png)

可以发现其最终实现了ImportSelector(选择器)和BeanClassLoaderAware(bean类加载器中间件)，重点关注一下AutoConfigurationImportSelector的selectImports方法。

~~~java
public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!this.isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        } else {
            try {
                AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
                AnnotationAttributes attributes = this.getAttributes(annotationMetadata);
              //
                List<String> configurations = this.getCandidateConfigurations(annotationMetadata, attributes);
                configurations = this.removeDuplicates(configurations);
                configurations = this.sort(configurations, autoConfigurationMetadata);
                Set<String> exclusions = this.getExclusions(annotationMetadata, attributes);
                this.checkExcludedClasses(configurations, exclusions);
                configurations.removeAll(exclusions);
                configurations = this.filter(configurations, autoConfigurationMetadata);
                this.fireAutoConfigurationImportEvents(configurations, exclusions);
                return (String[])configurations.toArray(new String[configurations.size()]);
            } catch (IOException var6) {
                throw new IllegalStateException(var6);
            }
        }
    }
~~~

该方法在springboot启动流程——bean实例化前被执行，**返回要实例化的类信息列表**。我们知道，如果获取到类信息，spring自然可以通过类加载器将类加载到jvm中，现在我们已经通过springboot的starter依赖方式依赖了我们需要的组件，那么这些组建的类信息在select方法中也是可以被获取到的，不要急我们继续向下分析。

~~~java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
        List<String> configurations = SpringFactoriesLoader.loadFactoryNames(this.getSpringFactoriesLoaderFactoryClass(), this.getBeanClassLoader());
        Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you are using a custom packaging, make sure that file is correct.");
        return configurations;
    }
~~~

该方法中的**getCandidateConfigurations**方法，其返回一个自动配置类的类名列表，方法调用了loadFactoryNames方法，查看该方法

~~~java
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();

        try {
            Enumeration<URL> urls = classLoader != null ? classLoader.getResources("META-INF/spring.factories") : ClassLoader.getSystemResources("META-INF/spring.factories");
            ArrayList result = new ArrayList();

      	      while(urls.hasMoreElements()) {
                URL url = (URL)urls.nextElement();
                Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
                String propertyValue = properties.getProperty(factoryClassName);
                String[] var8 = StringUtils.commaDelimitedListToStringArray(propertyValue);
                int var9 = var8.length;

                for(int var10 = 0; var10 < var9; ++var10) {
                    String factoryName = var8[var10];
                    result.add(factoryName.trim());
                }
            }

            return result;
        } catch (IOException var12) {
            throw new IllegalArgumentException("Unable to load factories from location [META-INF/spring.factories]", var12);
        }
    }
~~~

![249993-20171219164224912-643901744-2](assets/249993-20171219164224912-643901744-2.png)

在上面的代码可以看到自动配置器会根据传入的factoryClass.getName()到项目系统路径下所有的spring.factories文件中找到相应的key，从而加载里面的类。我们就选取这个mybatis-spring-boot-autoconfigure下的spring.factories文件

![249993-20171219164557240-1466961312](assets/249993-20171219164557240-1466961312.png)

进入org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration中，主要看一下类头：

![249993-20171219164818162-646159475](assets/249993-20171219164818162-646159475.png)

发现Spring的@Configuration，俨然是一个通过注解标注的springBean，继续向下看，

@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class})这个注解的意思是：当存在SqlSessionFactory.class, SqlSessionFactoryBean.class这两个类时才解析MybatisAutoConfiguration配置类，否则不解析这一个配置类，make sence，我们需要mybatis为我们返回会话对象，就必须有会话工厂相关类。

@CondtionalOnBean(DataSource.class)：只有处理已经被声明为bean的dataSource。

@ConditionalOnMissingBean(MapperFactoryBean.class)这个注解的意思是如果容器中不存在name指定的bean则创建bean注入，否则不执行

以上配置可以保证**sqlSessionFactory**、**sqlSessionTemplate**、**dataSource**等mybatis所需的组件均可被自动配置，@Configuration注解已经提供了Spring的上下文环境，所以以上组件的配置方式与Spring启动时通过mybatis.xml文件进行配置起到一个效果。通过分析我们可以发现，只要一个基于SpringBoot项目的类路径下存在SqlSessionFactory.class, SqlSessionFactoryBean.class，并且容器中已经注册了dataSourceBean，就可以触发自动化配置，意思说我们只要在maven的项目中加入了mybatis所需要的若干依赖，就可以触发自动配置，但引入mybatis原生依赖的话，每集成一个功能都要去修改其自动化配置类，那就得不到开箱即用的效果了。所以==Springboot为我们提供了统一的starter可以直接配置好相关的类，触发自动配置所需的依赖(mybatis)==如下：

~~~xml
<!--加载mybatis整合springboot-->
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>1.1.1</version>
		</dependency>
~~~

这里是截取的mybatis-spring-boot-starter的源码中pom.xml文件中所有依赖：

~~~xml
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-autoconfigure</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
    </dependency>
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis-spring</artifactId>
    </dependency>
  </dependencies>
~~~

 因为maven依赖的传递性，我们只要依赖starter就可以依赖到所有需要自动配置的类，实现开箱即用的功能。也体现出Springboot简化了Spring框架带来的大量XML配置以及复杂的依赖管理，让开发人员可以更加关注业务逻辑的开发。