# SpringBoot特点

## 依赖管理

- 父项目做依赖管理

  ```xml
  <!--项目依赖这个parent-->
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-parent</artifactId>
      <version>2.3.2.RELEASE</version>
      <relativePath/> 
  </parent>
  <!--starter-parent依赖这个spring-boot-dependencies-->
  <!-- 这个dependencies几乎包含了所有开发需要的包 -->
  <parent>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>2.3.2.RELEASE</version>
  </parent>
  ```

- 开发导入starter场景启动器

  官方提供的starter启动器名字都是这种 spring-boot-starter-*

  可以自定义starter场景启动器：*-spring-boot-starter

  https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-starter

- 无需关注版本号，自动版本仲裁

- 可以修改版本号

  ```xml
  <properties>
  	<lombok.version>1.18.12</lombok.version>
  </properties>
  ```

## 自动配置

- 自动配置SpringMVC

  - 引入SpringMVC全套组件
  - 自动配置好SpringMVC常用的组件

- 自动配好Web常见功能

- 默认的包结构

  - 会自动扫描启动类所在的包
  - 想要改变包路径：`@ComponentScan（basePackages="com.melon"）`

- 各种配置的默认值

  默认配置都最终映射到某一个Properties类上

- 按需加载所有自动配置项

  引入了哪些场景，该场景的自动配置才会开启

- 禁用部分自动配置类

  ```java
  @SpringBootApplication(exclude={DataSourceAutoConfiguration.class})
  public class MyApplication {
  }
  ```

# 容器功能

## 注解

### @Configuration

```java
public @interface Configuration {
    @AliasFor(annotation = Component.class)
	String value() default "";
    // 从配置类中取到的Bean是否从容器中获取
    // true-从容器中获取，且取到的是单例
    boolean proxyBeanMethods() default true;
}
// 示例
@Configuration(proxyBeanMethods = false)
public class MyConfig {
}
```

标识一个类是配置类，并将类加载到容器中

### @Import

```java
public @interface Import {
	Class<?>[] value();
}
// 示例
@Import(Apple.class)
public class MyConfig {
}
```

将一个类注入到容器中

以下注解都可以用来自动装配类

### @Bean

### @Autowired

### @Component

### @Service

### @Repository

### @Controller

### @Conditional：条件装配

![](./image/SpringBoot-按需加载注释.png)

# 自动装配

xxxAutoConfiguration是自动配置类

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(KafkaTemplate.class)
// 开启自动属性配置
@EnableConfigurationProperties(KafkaProperties.class)
@Import({ KafkaAnnotationDrivenConfiguration.class, KafkaStreamsAnnotationDrivenConfiguration.class })
public class KafkaAutoConfiguration {}

// 属性配置类，属性前缀prefix = spring.kafka
@ConfigurationProperties(prefix = "spring.kafka")
public class KafkaProperties {
```

- @EnableConfigurationProperties

  ```java
  @Import(EnableConfigurationPropertiesRegistrar.class)
  public @interface EnableConfigurationProperties {
      Class<?>[] value() default {};
  }
  ```

- @ConfigurationProperties

  ```java
  public @interface ConfigurationProperties {
      @AliasFor("prefix")
  	String value() default "";
      @AliasFor("value")
  	String prefix() default "";
      
  }
  ```

- @ConfigurationPropertiesScan 配置扫描路径

  ```java
  @Import(ConfigurationPropertiesScanRegistrar.class)
  @EnableConfigurationProperties
  public @interface ConfigurationPropertiesScan {
      @AliasFor("basePackages")
  	String[] value() default {};
      @AliasFor("value")
  	String[] basePackages() default {};
  }
  ```

- 使用 @Validated 对 @ConfigurationProperties 属性校验

  ```java
  @ConfigurationProperties(prefix="acme")
  @Validated
  public class AcmeProperties {
      // 非null
      @NotNull
      private InetAddress remoteAddress;
      @NotEmpty
      public String username;
      @Valid
      private Security security;
  }
  ```

# 启动类

主main函数，使用 @SpringBootApplication 注解标识

```java
@SpringBootApplication
// 等同于
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan
```

# 热部署

引入热部署包，每次修改后 Ctrl + F9 编译

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <optional>true</optional>
</dependency>
```



# 配置文件

默认配置文件名：application.properties

key=value的形式

或者

application.yml 格式



# 自定义Banner

classpath 路径下添加一个图片`banner.gif`, `banner.jpg`, or `banner.png`

或者指定图片路径：spring.banner.image.location



# Application 事件 和 监听

# 自定义Listener

- 自定义Listener 实现 `ApplicationListener<ApplicationEvent>`

  ```java
  public class MyApplicationListener implements ApplicationListener<ApplicationEvent> {
      @Override
      public void onApplicationEvent(ApplicationEvent event) {
          System.out.println("MyApplicationListener onApplicationEvent：" + event.toString());
      }
  }
  ```

- 创建SpringApplicationListener

  ```java
  public class MySpringApplicationRunListener implements SpringApplicationRunListener {
  
      private final SpringApplication application;
  
      private final String[] args;
  
      public MySpringApplicationRunListener(SpringApplication application, String[] args) {
          this.application = application;
          this.args = args;
      }
  
      @Override
      public void starting() {
          System.out.println("MySpringApplicationRunListener....starting....");
      }
  
      @Override
      public void environmentPrepared(ConfigurableEnvironment environment) {
          System.out.println("MySpringApplicationRunListener....environmentPrepared....");
      }
  
      @Override
      public void contextPrepared(ConfigurableApplicationContext context) {
          System.out.println("MySpringApplicationRunListener....contextPrepared....");
      }
  
      @Override
      public void contextLoaded(ConfigurableApplicationContext context) {
          System.out.println("MySpringApplicationRunListener....contextLoaded....");
      }
  
      @Override
      public void started(ConfigurableApplicationContext context) {
          System.out.println("MySpringApplicationRunListener....started....");
      }
  
      @Override
      public void running(ConfigurableApplicationContext context) {
          System.out.println("MySpringApplicationRunListener....running....");
      }
  
      @Override
      public void failed(ConfigurableApplicationContext context, Throwable exception) {
          System.out.println("MySpringApplicationRunListener....failed....");
      }
  }
  ```

- 添加一个/META-INF/spring.factories 文件，配置

  ```properties
  org.springframework.context.ApplicationListener=\
   com.melon.springboot.listener.MyApplicationListener
   
  org.springframework.boot.SpringApplicationRunListener=\
  com.melon.springboot.listener.MySpringApplicationRunListener
  ```

# 自定义Runner

ApplicationRunner和CommandLineRunner

容器启动后的一次性任务

- 自定义MyApplicationRunner实现ApplicationRunner，并使用@Component注入容器

  ```java
  @Component
  public class MyApplicationRunner implements ApplicationRunner {
      @Override
      public void run(ApplicationArguments args) throws Exception {
          System.out.println("MyApplicationRunner run");
      }
  }
  ```

- 自定义MyCommandLineRunner实现CommandLineRunner，并使用@Component注入容器

  ```java
  @Component
  public class MyCommandLineRunner implements CommandLineRunner {
      @Override
      public void run(String... args) throws Exception {
          System.out.println("MyCommandLineRunner run");
      }
  }
  ```

# 运行环境Profile

选择哪套环境

```java
@Profile("production")
public class ProductionConfiguration {
}
```

配置文件配置环境类型：

```properties
spring.profiles.active=dev,production
```



