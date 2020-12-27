# 1.什么是 Spring 框架?

Spring 是一种轻量级开发框架，旨在提高开发人员的开发效率以及系统的可维护性。Spring 官网：https://spring.io/。

我们一般说 Spring 框架指的都是 Spring Framework，它是很多模块的集合，使用这些模块可以很方便地协助我们进行开发。这些模块是：核心容器、数据访问/集成,、Web、AOP（面向切面编程）、工具、消息和测试模块。比如：Core Container 中的 Core 组件是Spring 所有组件的核心，Beans 组件和 Context 组件是实现IOC和依赖注入的基础，AOP组件用来实现面向切面编程。

Spring 官网列出的 Spring 的 6 个特征:

- **核心技术** ：依赖注入(DI)，AOP，事件(events)，资源，i18n，验证，数据绑定，类型转换，SpEL。
- **测试** ：模拟对象，TestContext框架，Spring MVC 测试，WebTestClient。
- **数据访问** ：事务，DAO支持，JDBC，ORM，编组XML。
- **Web支持** : Spring MVC和Spring WebFlux Web框架。
- **集成** ：远程处理，JMS，JCA，JMX，电子邮件，任务，调度，缓存。
- **语言** ：Kotlin，Groovy，动态语言。

# 2.列举一些重要的Spring模块？

下图对应的是 Spring4.x 版本。目前最新的5.x版本中 Web 模块的 Portlet 组件已经被废弃掉，同时增加了用于**异步响应式处理的 WebFlux 组件**。

[![Spring主要模块](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602155434)](https://camo.githubusercontent.com/d7846803a95e57f57a94197168714bfc58db6432/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f537072696e672545342542382542422545382541362538312545362541382541312545352539442539372e706e67)

- **Spring Core：** 基础,可以说 Spring 其他所有的功能都需要依赖于该类库。主要提供 IoC 依赖注入功能。
- **Spring Aspects** ： 该模块为与AspectJ的集成提供支持。
- **Spring AOP** ：提供了面向切面的编程实现。
- **Spring JDBC** : Java数据库连接。
- **Spring JMS** ：Java消息服务。
- **Spring ORM** : 用于支持Hibernate等ORM工具。
- **Spring Web** : 为创建Web应用程序提供支持。
- **Spring Test** : 提供了对 JUnit 和 TestNG 测试的支持。

# 3.常用注解

## 1. `@SpringBootApplication`

这个注解是 Spring Boot 项目的基石，创建 SpringBoot 项目之后会默认在主类加上。

我们可以把 `@SpringBootApplication`看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 注解的集合。

根据 SpringBoot 官网，这三个注解的作用分别是：

- `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
- `@ComponentScan`： 扫描被`@Component` (`@Service`,`@Controller`)注解的 bean，注解默认会扫描该类所在的包下所有的类。
- `@Configuration`：允许在 Spring 上下文中注册额外的 bean 或导入其他配置类

```java
package org.springframework.boot.autoconfigure;
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
   ......
}

package org.springframework.boot;
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

}
```

## 2. Spring Bean 相关

-  `@Autowired`

  自动导入对象到类中，被注入进的类同样要被 Spring 容器管理比如：Service 类注入到 Controller 类中。

- `Component`,`@Repository`,`@Service`, `@Controller`

  我们一般使用 `@Autowired` 注解让 Spring 容器帮我们自动装配 bean。要想把类标识成可用于 `@Autowired` 注解自动装配的 bean 的类,可以采用以下注解实现：

  - `@Component` ：通用的注解，可标注任意类为 `Spring` 组件。如果一个 Bean 不知道属于哪个层，可以使用`@Component` 注解标注。
  - `@Repository` : 对应持久层即 Dao 层，主要用于数据库相关操作。
  - `@Service` : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao 层。
  - `@Controller` : 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。

- `@RestController`

  `@RestController`注解是`@Controller和`@`ResponseBody`的合集,表示这是个控制器 bean,并且是将函数的返回值直 接填入 HTTP 响应体中,是 REST 风格的控制器。

  现在都是前后端分离，说实话我已经很久没有用过`@Controller`。如果你的项目太老了的话，就当我没说。*

  单独使用 `@Controller` 不加 `@ResponseBody`的话一般使用在要返回一个视图的情况，这种情况属于比较传统的 Spring MVC 的应用，对应于前后端不分离的情况。`@Controller` +`@ResponseBody` 返回 JSON 或 XML 形式数据

  关于`@RestController` 和 `@Controller`的对比，请看这篇文章：[@RestController vs @Controller](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485544&idx=1&sn=3cc95b88979e28fe3bfe539eb421c6d8&chksm=cea247a3f9d5ceb5e324ff4b8697adc3e828ecf71a3468445e70221cce768d1e722085359907&token=1725092312&lang=zh_CN#rd)。

- `@Bean`

  `@Bean` 注解**作用于方法**，通常是我们在标有该注解的方法中定义产生这个 bean,`@Bean`告诉了Spring这是某个类的示例，当我需要用它的时候还给我。

  `@Bean` 注解比 `Component` 注解的自定义性更强，而且很多地方我们只能通过 `@Bean` 注解来注册bean。比如当我们引用第三方库中的类需要装配到 `Spring`容器时，则只能通过 `@Bean`来实现。

- `@Scope`

  声明 Spring Bean 的作用域，使用方法:

  ```
  @Bean
  @Scope("singleton")
  public Person personSingleton() {
      return new Person();
  }
  ```

  **四种常见的 Spring Bean 的作用域：**

  - **singleton** : 唯一 bean 实例，Spring 中的 bean 默认都是单例的。
  - **prototype** : 每次请求都会创建一个新的 bean 实例。
  - **request** : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP request 内有效。
  - **session** : 每一次 HTTP 请求都会产生一个新的 bean，该 bean 仅在当前 HTTP session 内有效。

- `Configuration`

  一般用来声明配置类，可以使用 `@Component`注解替代，不过使用`Configuration`注解声明配置类更加语义化。

- `@PostConstruct`

  用来修饰方法，标记在项目启动的时候执行这个方法,一般用来执行某些初始化操作比如全局配置。`PostConstruct` **注解的方法会在构造函数之后执行,Servlet 的`init()`方法之前执行**。

- `@PreDestroy`

   当 bean 被 Web 容器的时候被调用，一般用来释放 bean 所持有的资源。`@PreDestroy` 注解的方法会在Servlet 的`destroy()`方法之前执行。

- 单例bean的**线程安全问题**：

  单例 bean 存在线程问题，主要是因为当多个线程操作同一个对象的时候，对这个对象的非静态成员变量的写操作会存在线程安全问题。

  常见的有两种解决办法：

  1. 在Bean对象中尽量避免定义可变的成员变量（不太现实）。
  2. 在类中定义一个ThreadLocal成员变量，将需要的可变成员变量保存在 ThreadLocal 中（推荐的一种方式）。

## 3. 处理常见的 HTTP 请求类型

**5 种常见的请求类型:**

- **GET** ：请求从服务器获取特定资源。举个例子：`GET /users`（获取所有学生）

  ```java
  @GetMapping("users")` 等价于`@RequestMapping(value="/users",method=RequestMethod.GET)
  ```

- **POST** ：在服务器上创建一个新的资源。举个例子：`POST /users`（创建学生）

  ```java
  @PostMapping("users")` 等价于`@RequestMapping(value="/users",method=RequestMethod.POST)
  ```

- **PUT** ：更新服务器上的资源（客户端提供更新后的整个资源）。举个例子：`PUT /users/12`（更新编号为 12 的学生）

  ```java
  @PutMapping("/users/{userId}")` 等价于`@RequestMapping(value="/users/{userId}",method=RequestMethod.PUT)
  ```

- **DELETE** ：从服务器删除特定的资源。举个例子：`DELETE /users/12`（删除编号为 12 的学生）

  ```java
  @DeleteMapping("/users/{userId}")`等价于`@RequestMapping(value="/users/{userId}",method=RequestMethod.DELETE)
  ```

- **PATCH** ：更新服务器上的资源（客户端提供更改的属性，可以看做作是部分更新），使用的比较少，这里就不举例子了。

## 4. 前后端传值

- `@PathVariable` 和 `@RequestParam`

  `@PathVariable`用于获取路径参数，`@RequestParam`用于获取查询参数。

  举个简单的例子：

  ```java
  @GetMapping("/klasses/{klassId}/teachers")
  public List<Teacher> getKlassRelatedTeachers(
           @PathVariable("klassId") Long klassId,
           @RequestParam(value = "type", required = false) String type ) {
  ...
  }
  ```

  如果我们请求的 url 是：`/klasses/{123456}/teachers?type=web`

  那么我们服务获取到的数据就是：`klassId=123456,type=web`。

- `@RequestBody`

  用于读取 Request 请求（可能是 POST,PUT,DELETE,GET 请求）的 body 部分并且**Content-Type 为 application/json** 格式的数据，接收到数据之后会自动将数据绑定到 Java 对象上去。系统会使用`HttpMessageConverter`或者自定义的`HttpMessageConverter`将请求的 body 中的 json 字符串转换为 java 对象。

  需要注意的是：**一个请求方法只可以有一个`@RequestBody`，但是可以有多个`@RequestParam`和`@PathVariable`**。 如果你的方法必须要用两个 `@RequestBody`来接受数据的话，大概率是你的数据库设计或者系统设计出问题了！

## 5. 读取配置信息

**很多时候我们需要将一些常用的配置信息比如阿里云 oss、发送短信、微信认证的相关配置信息等等放到配置文件中。**

**下面我们来看一下 Spring 为我们提供了哪些方式帮助我们从配置文件中读取这些配置信息。**

我们的数据源`application.yml`内容如下：：

```yaml
wuhan2020: 2020年初武汉爆发了新型冠状病毒，疫情严重，但是，我相信一切都会过去！武汉加油！中国加油！

my-profile:
  name: Guide哥
  email: koushuangbwcx@163.com

library:
  location: 湖北武汉加油中国加油
  books:
    - name: 天才基本法
      description: 二十二岁的林朝夕在父亲确诊阿尔茨海默病这天，得知自己暗恋多年的校园男神裴之即将出国深造的消息——对方考取的学校，恰是父亲当年为她放弃的那所。
    - name: 时间的秩序
      description: 为什么我们记得过去，而非未来？时间“流逝”意味着什么？是我们存在于时间之内，还是时间存在于我们之中？卡洛·罗韦利用诗意的文字，邀请我们思考这一亘古难题——时间的本质。
    - name: 了不起的我
      description: 如何养成一个新习惯？如何让心智变得更成熟？如何拥有高质量的关系？ 如何走出人生的艰难时刻？
```

- `@value`(常用)

  使用 `@Value("${property}")` 读取比较简单的配置信息：

  ```java
  @Value("${wuhan2020}")
  String wuhan2020;
  ```

- `@ConfigurationProperties`(常用)

  通过`@ConfigurationProperties`读取配置信息并与 bean 绑定。

  ```java
  @Component
  @ConfigurationProperties(prefix = "library")
  class LibraryProperties {
      @NotEmpty
      private String location;
      private List<Book> books;
  
      @Setter
      @Getter
      @ToString
      static class Book {
          String name;
          String description;
      }
    省略getter/setter
    ......
  }
  ```

- `PropertySource`（不常用）

  `@PropertySource`读取指定 properties 文件

## 6. 参数校验

**数据的校验的重要性就不用说了，即使在前端对数据进行校验的情况下，我们还是要对传入后端的数据再进行一遍校验，避免用户绕过浏览器直接通过一些 HTTP 工具直接向后端请求一些违法数据。**

**JSR(Java Specification Requests）** 是一套 JavaBean 参数校验的标准，它定义了很多常用的校验注解，我们可以直接将这些注解加在我们 JavaBean 的属性上面，这样就可以在需要校验的时候进行校验了，非常方便！

校验的时候我们实际用的是 **Hibernate Validator** 框架。Hibernate Validator 是 Hibernate 团队最初的数据校验框架，Hibernate Validator 4.x 是 Bean Validation 1.0（JSR 303）的参考实现，Hibernate Validator 5.x 是 Bean Validation 1.1（JSR 349）的参考实现，目前最新版的 Hibernate Validator 6.x 是 Bean Validation 2.0（JSR 380）的参考实现。

SpringBoot 项目的 spring-boot-starter-web 依赖中已经有 hibernate-validator 包，不需要引用相关依赖。

需要注意的是： **所有的注解，推荐使用 JSR 注解，即`javax.validation.constraints`，而不是`org.hibernate.validator.constraints`**

- 一些常用的字段验证的注解
  - `@NotEmpty` 被注释的字符串的不能为 null 也不能为空
  - `@NotBlank` 被注释的字符串非 null，并且必须包含一个非空白字符
  - `@Null` 被注释的元素必须为 null
  - `@NotNull` 被注释的元素必须不为 null
  - `@AssertTrue` 被注释的元素必须为 true
  - `@AssertFalse` 被注释的元素必须为 false
  - `@Pattern(regex=,flag=)`被注释的元素必须符合指定的正则表达式
  - `@Email` 被注释的元素必须是 Email 格式。
  - `@Min(value)`被注释的元素必须是一个数字，其值必须大于等于指定的最小值
  - `@Max(value)`被注释的元素必须是一个数字，其值必须小于等于指定的最大值
  - `@DecimalMin(value)`被注释的元素必须是一个数字，其值必须大于等于指定的最小值
  - `@DecimalMax(value)` 被注释的元素必须是一个数字，其值必须小于等于指定的最大值
  - `@Size(max=, min=)`被注释的元素的大小必须在指定的范围内
  - `@Digits (integer, fraction)`被注释的元素必须是一个数字，其值必须在可接受的范围内
  - `@Past`被注释的元素必须是一个过去的日期
  - `@Future` 被注释的元素必须是一个将来的日期

- 验证请求体(RequestBody)

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class Person {
  
      @NotNull(message = "classId 不能为空")
      private String classId;
  
      @Size(max = 33)
      @NotNull(message = "name 不能为空")
      private String name;
  
      @Pattern(regexp = "((^Man$|^Woman$|^UGM$))", message = "sex 值不在可选范围")
      @NotNull(message = "sex 不能为空")
      private String sex;
  
      @Email(message = "email 格式不正确")
      @NotNull(message = "email 不能为空")
      private String email;
  
  }
  ```

  我们在需要验证的参数上加上了`@Valid`注解，如果验证失败，它将抛出`MethodArgumentNotValidException`。

  ```java
  @RestController
  @RequestMapping("/api")
  public class PersonController {
  
      @PostMapping("/person")
      public ResponseEntity<Person> getPerson(@RequestBody @Valid Person person) {
          return ResponseEntity.ok().body(person);
      }
  }
  ```

- 验证请求参数(Path Variables 和 Request Parameters)

  **一定一定不要忘记在类上加上 `Validated` 注解了，这个参数可以告诉 Spring 去校验方法参数。**

  ```java
  @RestController
  @RequestMapping("/api")
  @Validated
  public class PersonController {
  
      @GetMapping("/person/{id}")
      public ResponseEntity<Integer> getPersonByID(@Valid @PathVariable("id") @Max(value = 5,message = "超过 id 的范围了") Integer id) {
          return ResponseEntity.ok().body(id);
      }
  }
  ```

  更多关于如何在 Spring 项目中进行参数校验的内容，请看《[如何在 Spring/Spring Boot 中做参数校验？你需要了解的都在这里！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485783&idx=1&sn=a407f3b75efa17c643407daa7fb2acd6&chksm=cea2469cf9d5cf8afbcd0a8a1c9cc4294d6805b8e01bee6f76bb2884c5bc15478e91459def49&token=292197051&lang=zh_CN#rd)》这篇文章。

## 7. 全局处理 Controller 层异常

介绍一下我们 Spring 项目必备的全局处理 Controller 层异常。

**相关注解：**

1. `@ControllerAdvice` :注解定义全局异常处理类
2. `@ExceptionHandler` :注解声明异常处理方法

如何使用呢？拿我们在第 5 节参数校验这块来举例子。如果方法参数不对的话就会抛出`MethodArgumentNotValidException`，我们来处理这个异常。

```
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {

    /**
     * 请求参数异常处理
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<?> handleMethodArgumentNotValidException(MethodArgumentNotValidException ex, HttpServletRequest request) {
       ......
    }
}
```

更多关于 Spring Boot 异常处理的内容，请看我的这两篇文章：

1. [SpringBoot 处理异常的几种常见姿势](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485568&idx=2&sn=c5ba880fd0c5d82e39531fa42cb036ac&chksm=cea2474bf9d5ce5dcbc6a5f6580198fdce4bc92ef577579183a729cb5d1430e4994720d59b34&token=2133161636&lang=zh_CN#rd)
2. [使用枚举简单封装一个优雅的 Spring Boot 全局异常处理！](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486379&idx=2&sn=48c29ae65b3ed874749f0803f0e4d90e&chksm=cea24460f9d5cd769ed53ad7e17c97a7963a89f5350e370be633db0ae8d783c3a3dbd58c70f8&token=1054498516&lang=zh_CN#rd)

## 8.事务 `@Transactional`-重点

在要开启事务的方法上使用`@Transactional`注解即可!

```java
@Transactional(rollbackFor = Exception.class)
public void save() {
  ......
}
```

我们知道 Exception 分为运行时异常 RuntimeException 和非运行时异常。在`@Transactional`注解中如果不配置`rollbackFor`属性,那么事物只会在遇到`RuntimeException`的时候才会回滚,加上`rollbackFor=Exception.class`,可以让事物在遇到非运行时异常时也回滚。

`@Transactional` 注解一般用在可以作用在`类`或者`方法`上。

- **作用于类**：当把`@Transactional 注解放在类上时，表示所有该类的`public 方法都配置相同的事务属性信息。
- **作用于方法**：当类配置了`@Transactional`，方法也配置了`@Transactional`，方法的事务会覆盖类的事务配置信息。

更多关于关于 Spring 事务的内容请查看：

1. [一口气说出 6 种 @Transactional 注解失效场景](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247486483&idx=2&sn=77be488e206186803531ea5d7164ec53&chksm=cea243d8f9d5cacecaa5c5daae4cde4c697b9b5b21f96dfc6cce428cfcb62b88b3970c26b9c2&token=816772476&lang=zh_CN#rd)

#### 8.1 属性

- **propagation**属性-事务传播

  `propagation` 代表事务的传播行为，默认值为 `Propagation.REQUIRED`，其他的属性信息如下：

  - `Propagation.REQUIRED`：如果当前存在事务，则加入该事务，如果当前不存在事务，则创建一个新的事务。**(** 也就是说如果A方法和B方法都添加了注解，在默认传播模式下，A方法内部调用B方法，会把两个方法的事务合并为一个事务 **）**
  - `Propagation.SUPPORTS`：如果当前存在事务，则加入该事务；如果当前不存在事务，则以非事务的方式继续运行。
  - `Propagation.MANDATORY`：如果当前存在事务，则加入该事务；如果当前不存在事务，则抛出异常。
  - `Propagation.REQUIRES_NEW`：重新创建一个新的事务，如果当前存在事务，暂停当前的事务。**(** 当类A中的 a 方法用默认`Propagation.REQUIRED`模式，类B中的 b方法加上采用 `Propagation.REQUIRES_NEW`模式，然后在 a 方法中调用 b方法操作数据库，然而 a方法抛出异常后，b方法并没有进行回滚，因为`Propagation.REQUIRES_NEW`会暂停 a方法的事务 **)**
  - `Propagation.NOT_SUPPORTED`：以非事务的方式运行，如果当前存在事务，暂停当前的事务。
  - `Propagation.NEVER`：以非事务的方式运行，如果当前存在事务，则抛出异常。
  - `Propagation.NESTED` ：和 Propagation.REQUIRED 效果一样。

- **isolation** 属性

  `isolation` ：事务的隔离级别，默认值为 `Isolation.DEFAULT`。

  - **TransactionDefinition.ISOLATION_DEFAULT:** 使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.
  - **TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**
  - **TransactionDefinition.ISOLATION_READ_COMMITTED:** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**
  - **TransactionDefinition.ISOLATION_REPEATABLE_READ:** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**
  - **TransactionDefinition.ISOLATION_SERIALIZABLE:** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

- timeout 属性

  `timeout` ：事务的超时时间，默认值为 -1。如果超过该时间限制但事务还没有完成，则自动回滚事务。

- readOnly 属性

  `readOnly` ：指定事务是否为只读事务，默认值为 false；为了忽略那些不需要事务的方法，比如读取数据，可以设置 read-only 为 true。

- rollbackFor 属性

  `rollbackFor` ：用于指定能够触发事务回滚的异常类型，可以指定多个异常类型。

- noRollbackFor属性

  `noRollbackFor`：抛出指定的异常类型，不回滚事务，也可以指定多个异常类型。

#### 8.2 @Transactional失效场景

1. @Transactional 应用在非 public 修饰的方法上

   之所以会失效是因为在Spring AOP 代理时，如上图所示 `TransactionInterceptor` （事务拦截器）在目标方法执行前后进行拦截，`DynamicAdvisedInterceptor`（CglibAopProxy 的内部类）的 intercept 方法或 `JdkDynamicAopProxy` 的 invoke 方法会间接调用 `AbstractFallbackTransactionAttributeSource`的 `computeTransactionAttribute` 方法，获取Transactional 注解的事务配置信息。

   ```java
   protected TransactionAttribute computeTransactionAttribute(Method method,
       Class<?> targetClass) {
           // Don't allow no-public methods as required.
           if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
           returnnull;
   }
   ```

   此方法会检查目标方法的修饰符是否为 public，不是 public则不会获取`@Transactional` 的属性配置信息。

   **注意：`protected`、`private` 修饰的方法上使用 `@Transactional` 注解，虽然事务无效，但不会有任何报错，这是我们很容犯错的一点。**

2. @Transactional 注解属性 propagation 设置错误

   这种失效是由于配置错误，若是错误的配置以下三种 propagation，事务将不会发生回滚。

   `TransactionDefinition.PROPAGATION_SUPPORTS`：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。

   `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`：以非事务方式运行，如果当前存在事务，则把当前事务挂起。

   `TransactionDefinition.PROPAGATION_NEVER`：以非事务方式运行，如果当前存在事务，则抛出异常。

3. @Transactional  注解属性 rollbackFor 设置错误

   `rollbackFor` 可以指定能够触发事务回滚的异常类型。Spring默认抛出了未检查`unchecked`异常（继承自 `RuntimeException` 的异常）或者 `Error`才回滚事务；其他异常不会触发回滚事务。如果在事务中抛出其他类型的异常，但却期望 Spring 能够回滚事务，就需要指定 **rollbackFor**属性。

4. 同一个类中方法调用，导致@Transactional失效

   其实这还是由于使用`Spring AOP`代理造成的，因为只有当事务方法被当前类以外的代码调用时，才会由`Spring`生成的代理对象来管理。

5. 异常被你的 catch“吃了”导致@Transactional失效

   在业务方法中一般不需要catch异常，如果非要catch一定要抛出`throw new RuntimeException()`，或者注解中指定抛异常类型`@Transactional(rollbackFor=Exception.class)`，否则会导致事务失效，数据commit造成数据不一致，所以有些时候 try catch反倒会画蛇添足。

6. 数据库引擎不支持事务

   用的MySQL数据库默认使用支持事务的`innodb`引擎。一旦数据库引擎切换成不支持事务的`myisam`，那事务就从根本上失效了。

## 9. json 数据处理

- 过滤 json 数据

  **`@JsonIgnoreProperties` 作用在类上用于过滤掉特定字段不返回或者不解析。**

  ```java
  //生成json时将userRoles属性过滤
  @JsonIgnoreProperties({"userRoles"})
  public class User {
  
      private String userName;
      private String fullName;
      private String password;
      
      // `@JsonIgnore`一般用于类的属性上，作用和上面的`@JsonIgnoreProperties` 一样。
      @JsonIgnore
      private List<UserRole> userRoles = new ArrayList<>();
  }
  ```

- 格式化 json 数据

  `@JsonFormat`一般用来格式化 json 数据。：

  比如：

  ```java
  @JsonFormat(shape=JsonFormat.Shape.STRING, pattern="yyyy-MM-dd'T'HH:mm:ss.SSS'Z'", timezone="GMT")
  private Date date;
  ```

## 10. 测试相关

**`@ActiveProfiles`一般作用于测试类上， 用于声明生效的 Spring 配置文件。**

```
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ActiveProfiles("test")
@Slf4j
public abstract class TestBase {
  ......
}
```

**`@Test`声明一个方法为测试方法**

**`@Transactional`被声明的测试方法的数据会回滚，避免污染测试数据。**

**`@WithMockUser` Spring Security 提供的，用来模拟一个真实用户，并且可以赋予权限。**

```
    @Test
    @Transactional
    @WithMockUser(username = "user-id-18163138155", authorities = "ROLE_TEACHER")
    void should_import_student_success() throws Exception {
        ......
    }
```

# 4. Spring IoC 和 AOP

## IOC

IoC（Inverse of Control:控制反转）是一种**设计思想**，就是 **将原本在程序中手动创建对象的控制权，交由Spring框架来管理。** IoC 在其他语言中也有应用，并非 Spring 特有。 **IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个Map（key，value）,Map 中存放的是各种对象。**

IoC 最常见以及最合理的实现方式叫做**依赖注入**（Dependency Injection，简称 DI）。

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 **IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。** 在实际项目中一个 Service 类可能有几百甚至上千个类作为它的底层，假如我们需要实例化这个 Service，你可能要每次都要搞清这个 Service 所有底层类的构造函数，这可能会把人逼疯。如果利用 IoC 的话，你只需要配置好，然后在需要的地方引用就行了，这大大增加了项目的可维护性且降低了开发难度。

Spring 时代我们一般通过 XML 文件来配置 Bean，后来开发人员觉得 XML 文件来配置不太好，于是 SpringBoot 注解配置就慢慢开始流行起来。

推荐阅读：https://www.zhihu.com/question/23277575/answer/169698662

IoC源码阅读：https://javadoop.com/post/spring-ioc

**Spring IoC的初始化过程：**

![image-20200602172639655](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602172641.png)

## AOP

AOP(Aspect-Oriented Programming:面向切面编程)能够将那些与业务无关，**却为业务模块所共同调用的逻辑或责任（例如事务处理、日志管理、权限控制等）封装起来**，便于**减少系统的重复代码**，**降低模块间的耦合度**，并**有利于未来的可拓展性和可维护性**。

**Spring AOP就是基于动态代理的**，如果要代理的对象，实现了某个接口，那么Spring AOP会使用**JDK Proxy**，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用**Cglib** ，这时候Spring AOP会使用 **Cglib** 生成一个被代理对象的子类来作为代理，如下图所示：

[![SpringAOPProcess](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602174852)](https://camo.githubusercontent.com/c54f144f88f5e38e1adccfaebf3388eaeb45a333/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f323031392d362f537072696e67414f5050726f636573732e6a7067)

当然你也可以使用 AspectJ ,Spring AOP 已经集成了AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。

使用 AOP 之后我们可以把一些通用功能抽象出来，在需要用到的地方直接使用即可，这样大大简化了代码量。我们需要增加新功能时也方便，这样也提高了系统扩展性。日志功能、事务管理等等场景都用到了 AOP 。

**Spring AOP 和 AspectJ AOP 有什么区别：**

- **Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。** Spring AOP 基于代理(Proxying)，而 AspectJ 基于字节码操作(Bytecode Manipulation)。
- Spring AOP 已经集成了 AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。AspectJ 相比于 Spring AOP 功能更加强大，但是 Spring AOP 相对来说更简单，

如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择 AspectJ ，它比Spring AOP 快很多。

> 在1.6和1.7的时候，JDK动态代理的速度要比CGLib动态代理的速度要慢，但是并没有教科书上的10倍差距，在JDK1.8的时候，JDK动态代理的速度已经比CGLib动态代理的速度快很多了。

# 5. Spring MVC

**简单来说：**

客户端发送请求-> 前端控制器 DispatcherServlet 接受客户端请求 -> 找到处理器映射 HandlerMapping 解析请求对应的 Handler-> HandlerAdapter 会根据 Handler 来调用真正的处理器开处理请求，并处理相应的业务逻辑 -> 处理器返回一个模型视图 ModelAndView -> 视图解析器进行解析 -> 返回一个视图对象->前端控制器 DispatcherServlet 渲染数据（Moder）->将得到视图对象返回给用户

上图的一个笔误的小问题：Spring MVC 的入口函数也就是前端控制器 DispatcherServlet 的作用是接收请求，响应结果。

**流程说明（重要）：**

（1）客户端（浏览器）发送请求，直接请求到 DispatcherServlet。

（2）DispatcherServlet 根据请求信息调用 HandlerMapping，解析请求对应的 Handler。

（3）解析到对应的 Handler（也就是我们平常说的 Controller 控制器）后，开始由 HandlerAdapter 适配器处理。

（4）HandlerAdapter 会根据 Handler 来调用真正的处理器开处理请求，并处理相应的业务逻辑。

（5）处理器处理完业务后，会返回一个 ModelAndView 对象，Model 是返回的数据对象，View 是个逻辑上的 View。

（6）ViewResolver 会根据逻辑 View 查找实际的 View。

（7）DispaterServlet 把返回的 Model 传给 View（视图渲染）。

（8）把 View 返回给请求者（浏览器）

# 6.常见问题

## 6.1 bean 的生命周期

这部分网上有很多文章都讲到了，下面的内容整理自：https://yemengying.com/2016/07/14/spring-bean-life-cycle/ ，除了这篇文章，再推荐一篇很不错的文章 ：https://www.cnblogs.com/zrtqsk/p/3735273.html 。

- Bean 容器找到配置文件中 Spring Bean 的定义。
- Bean 容器利用 Java Reflection API 创建一个Bean的实例。
- 如果涉及到一些属性值 利用 `set()`方法设置一些属性值。
- 如果 Bean 实现了 `BeanNameAware` 接口，调用 `setBeanName()`方法，传入Bean的名字。
- 如果 Bean 实现了 `BeanClassLoaderAware` 接口，调用 `setBeanClassLoader()`方法，传入 `ClassLoader`对象的实例。
- 与上面的类似，如果实现了其他 `*.Aware`接口，就调用相应的方法。
- 如果有和加载这个 Bean 的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessBeforeInitialization()` 方法
- 如果Bean实现了`InitializingBean`接口，执行`afterPropertiesSet()`方法。
- 如果 Bean 在配置文件中的定义包含 init-method 属性，执行指定的方法。
- 如果有和加载这个 Bean的 Spring 容器相关的 `BeanPostProcessor` 对象，执行`postProcessAfterInitialization()` 方法
- 当要销毁 Bean 的时候，如果 Bean 实现了 `DisposableBean` 接口，执行 `destroy()` 方法。
- 当要销毁 Bean 的时候，如果 Bean 在配置文件中的定义包含 destroy-method 属性，执行指定的方法。

![Spring Bean 生命周期](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602205028)



## 6.2 Spring用到了哪些设计模式

关于下面一些设计模式的详细介绍，可以看笔主前段时间的原创文章[《面试官:“谈谈Spring中都用到了那些设计模式?”。》](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485303&idx=1&sn=9e4626a1e3f001f9b0d84a6fa0cff04a&chksm=cea248bcf9d5c1aaf48b67cc52bac74eb29d6037848d6cf213b0e5466f2d1fda970db700ba41&token=255050878&lang=zh_CN#rd) 。

### 工厂设计模式

Spring使用工厂模式可以通过 `BeanFactory` 或 `ApplicationContext` 创建 bean 对象。

**两者对比：**

- `BeanFactory` ：延迟注入(使用到某个 bean 的时候才会注入),相比于`BeanFactory`来说会占用更少的内存，程序启动速度更快。
- `ApplicationContext` ：容器启动的时候，不管你用没用到，一次性创建所有 bean 。`BeanFactory` 仅提供了最基本的依赖注入支持，`ApplicationContext` 扩展了 `BeanFactory` ,除了有`BeanFactory`的功能还有额外更多功能，所以一般开发人员使用`ApplicationContext`会更多。

ApplicationContext的三个实现类：

1. `ClassPathXmlApplication`：把上下文文件当成类路径资源。
2. `FileSystemXmlApplication`：从文件系统中的 XML 文件载入上下文定义信息。
3. `XmlWebApplicationContext`：从Web系统中的XML文件载入上下文定义信息。

### 代理设计模式

**Spring AOP 就是基于动态代理的**，如果要代理的对象，实现了某个接口，那么Spring AOP会使用**JDK Proxy**，去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候Spring AOP会使用**Cglib** ，这时候Spring AOP会使用 **Cglib** 生成一个被代理对象的子类来作为代理

### 单例设计模式

**Spring 中 bean 的默认作用域就是 singleton(单例)的。**Spring 通过 `ConcurrentHashMap` 实现单例注册表的特殊方式实现单例模式。Spring 实现单例的核心代码如下：

```
// 通过 ConcurrentHashMap（线程安全） 实现单例注册表
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
        synchronized (this.singletonObjects) {
            // 检查缓存中是否存在实例  
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //...省略了很多代码
                try {
                    singletonObject = singletonFactory.getObject();
                }
                //...省略了很多代码
                // 如果实例对象在不存在，我们注册到单例注册表中。
                addSingleton(beanName, singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
    }
    //将对象添加到单例注册表
    protected void addSingleton(String beanName, Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

            }
        }
}
```

### 模板方法模式

Spring 中 `jdbcTemplate`、`hibernateTemplate` 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。一般情况下，我们都是使用继承的方式来实现模板模式，但是 Spring 并没有使用这种方式，而是使用Callback 模式与模板方法模式配合，既达到了代码复用的效果，同时增加了灵活性。

### 装饰模式

装饰者模式可以动态地给对象添加一些额外的属性或行为。相比于使用继承，装饰者模式更加灵活。简单点儿说就是当我们需要修改原有的功能，但我们又不愿直接去修改原有的代码时，设计一个Decorator套在原有代码外面。其实在 JDK 中就有很多地方用到了装饰者模式，比如 `InputStream`家族，`InputStream` 类下有 `FileInputStream` (读取文件)、`BufferedInputStream` (增加缓存,使读取文件速度大大提升)等子类都在不修改`InputStream` 代码的情况下扩展了它的功能。

Spring 中配置 DataSource 的时候，DataSource 可能是不同的数据库和数据源。我们能否根据客户的需求在少修改原有类的代码下动态切换不同的数据源？这个时候就要用到装饰者模式。Spring 中用到的装饰模式在类名上含有 **`Wrapper`或者 `Decorator`**。这些类基本上都是动态地给一个对象添加一些额外的职责。

### 观察者模式

Spring 事件驱动模型就是观察者模式很经典的一个应用。Spring 事件驱动模型非常有用，在很多场景都可以解耦我们的代码。比如我们每次添加商品的时候都需要重新更新商品索引，这个时候就可以利用观察者模式来解决这个问题。

**事件角色：**

`ApplicationEvent` (`org.springframework.context`包下)充当事件的角色,这是一个抽象类，它继承了`java.util.EventObject`并实现了 `java.io.Serializable`接口。

Spring 中默认存在以下事件，他们都是对 `ApplicationContextEvent` 的实现(继承自`ApplicationContextEvent`)：

- `ContextStartedEvent`：`ApplicationContext` 启动后触发的事件;
- `ContextStoppedEvent`：`ApplicationContext` 停止后触发的事件;
- `ContextRefreshedEvent`：`ApplicationContext` 初始化或刷新完成后触发的事件;
- `ContextClosedEvent`：`ApplicationContext` 关闭后触发的事件。

**事件监听者角色：**

`ApplicationListener` 充当了事件监听者角色，它是一个接口，里面只定义了一个 `onApplicationEvent（）`方法来处理`ApplicationEvent`。接口定义看出接口中的事件只要实现了 `ApplicationEvent`就可以了。所以，在 Spring中我们只要实现 `ApplicationListener` 接口实现 `onApplicationEvent()` 方法即可完成监听事件

**事件发布者角色：**

`ApplicationEventPublisher` 充当了事件的发布者，它也是一个接口。

`ApplicationEventPublisher` 接口的`publishEvent（）`这个方法在`AbstractApplicationContext`类中被实现，阅读这个方法的实现，你会发现实际上事件真正是通过`ApplicationEventMulticaster`来广播出去的。

```java
// 定义一个事件,继承自ApplicationEvent并且写相应的构造函数
public class DemoEvent extends ApplicationEvent{
    private static final long serialVersionUID = 1L;

    private String message;

    public DemoEvent(Object source,String message){
        super(source);
        this.message = message;
    }

    public String getMessage() {
         return message;
          }


// 定义一个事件监听者,实现ApplicationListener接口，重写 onApplicationEvent() 方法；
@Component
public class DemoListener implements ApplicationListener<DemoEvent>{

    //使用onApplicationEvent接收消息
    @Override
    public void onApplicationEvent(DemoEvent event) {
        String msg = event.getMessage();
        System.out.println("接收到的信息是："+msg);
    }

}
// 发布事件，可以通过ApplicationEventPublisher  的 publishEvent() 方法发布消息。
@Component
public class DemoPublisher {

    @Autowired
    ApplicationContext applicationContext;

    public void publish(String message){
        //发布事件
        applicationContext.publishEvent(new DemoEvent(this, message));
    }
}
```

### 适配器模式

**spring AOP中的适配器模式：**

我们知道 Spring AOP 的实现是基于代理模式，但是 Spring AOP 的增强或通知(Advice)使用到了适配器模式，与之相关的接口是`AdvisorAdapter` 。Advice 常用的类型有：`BeforeAdvice`（目标方法调用前,前置通知）、`AfterAdvice`（目标方法调用后,后置通知）、`AfterReturningAdvice`(目标方法执行结束后，return之前)等等。每个类型Advice（通知）都有对应的拦截器:`MethodBeforeAdviceInterceptor`、`AfterReturningAdviceAdapter`、`AfterReturningAdviceInterceptor`。Spring预定义的通知要通过对应的适配器，适配成 `MethodInterceptor`接口(方法拦截器)类型的对象（如：`MethodBeforeAdviceInterceptor` 负责适配 `MethodBeforeAdvice`）。

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602215931.png)

**spring MVC中的适配器模式：**

在Spring MVC中，`DispatcherServlet` 根据请求信息调用 `HandlerMapping`，解析请求对应的 `Handler`。解析到对应的 `Handler`（也就是我们平常说的 `Controller` 控制器）后，开始由`HandlerAdapter` 适配器处理。`HandlerAdapter` 作为期望接口，具体的适配器实现类用于对目标类进行适配，`Controller` 作为需要适配的类。

**为什么要在 Spring MVC 中使用适配器模式？** Spring MVC 中的 `Controller` 种类众多，不同类型的 `Controller` 通过不同的方法来对请求进行处理。如果不利用适配器模式的话，`DispatcherServlet` 直接获取对应类型的 `Controller`，需要的自行来判断，像下面这段代码一样：

```java
if(mappedHandler.getHandler() instanceof MultiActionController){  
   ((MultiActionController)mappedHandler.getHandler()).xxx  
}else if(mappedHandler.getHandler() instanceof XXX){  
    ...  
}else if(...){  
   ...  
}  
```

## 6.3 循环依赖问题

《[Spring 如何解决循环依赖的问题](https://www.jianshu.com/p/8bb67ca11831)》

《[Spring加载单例对象探究](https://my.oschina.net/tjuzhao/blog/809505)》

循环依赖其实就是循环引用，也就是两个或则两个以上的bean互相持有对方，最终形成闭环。比如A依赖于B，B依赖于C，C又依赖于A。

```java
// 基于构造器的循环依赖，会导致启动失败
@Service
public class A {  
    public A(B b) {  }
}

@Service
public class B {  
    public B(C c) {  
    }
}

@Service
public class C {  
    public C(A a) {  }
}
```

```java
// field属性注入循环依赖，可以正常启动
@Service
public class A1 {  
    @Autowired  
    private B1 b1;
}

@Service
public class B1 {  
    @Autowired  
    public C1 c1;
}

@Service
public class C1 {  
    @Autowired  public A1 a1;
}
```

```java
// field属性注入循环依赖（prototype）--启动失败
@Service
@Scope("prototype")
public class A1 {  
    @Autowired  
    private B1 b1;
}

@Service
@Scope("prototype")
public class B1 {  
    @Autowired  
    public C1 c1;
}

@Service
@Scope("prototype")
public class C1 {  
    @Autowired  public A1 a1;
}
```

**现象总结**：同样对于循环依赖的场景，构造器注入和prototype类型的属性注入都会初始化Bean失败。因为@Service默认是单例的，所以单例的属性注入是可以成功的。

**依赖注入的过程：**

Spring创建好了BeanDefinition之后呢，**会开始实例化Bean**，并且**对Bean的依赖属性进行填充**。实例化时底层使用了CGLIB或Java反射技术。下图中instantiateBean核PopulateBean方法很重要！

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602230525.png)

```java
一级缓存：
/** 保存所有的singletonBean的实例 */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

二级缓存：
/** 保存所有早期创建的Bean对象，这个Bean还没有完成依赖注入 */
private final Map<String, Object> earlySingletonObjects = new HashMap<String, Object>(16);
三级缓存：
/** singletonBean的生产工厂*/
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<String, ObjectFactory<?>>(16);
 
/** 保存所有已经完成初始化的Bean的名字（name） */
private final Set<String> registeredSingletons = new LinkedHashSet<String>(64);
 
/** 标识指定name的Bean对象是否处于创建状态  这个状态非常重要 */
private final Set<String> singletonsCurrentlyInCreation =
    Collections.newSetFromMap(new ConcurrentHashMap<String, Boolean>(16));

```

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602230916.png)

![img](https://520li.oss-cn-hangzhou.aliyuncs.com/img/20200602230941.png)

**总结：**

- 1）AbstractBeanFactory的子类调用getBean()，从DefaultSingletonBeanRegistry中缓存查找对象，如果找到，返回；如果没有找到，进入下一步。
- 2）进入对象创建流程，首先判断是否是单例对象（本文只讨论单例对象）。
- 3）如果是单例对象，先加载其他依赖的对象
- 4）调用DefaultSingletonBeanRegistry的getSingleton(String beanName, ObjectFactory<?> singletonFactory) 创建对象。
- 5）在创建过程中，如果对象已经在提早曝光对象缓存中，即**earlySingletonObjects**，返回即可；否则，**首先将该对象加入正在创建缓存中**，然后创建对象，先放提前曝光单例对象缓存中，即earlySingletonObjects中。当该对象创建完毕后，将该对象从earlySingletonObjects中移除，并加入到单例对象缓存中，即singletonObjects 中。

**prototype问题：**

prototypeBean有一个关键的属性，保存着正在创建的prototype的beanName，在流程上并没有暴露任何factory之类的缓存。并且在beforePrototypeCreation(String beanName)方法时，把每个正在创建的prototype的BeanName放入一个set中，并且会循环依赖时检查beanName是否处于创建状态，如果是就抛出异常

## 6.4 spring-boot 自动配置

我们知道 `@SpringBootApplication`看作是 `@Configuration`、`@EnableAutoConfiguration`、`@ComponentScan` 注解的集合。

- `@EnableAutoConfiguration`：启用 SpringBoot 的自动配置机制
- `@ComponentScan`： 扫描被`@Component` (`@Service`,`@Controller`)注解的bean，注解默认会扫描该类所在的包下所有的类。
- `@Configuration`：允许在上下文中注册额外的bean或导入其他配置类

`@EnableAutoConfiguration`是启动自动配置的关键，源码如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
// 重点是Import注解
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	Class<?>[] exclude() default {};

	String[] excludeName() default {};

}
```

`@EnableAutoConfiguration` 注解通过Spring 提供的 `@Import` 注解导入了`AutoConfigurationImportSelector`类（`@Import` 注解稍后详细说明）。

`AutoConfigurationImportSelector`类中`getCandidateConfigurations`方法会将所有自动配置类的信息以 List 的形式返回。这些配置信息会被 Spring 容器作 bean 来管理。

```java
// 由于实现了ImportSelector接口（DeferredImportSelector继承的），在@import时会自动执行selectImports方法
public class AutoConfigurationImportSelector
		implements DeferredImportSelector, BeanClassLoaderAware, ResourceLoaderAware,
		BeanFactoryAware, EnvironmentAware, Ordered {
        
    // 返回一个包含所有自动配置类信息的字符串数组
    @Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return NO_IMPORTS;
		}
		AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader
				.loadMetadata(this.beanClassLoader);
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
		List<String> configurations = getCandidateConfigurations(annotationMetadata,
				attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return StringUtils.toStringArray(configurations);
	}
            
}
```

重点在于方法`getCandidateConfigurations()`返回了自动配置类的信息列表，而它通过调用`SpringFactoriesLoader.loadFactoryNames()`来扫描加载含有META-INF/spring.factories文件的jar包，该文件记录了具有哪些自动配置类。

```java
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata,
			AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations,
                    "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}

// SpringFactoriesLoader类下的方法
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
    String factoryClassName = factoryClass.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        Enumeration<URL> urls = (classLoader != null ?
                                 // public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";
                                 classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                                 ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                List<String> factoryClassNames = Arrays.asList(
                    StringUtils.commaDelimitedListToStringArray((String) entry.getValue()));
                result.addAll((String) entry.getKey(), factoryClassNames);
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

自动配置类一般都使用条件注解来标注是否需要加载，详见下文`@Conditional`注解

### `@import`

《[Springboot2 注解@Import的使用](https://www.cnblogs.com/happyflyingpig/p/12181569.html)》

@Import可以**导入bean或者@Configuration修饰的配置类**。如果配置类在标准的springboot的包结构下，就是SpringbootApplication启动类在包的根目录下，配置类在子包下。就不需要使用@Import导入配置类，如果**配置类在第三方的jar下**，我们想要引入这个配置类，就需要@Import对其引入到工程中才能生效。因为这个时候配置类不在springboot默认的扫描范围内。@Import 相当于Spring xml配置文件中的`<import /> `标签。

1）导入普通类

```java
@Getter
@Setter
public class MySqlConn {
    private String driverClass = "xxxx";
    private String url = "jdbc:mysql://localhost:3306/mydb";
    private String userName = "root";
    private String password = "root";

}

// 在springboot的启动类@Import这个类，这个类就可以被其他类当做bean来引用。
@SpringBootApplication
@Import({MySqlConn.class})
public class SpringBootApplication1 {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootApplication1.class, args);
    }

}
```

2）和@Configuration搭配使用

```java
@Configuration
public class MySqlConfig {
    @Bean
    public MySqlServerBean getBean() {
        MySqlServerBean bean = new MySqlServerBean();
        bean.setUserName("lyj");
        return bean;
    }
}
```

3）和`ImportSelector`搭配使用

**自定义一个SpringStartSelector实现接口ImportSelector，BeanFactoryAware。重写selectImports()方法和setBeanFactory()方法。这里在程序启动的时候会先运行setBeanFactory()方法，我们可以在这里获取到beanFactory。selectImports()方法回了了AppCoinfig的全类名。**

```java
// 
public class SpringStartSelector implements ImportSelector, BeanFactoryAware {
    private BeanFactory beanFactory;

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        System.out.println("importingClassMetaData:" + importingClassMetadata);
        System.out.println("beanFactory:" + beanFactory);
        return new String[]{AppConfig.class.getName()};
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
}
```

### `@Conditional`

《[SpringBoot(15)—@Conditional注解](https://www.cnblogs.com/qdhxhz/p/11020434.html)》

自动配置类一般都使用条件注解来标注是否需要加载，@Conditional是由Spring 4提供的一个新特性，用于根据特定条件来控制Bean的创建行为。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
    // 传入Condition的子类
    Class<? extends Condition>[] value();
}

```

```java
@FunctionalInterface
public interface Condition {
	
    // 实现类需要实现该方法，通过该方法控制bean是否加载
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

- @ConditionalOnBean         //	当给定的在bean存在时,则实例化当前Bean 
- @ConditionalOnMissingBean  //	当给定的在bean不存在时,则实例化当前Bean 
- @ConditionalOnClass        //	当给定的类名在类路径上存在，则实例化当前Bean 
- @ConditionalOnMissingClass //	当给定的类名在类路径上不存在，则实例化当前Bean

**@ConditionalOnBean注解实现：**

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnBeanCondition.class)
public @interface ConditionalOnBean {
    /**
     * 需要作为条件的类的Class对象数组
     */
    Class<?>[] value() default {};
    /**
     * 需要作为条件的类的Name,Class.getName()
     */
    String[] type() default {};
    /**
     *  (用指定注解修饰的bean)条件所需的注解类
     */
    Class<? extends Annotation>[] annotation() default {};
    /**
     * spring容器中bean的名字
     */
    String[] name() default {};
    /**
     * 搜索容器层级,当前容器,父容器
     */
    SearchStrategy search() default SearchStrategy.ALL;
    /**
     * 可能在其泛型参数中包含指定bean类型的其他类
     */
    Class<?>[] parameterizedContainer() default {};
}
```

```java
@Order(Ordered.HIGHEST_PRECEDENCE)
class OnClassCondition extends SpringBootCondition
		implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {

	private BeanFactory beanFactory;

	private ClassLoader beanClassLoader;

	@Override
	public boolean[] match(String[] autoConfigurationClasses,
			AutoConfigurationMetadata autoConfigurationMetadata) {
		ConditionEvaluationReport report = getConditionEvaluationReport();
		ConditionOutcome[] outcomes = getOutcomes(autoConfigurationClasses,
				autoConfigurationMetadata);
		boolean[] match = new boolean[outcomes.length];
		for (int i = 0; i < outcomes.length; i++) {
			match[i] = (outcomes[i] == null || outcomes[i].isMatch());
			if (!match[i] && outcomes[i] != null) {
				logOutcome(autoConfigurationClasses[i], outcomes[i]);
				if (report != null) {
					report.recordConditionEvaluation(autoConfigurationClasses[i], this,
							outcomes[i]);
				}
			}
		}
		return match;
	}
    
    @Override
	public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
		this.beanFactory = beanFactory;
	}

	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.beanClassLoader = classLoader;
	}
}
```

## 6.5 常见的扩展点

### BeanDefinition与BeanFactory扩展

#### BeanDefinitionRegistryPostProcessor接口

```java
/**
 * Extension to the standard {@link BeanFactoryPostProcessor} SPI, allowing for
 * the registration of further bean definitions <i>before</i> regular
 * BeanFactoryPostProcessor detection kicks in. In particular,
 * BeanDefinitionRegistryPostProcessor may register further bean definitions
 * which in turn define BeanFactoryPostProcessor instances.
 *
 * @author Juergen Hoeller
 * @since 3.0.1
 * @see org.springframework.context.annotation.ConfigurationClassPostProcessor
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

    /**
     * Modify the application context's internal bean definition registry after its
     * standard initialization. All regular bean definitions will have been loaded,
     * but no beans will have been instantiated yet. This allows for adding further
     * bean definitions before the next post-processing phase kicks in.
     * @param registry the bean definition registry used by the application context
     * @throws org.springframework.beans.BeansException in case of errors
     */
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

这个接口扩展了标准的BeanFactoryPostProcessor 接口，允许在普通的BeanFactoryPostProcessor接口实现类执行之前注册更多的BeanDefinition。特别地是，BeanDefinitionRegistryPostProcessor可以注册BeanFactoryPostProcessor的BeanDefinition。

postProcessBeanDefinitionRegistry方法**可以修改在BeanDefinitionRegistry接口实现类中注册的任意BeanDefinition，也可以增加和删除BeanDefinition**。原因是**这个方法执行前所有常规的BeanDefinition已经被加载到BeanDefinitionRegistry接口实现类中，但还没有bean被实例化**。

#### BeanFactoryPostProcessor接口

BeanFactory生成后，如果想对BeanFactory进行一些处理，该怎么办呢？BeanFactoryPostProcessor接口就是用来处理BeanFactory的。

```java
/**
 * Allows for custom modification of an application context's bean definitions,
 * adapting the bean property values of the context's underlying bean factory.
 *
 * <p>Application contexts can auto-detect BeanFactoryPostProcessor beans in
 * their bean definitions and apply them before any other beans get created.
 *
 * <p>Useful for custom config files targeted at system administrators that
 * override bean properties configured in the application context.
 *
 * <p>See PropertyResourceConfigurer and its concrete implementations
 * for out-of-the-box solutions that address such configuration needs.
 *
 * <p>A BeanFactoryPostProcessor may interact with and modify bean
 * definitions, but never bean instances. Doing so may cause premature bean
 * instantiation, violating the container and causing unintended side-effects.
 * If bean instance interaction is required, consider implementing
 * {@link BeanPostProcessor} instead.
 *
 * @author Juergen Hoeller
 * @since 06.07.2003
 * @see BeanPostProcessor
 * @see PropertyResourceConfigurer
 */
public interface BeanFactoryPostProcessor {

    /**
     * Modify the application context's internal bean factory after its standard
     * initialization. All bean definitions will have been loaded, but no beans
     * will have been instantiated yet. This allows for overriding or adding
     * properties even to eager-initializing beans.
     * @param beanFactory the bean factory used by the application context
     * @throws org.springframework.beans.BeansException in case of errors
     */
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```

这个接口允许**自定义修改应用程序上下文的BeanDefinition，调整上下文的BeanFactory的bean属性值**。应用程序上下文可以在BeanFactory的BeanDefinition中自动检测BeanFactoryPostProcessor bean，并在创建任何其他bean之前应用它们。对于定位于系统管理员的自定义配置文件非常有用，它们将覆盖应用程序上下文中配置的bean属性。请参阅PropertyResourceConfigurer及其具体实现，了解解决此类配置需求的开箱即用解决方案。BeanFactoryPostProcessor可能与bean定义交互并修改，但永远不应该将bean实例化。 这样做可能会导致过早的bean实例化，违反容器执行顺序并导致意想不到的副作用。如果需要bean实例交互，请考虑实现BeanPostProcessor接口。

postProcessBeanFactory方法**在BeanFactory初始化后，所有的bean定义都被加载，但是没有bean会被实例化时，允许重写或添加属性**。

### Bean实例化中的扩展

#### BeanPostProcessor接口

```java
/**
 * Factory hook that allows for custom modification of new bean instances,
 * e.g. checking for marker interfaces or wrapping them with proxies.
 *
 * <p>ApplicationContexts can autodetect BeanPostProcessor beans in their
 * bean definitions and apply them to any beans subsequently created.
 * Plain bean factories allow for programmatic registration of post-processors,
 * applying to all beans created through this factory.
 *
 * <p>Typically, post-processors that populate beans via marker interfaces
 * or the like will implement {@link #postProcessBeforeInitialization},
 * while post-processors that wrap beans with proxies will normally
 * implement {@link #postProcessAfterInitialization}.
 *
 * @author Juergen Hoeller
 * @since 10.10.2003
 * @see InstantiationAwareBeanPostProcessor
 * @see DestructionAwareBeanPostProcessor
 * @see ConfigurableBeanFactory#addBeanPostProcessor
 * @see BeanFactoryPostProcessor
 */
public interface BeanPostProcessor {

    /**
     * Apply this BeanPostProcessor to the given new bean instance <i>before</i> any bean
     * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
     * or a custom init-method). The bean will already be populated with property values.
     * The returned bean instance may be a wrapper around the original.
     * @param bean the new bean instance
     * @param beanName the name of the bean
     * @return the bean instance to use, either the original or a wrapped one;
     * if {@code null}, no subsequent BeanPostProcessors will be invoked
     * @throws org.springframework.beans.BeansException in case of errors
     * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
     */
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

    /**
     * Apply this BeanPostProcessor to the given new bean instance <i>after</i> any bean
     * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
     * or a custom init-method). The bean will already be populated with property values.
     * The returned bean instance may be a wrapper around the original.
     * <p>In case of a FactoryBean, this callback will be invoked for both the FactoryBean
     * instance and the objects created by the FactoryBean (as of Spring 2.0). The
     * post-processor can decide whether to apply to either the FactoryBean or created
     * objects or both through corresponding {@code bean instanceof FactoryBean} checks.
     * <p>This callback will also be invoked after a short-circuiting triggered by a
     * {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation} method,
     * in contrast to all other BeanPostProcessor callbacks.
     * @param bean the new bean instance
     * @param beanName the name of the bean
     * @return the bean instance to use, either the original or a wrapped one;
     * if {@code null}, no subsequent BeanPostProcessors will be invoked
     * @throws org.springframework.beans.BeansException in case of errors
     * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
     * @see org.springframework.beans.factory.FactoryBean
     */
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```

这个接口，**允许自定义修改新的bean实例，例如检查标记接口或用代理包装**，注意，如果有相互依赖的bean，这里可能无法使用代理。

postProcessBeforeInitialization方法，在任何bean初始化回调（如InitializingBean的afterPropertiesSet或自定义init方法）之前，将此BeanPostProcessor应用于给定的新的bean实例。 这个bean已经被填充了属性值。 返回的bean实例可能是原始的包装器。

postProcessAfterInitialization方法，在Bean初始化回调（如InitializingBean的afterPropertiesSet或自定义init方法）之后，将此BeanPostProcessor应用于给定的新bean实例。 这个bean已经被填充了属性值。 返回的bean实例可能是原始的包装器。这个方法也会在InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation方法生成对象后再次不让他生成对象（具体可以参考Spring生成bean的过程）。

### 用来感知IOC容器的各种Aware扩展

#### BeanNameAware

实现该接口并重写void setBeanName(String var1)方法；获取该bean在BeanFactory配置中的名字

#### ApplicationContextAware

实现该接口，并重写setApplicationContext(ApplicationContext applicationContext)方法，获取spring 上下文环境的对象，然后通过该上下文对象获取spring容器中的bean对象

#### BeanFactoryAware

实现该接口，并重写void setBeanFactory(BeanFactory beanFactory) 方法，Bean获取配置他们的BeanFactory的引用

#### BeanClassLoaderAware

实现该接口，并重写void setBeanClassLoader(ClassLoader var1)方法，获取Bean的类加载器

#### ServletContextAware

实现该接口，并重写void setServletContext(ServletContext servletContext)方法；获取servletContext容器。

#### ResourceLoaderAware

实现该接口，并重写void setServletContext(ServletContext servletContext)方法；获取ResourceLoader对象，便能够通过它获得各种资源。

## 6.6 BeanFactory 与 FactoryBean

**BeanFacotry是spring中比较原始的Factory。如XMLBeanFactory就是一种典型的BeanFactory。原始的BeanFactory无法支持spring的许多插件，如AOP功能、Web应用等。** 
**ApplicationContext接口,它由BeanFactory接口派生而来，ApplicationContext包含BeanFactory的所有功能，通常建议比BeanFactory优先**

BeanFactory和FactoryBean的区别

- BeanFactory是接口，提供了OC容器最基本的形式，给具体的IOC容器的实现提供了规范，

- FactoryBean也是接口，IOC容器中Bean的实现提供了更加灵活的方式，FactoryBean在IOC容器的基础上**给Bean的实现加上了一个简单工厂模式和装饰模式**。 我们可以在getObject()方法中灵活配置。其实在Spring源码中有很多FactoryBean的实现类。

区别：**BeanFactory是个Factory**，也就是IOC容器或对象工厂，**FactoryBean是个Bean**。在Spring中，**所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的**。但对FactoryBean而言，**这个Bean不是简单的Bean，而是一个能生产或者修饰对象生成的工厂Bean,它的实现与设计模式中的工厂模式和修饰器模式类似** 

# 7.如何搭建spring-boot-starter

我们可以认为starter是一种服务,在使用某个功能的开发者不需要关注各种依赖库的处理，不需要具体的配置信息，由Spring Boot自动注入的bean。

我们在开发springboot项目的时候,经常会引用spring-boot-starter-web,spring-boot-starter-data-jpa等一些常用的依赖,我们也可以自己封装一个服务,然后再需要用的项目中引入依赖即可使用.

首先先创建一个Maven项目

**1.引入构建starter的核心依赖**

```xml
<properties>
     <srping.boot.version>2.3.0.RELEASE</srping.boot.version>
     <slf4j.version>1.7.25</slf4j.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-autoconfigure</artifactId>
        <version>${srping.boot.version}</version>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot</artifactId>
        <version>${srping.boot.version}</version>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-configuration-processor</artifactId>
        <version>${srping.boot.version}</version>
    </dependency>

    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j.version}</version>
    </dependency>
</dependencies>
```

**2.创建配置类**

通过@ConfigurationProperties注解绑定前缀是my.db得属性

```java
@ConfigurationProperties(prefix = "my.db")
public class DbConfig {
    private String username;
    private String password;
    //省略get,set方法
}
```

**3.封装服务提供者**

该类只有一个toString方法输出属性信息

```java
public class DbServer {

    private String username;
    private String password;
    //省略get,set方法
 	@Override
    public String toString(){
        return "DbServer{username=" + username + " ,password=" + password + "}";
    }
}
```

**4.注册Bean对象**

下面代码通关EnableConfigurationProperties注入了DbConfig对象,然后把属性赋值给了DbServer对象,最后调用了print打印方法

```java
@Configuration
@EnableConfigurationProperties(DbConfig.class)
public class DbAutoConfiguration {
    Logger logger= LoggerFactory.getLogger(this.getClass());
    @Bean
    public DbServer getBean(DbConfig dbConfig) {
        //创建组件实例
        logger.info("开始注册我的Bean对象");
        DbServer dbServer = new DbServer();
        dbServer.setUsername(dbConfig.getUsername());
        dbServer.setPassword(dbConfig.getPassword());
        return dbServer;
    }
}
```

**5.暴露需要被装配的类**

在resources目录下新建**META-INF/spring.factories**文件

第2行添加需要暴露的类的名称(包含包名)

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.ljm.boot.starter.config.DbAutoConfiguration
```

该文件会被 @SpringBootApplication注解中会下面被扫描并加载.

- @SpringBootApplication
  - @EnableAutoConfiguration
    - @Import({AutoConfigurationImportSelector.class})
      - getCandidateConfigurations()
        - loadFactoryNames() 中被扫描到并加载