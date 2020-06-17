# Spring WebFlux - Spring 5 WebFlux 导读



标签: `翻译`, `Reactive`, `WebFlux`, `Spring Web`, `HTTP 客户端`

原文: [Guide to Spring 5 WebFlux](https://www.baeldung.com/spring-webflux)


### 1. 概述

`Spring WebFlux`在Spring 5中引入，提供了构建响应式web应用的能力。

在本示例中，我们会使用响应式web组件`RestController`和`WebClient`，来构建一个小小的响应式REST应用。

我们也会考虑使用`Spring Security`来增强上面构建的响应式应用的安全性。

### 2. Spring WebFlux 框架

`Spring WebFlux`并没有重复造轮子，而是在内部直接使用[Project Reactor](https://projectreactor.io/)和它的`publisher`实现: `Flux`和`Mono`。

新框架支持两种编程模式：
* 注解方式
* 函数方式

本文我们只关注注解方式的。函数方式的在另一篇文章：[Spring5中的函数式Web框架](https://www.baeldung.com/spring-5-functional-web)



### 3. 依赖

让我们从引入`spring-boot-starter-webflux`依赖起步。这个依赖会把所有其他必需依赖都引入进来：
* `spring-boot`和`spring-boot-starter`基础依赖
* `spring-webflux`框架
* 响应式streams所需的`reactor-core`，还有`reactor-netty`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

最新的[spring-boot-starter-webflux](https://search.maven.org/classic/#search%7Cgav%7C1%7Cg%3A%22org.springframework.boot%22%20AND%20a%3A%22spring-boot-starter-webflux%22)可以在Maven Central仓库中下载

### 4. Reactive REST 应用

我们下面使用`Spring WebFlux`构建一个很简单的响应式REST应用 - 员工管理系统(`EmployeeManagement`):
* 我们会使用一个简单的`Employee` model，包含`id`和`name`
* 我们会使用`RestController`和`WebClient`构建一些REST API，来发布(publish)和获取(retrieve)单个或多个`Employee`
* 我们也会使用`WebFlux`和`Spring Security`来构建安全的响应式终端


### 5. Reactive RestController

`Spring WebFlux`支持使用注解方式进行配置。

首先，我们创建一个被注解注释的Controller来提供`Employee`数据流

```java
@RestController
@RequestMapping("/employees")
public class EmployeeController {

    private final EmployeeRepository employeeRepository;

    // constructor...
}
```

`EmployeeRepository`可以是任何非阻塞响应式流的数据仓库。


#### 5.1 单个资源

下面我们创建一个获取单个资源的Controller

```java
@GetMapping("/{id}")
private Mono<Employee> getEmployeeById(@PathVariable String id) {
    return employeeRepository.findEmployeeById(id);
}
```

为了获取单个资源，这里使用的`Mono`类型的`Employee`。因为`Mono`类型最多会吐出一个元素。

#### 5.2 多个资源

我们也会创建一个获取所有`Employee`的Controller。

```java
@GetMapping
private Flux<Employee> getAllEmployees() {
    return employeeRepository.findAllEmployees();
}
```

为了获取多个资源，这里使用了`Flux`类型的`Employee`。因为`Flux`类型会吐出0~n个元素。


### 6. Reactive Web Client

Spring 5中引入的 [`WebClient`](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client) 是一个支持响应式流的非阻塞客户端

在客户端的角度，我们使用`WebClient`从上面创建的Controller中获取数据

首先创建一个简单的`EmployeeWebClient`
```java
public class EmployeeWebClient {

    WebClient client = WebClient.create("http://localhost:8080");

    // ...
}
```
上面使用工厂方法创建了一个`WebClient`。

#### 6.1 获取单个资源
访问`/employee/{id}`接口获取单个资源

```java
Mono<Employee> employeeMono = client.get()
  .uri("/employees/{id}", "1")
  .retrieve()
  .bodyToMono(Employee.class);

employeeMono.subscribe(System.out::println);
```

#### 6.2 获取多个资源
访问`/employees`接口获取多个资源

```java
Flux<Employee> employeeFlux = client.get()
  .uri("/employees")
  .retrieve()
  .bodyToFlux(Employee.class);

employeeFlux.subscribe(System.out::println);
```


### 7. Spring WebFlux Security

我们可以用`Spring Security`来增加上面构建的响应式服务的安全性。

假设在`EmployeeController`新增了一个方法，这个方法能够更新`Employee`的内容。为了限制数据不被随意修改，我们希望只有`ADMIN`这样一个角色可以进行这种操作。

如下在`EmployeeController`新增一个方法：
```java
@PostMapping("/update")
private Mono<Employee> updateEmployee(@RequestBody Employee employee) {
    return employeeRepository.updateEmployee(employee);
}
```

为了限制上述方法的访问权限，我们创建`SecurityConfig`配置类，来先限定接口只能被`ADMIN`角色访问：
```java
@EnableWebFluxSecurity
public class EmployeeWebSecurityConfig {

    // ...

    @Bean
    public SecurityWebFilterChain springSecurityFilterChain(
      ServerHttpSecurity http) {
        http.csrf().disable()
          .authorizeExchange()
          .pathMatchers(HttpMethod.POST, "/employees/update").hasRole("ADMIN")
          .pathMatchers("/**").permitAll()
          .and()
          .httpBasic();
        return http.build();
    }
}
```

`@EnableWebFluxSecurity`注解为Spring Security WebFlux提供了一些通用配置。详情请见另一篇文章：[configuring and working with Spring WebFlux security](https://www.baeldung.com/spring-security-5-reactive)


### 8. 总结

在本文中，我们简单学习了怎样使用`Spring WebFlux`构建一个响应式应用，也学习了怎样使用`RestController`和`WebClient`来发布(publish, 响应式工程专用术语)和消费(consume)响应式的流(stream)。我们还学习了怎样使用`Spring Security`来增强服务的安全性。

除了响应式`RestController`和`WebClient`，`WebFlux`框架还支持响应式`WebSocket`和`WebSocketClient`，来使用Socket类型的响应式流。详情请见文章[working with Reactive WebSocket with Spring 5](https://www.baeldung.com/spring-webflux)




