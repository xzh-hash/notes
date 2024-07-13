## SpringBoot

### SpringApplication

#### lazyInit

延迟初始化可以通过编程方式启用，使用SpringApplicationBuilder上的lazyInitialization方法或SpringApplication上的setLazyInitialization方法。或者，也可以使用spring.main来启用它。延迟初始化属性，示例如下:

```yml
spring:
  main:
    lazy-initialization: true
```

应用创建的时候使用的启动方式

```
public static void main(String[] args) {
    SpringApplication.run(UserApplication.class,args);
}
```

如果要使用setLazyInitialization（）方法，可以先创建SpringApplication对象，然后使用对象提供的方法来设置启动相关配置，举个栗子，懒加载启动：

```
public static void main(String[] args) {
        SpringApplication application = new SpringApplication(UserApplication.class);
        application.setLazyInitialization(true);
        application.run();
}
```

使用SpringApplicationBuilder启动

```
public static void main(String[] args) {
        SpringApplicationBuilder springApplicationBuilder = new SpringApplicationBuilder(UserApplication.class);
        springApplicationBuilder.lazyInitialization(true);
        springApplicationBuilder.run(args);
}
```

SpringApplicationBuilder相关，提供了设置配置文件路径的方法。

https://blog.csdn.net/qq_51778285/article/details/126176378



@lazy 可以作用在bean，方法，字段等上面，使一些特定的bean懒加载。

[(139条消息) spring @lazy注解的使用_Java架构狮的博客-CSDN博客_lazy spring](https://blog.csdn.net/m0_69860228/article/details/124714792) 



### springboot的自动装配原理

springboot的自动装配主要是由<artifactId>spring-boot-autoconfigure</artifactId>  这个jar包实现，这份jar包集成在<artifactId>spring-boot-starter-web</artifactId>里面。

springboot的自动装配原理在于他的启动类上的@SpringBootApplication注解，这是一个复合注解。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}
```

@EnableAutoConfiguration 是关键的自动装配注解，这个注解里面复合了@Import(AutoConfigurationImportSelector.class) 这个注解。

```
@Import注解提供了三种用法

1、@Import一个普通类 spring会将该类加载到spring容器中

2、@Import一个类，该类实现了ImportBeanDefinitionRegistrar接口，在重写的registerBeanDefinitions方法里面，能拿到BeanDefinitionRegistry bd的注册器，能手工往beanDefinitionMap中注册 beanDefinition

3、@Import一个类 该类实现了ImportSelector 重写selectImports方法该方法返回了String[]数组的对象，数组里面的类都会注入到spring容器当中
原文链接：https://blog.csdn.net/weixin_45453628/article/details/124234317
```

AutoConfigurationImportSelector.class 这个类应用了@Import注解的第三种用法，其实现了DeferredImportSelector接口。使 String[] selectImports(AnnotationMetadata annotationMetadata) 方法执行。

selectImports方法会在classpath路径下，搜索所有的spring.factories文件（不仅仅是扫描spring-boot-autoconfigure 的jar包，而是classpath下）。

$\color{red} {第三方jar包使用自动装配功能} $

spring-boot-autoconfigre 的spring.facroties 文件中并没有包含所有的第三方jar包，实际上也不可能去包含。

所以第三方的jar包只要集成spring-boot-autoconfigre 即可使用自动装配，比如<artifactId>mybatis-plus-boot-starter</artifactId>中就集成了springboot的自动配置，mybatis有一个自己的spring.factories文件，也会被扫描进去进行自动装配。

$\color{red}{如何过滤未使用的jar包？}$

在通过srping.facroties文件获取了所有的configration 配置类后，这些配置类上会有一个@ConditionalOnClass({LanguageDriver.class}) 条件注解，如果相应的条件类不存在，就说明没有被引用，这样就不会被注入了。

### Bean的创建过程

1. springboot refresh()方法使用反射，invoke 类的无参构造
2. 创建对象
3. 依赖注入
4. 初始化前（@PostConstruct）
5. 初始化（实现InitializingBean）
6. 初始化后（AOP）
7. 代理对象
8. Bean对象放入Map

### Beanfactory

增强器BeanFactoryPostProcessor

###Aware

Aware系列接口，主要用于辅助Spring bean访问Spring容器 。

实现了Aware系列接口的bean可以访问Spring容器 



