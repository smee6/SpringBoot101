# IoC (Inversion of Control)
- 제어의 역전, 원래 객체랑 의존성 개발자가 하던걸 컨테이너(프레임워크)에게 맡겨서 제어하게 하는 방식
- Spring IoC Container 뭐 이런 개념으로 보면 될 듯

# DI (의존성 주입) 컨테이너
- 객체 생성,의존성 주입, 생명주기 관리 => Spring 핵심 엔진
- ApplicationContext
- 동작흐름
  1. @SpringBootApplication => @ComponentScan
  2. 패키지+하위패키지에서 @Component 계열 클래스들 찾고 => Bean으로 생성해서 등록 => Bean간 의존성 분석
  3. 필요한 Bean을 생성자/필드/메서드에 각각 주입 (의존성 주입, DI)
  4. 초기화 @PostConstructor => 이후 앱에서 사용 (어플리케이션 로직)

# 의존성 주입
- 의존성 : 한 객체가 다른 객체를 사용해야만 동작하는거 (컨트롤러에서 서비스 호출하고 그러는거지 뭐)
- 근데 뭐가 다르냐면 원래는
```java
public class UserController {
    private UserService userService = new UserService(); // 직접 생성
}
```
이렇게 하던걸 스프링에서는 직접 안만들고 
```java
@RestController
public class UserController {
    private final UserService userService;

    // 생성자 주입
    public UserController(UserService userService) {
        this.userService = userService; // 컨테이너가 주입
    }
}
```
이런식으로 해서 생성자에서 저렇게 해두면 컨테이너가 주입함 
=> @Autowired가 붙어있는데 생략되었다고 보면됨 스프링 4.3 이상부턴 생성자 1개면 자동으로 된다고 치고 생략함

## @Autowired
- Spring IoC 컨테이너가 관리하는 Bean을 자동으로 주입해주는 어노테이션
- 개발자가 new로 객체를 직접 생성하지 않고, 컨테이너가 생성한 Bean을 가져와서 연결해줌
- DI(Dependency Injection를 구현하는 방법 중 하나

# @SpringBootApplication 
```java
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}
```
- 세개의 어노테이션을 합친 메타 어노테이션
- @SpringBootConfiguration : @Configuration과 동일 → Bean 정의 클래스임을 표시
- @EnableAutoConfiguration : Spring Boot의 자동 설정(Auto Configuration) 기능 활성화, 클래스패스, 설정 파일 등을 보고 필요한 Bean을 자동 등록
- @ComponentScan : 현재 패키지와 하위 패키지에서 @Component, @Service, @Repository, @Controller 등을 찾아 Bean 등록

## 스프링 서버 시작 원리
- Spring Boot는 내장 웹 서버를 포함하고 있어서,spring-boot-starter-web 의존성을 추가하면 Tomcat이 기본 내장
- SpringApplication.run() 과정에서 ServletWebServerFactory Bean이 생성
- Tomcat 인스턴스가 초기화되고, 지정한 포트(기본 8080)에서 HTTP 요청 대기
- @RestController나 @Controller로 등록된 Bean이 DispatcherServlet에 매핑되어 요청 처리

### 어떻게 Spring Boot에서 Url을 가로채서 뭔가 동작을 하는가?
- Spring Boot는 ServletWebServerFactory를 통해 Tomcat을 띄움
- Spring MVC 초기화 과정에서 DispatcherServlet Bean이 자동 생성 및 등록
- 모든 요청 URL("/")을 DispatcherServlet이 가로채서 처리

### 서블렛? Servlet이 뭐야
```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.getWriter().write("Hello Servlet!");
    }
}
// 순수 서블렛의 예
```
- Servlet은 Java 기반 웹 애플리케이션에서 HTTP 요청과 응답을 처리하는 클래스
- Java EE 표준에서 제공하는 웹 컴포넌트
- 웹 서버(Tomcat, Jetty 등)에 의해 실행
- 요청을 받고, 응답을 만들어서 클라이언트에 반환

### Dispatcher가 뭔가.
- "분배기" 또는 "중앙 라우터" 역할
- 요청을 받아서 적절한 처리기로 전달하는 역할
- 예: /users 요청은 UserController로, /orders 요청은 OrderController로

### DispatcherServlet
```text
DispatcherServlet = Servlet + Dispatcher
Spring MVC에서 모든 HTTP 요청을 받아서 적절한 Controller로 전달하는 중앙 서블릿
```
<동작의 흐름>  
1. 클라이언트 요청 → Tomcat이 HTTP 요청을 받음
2. Tomcat이 DispatcherServlet에 요청을 전달
3. DispatcherServlet이 HandlerMapping을 통해 어떤 Controller가 처리할지 찾음
4. 해당 Controller 메서드 실행
5. 결과(Model + View)를 받아서 ViewResolver로 응답 생성
6. HTTP 응답 반환

# Bean
- 스프링이 직접 생성하고 Lifecycle도 관리하는 객체
- 자동등록과 수동등록으로 나뉨

### 자동등록
| 어노테이션	| 용도	| 특징
|---|---|---|
|@Component	| 일반적인 Bean 등록	| 범용, 모든 클래스에 사용 가능|
|@Service	| 비즈니스 로직 Bean	| 의미상 서비스 레이어 표시|
|@Repository| 데이터 접근 Bean	| JPA 예외 변환(AOP) 기능 포함|
|@Controller	| 웹 요청 처리 Bean	| MVC 컨트롤러, View 반환|
|@RestController	| REST API 처리 Bean | JSON 반환, @Controller + @ResponseBody|
- @Component 계열 애노테이션을 클래스에 붙이면, **@ComponentScan**이 해당 클래스를 찾아 Bean으로 등록
- Spring은 DI 컨테이너에 등록하고 생명주기까지 관리
- 자동 등록은 클래스 자체가 Bean
  
### 수동등록
@Configuration 클래스 만들고 그 안에 메소드를 @Bean 달아서 등록
```java
package com.example.demo.config;

import com.example.demo.service.UserService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// @Configuration: Bean 정의 클래스
@Configuration
public class AppConfig {

    // @Bean: 메서드 반환 객체를 Bean으로 등록
    @Bean
    public UserService userService() {
        return new UserService(); // 직접 생성
    }
}
```

### 차이점
- 자동 등록을 기본으로 사용
- 자동 등록은 @ComponentScan 범위 안에 있어야 함
  
---
## @Repository
- 단순 표시가 아니라 Spring Data Access Exception 변환기가 적용
- 예: JPA, JDBC에서 발생하는 DB 예외를 Spring의 일관된 예외 계층으로 변환
- Node.js로 비유하면, DB 드라이버의 다양한 에러를 공통 포맷으로 변환하는 미들웨어 같은 역할

## @Controller
- Spring MVC에서 View 반환을 위한 컨트롤러로 인식
- @RequestMapping과 함께 사용 시 DispatcherServlet이 라우팅 처리
- 반환값이 View 이름으로 해석됨 (ex: return "home"; → home.html 렌더링)

## @ResponseBody
- 기본적으로 @Controller 메서드는 View 이름을 반환하고, ViewResolver가 HTML을 렌더링
- @ResponseBody를 붙이면 ViewResolver를 거치지 않고 반환값을 HTTP Body에 직접 작성
- 주로 JSON, XML, 문자열 응답을 보낼 때 사용
```java
@Controller
public class UserController {

    @GetMapping("/user")
    @ResponseBody
    public User getUser() {
        return new User("Tom", "tom@example.com");
    }
}
// User 객체가 JSON으로 변환되어 응답
// 변환은 Jackson 라이브러리(Spring Boot 기본 포함)가 담당
```

## @RestController
- @Controller + @ResponseBody 합성 애노테이션
- 반환값을 JSON/XML 등 HTTP 바디에 직접 작성
- View 렌더링 없이 API 응답 전용
```java
@RestController
public class UserApi {
    @GetMapping("/user")
    public User getUser() {
        return new User("Tom", "tom@example.com");
    }
}
```
## @Component & @Service
- **@Component**와 **@Service**는 기능적으로 동일
- 차이는 **이 클래스가 어떤 역할을 하는지**를 개발자와 유지보수 팀에게 알려주는 **의미**
- @Service → 비즈니스 로직 담당 클래스
- @Component → 범용 유틸, 설정, 기타 Bean

## @RequestMapping
- URL + HTTP Method를 Controller 메서드에 매핑
- 가장 범용적인 매핑 어노테이션
- 클래스 레벨과 메서드 레벨 모두 사용 가능
```java
@Controller
@RequestMapping("/users") // 클래스 레벨: 공통 URL prefix
public class UserController {

    @RequestMapping(method = RequestMethod.GET) // GET 요청 처리
    public String getUsers() {
        return "userList";
    }

    @RequestMapping(method = RequestMethod.POST) // POST 요청 처리
    public String createUser() {
        return "userCreated";
    }
}
```
- 단점: HTTP Method를 지정하려면 RequestMethod.GET처럼 길게 써야 함 → 그래서 @GetMapping, @PostMapping 같은 단축형이 생김

## @GetMapping , @PostMapping
```java
//이건 getmapping의 예제

@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return new User(id, "Tom");
    }
}
// 아래는 postmapping

@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public User createUser(@RequestBody User user) {
        // DB 저장 로직
        return user;
    }
}
```

## @ResponseStatus
- Controller 메서드의 HTTP 응답 상태 코드를 지정
- 예외 처리 클래스(@ExceptionHandler)에도 사용 가능
```java
@ResponseStatus(HttpStatus.CREATED) // 201 Created
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    return user;
}
// ➡ 응답 상태 코드가 201 Created로 설정

@ResponseStatus(HttpStatus.NOT_FOUND) // 404
public class UserNotFoundException extends RuntimeException {}
➡ 해당 예외가 발생하면 자동으로 404 응답 반환
```

## @PathVariable
- URL 경로에 있는 값을 메서드 파라미터로 바인딩
- REST API에서 리소스 식별자(ID 등)를 전달할 때 사용
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return new User(id, "Tom");
}
//➡ /users/5 요청 시 id = 5로 매핑

@GetMapping("/users/{userId}")
public User getUser(@PathVariable("userId") Long id) { ... }
// URL 경로의 {} 안 이름과 파라미터 이름이 같아야 함
// 이름이 다르면 @PathVariable("경로변수명")으로 지정
```

## @RequestParam
- 쿼리 파라미터(?key=value)나 폼 데이터를 메서드 파라미터로 바인딩
- GET 요청에서 자주 사용
```java
@GetMapping("/search")
public String search(@RequestParam String keyword) {
    return "Searching for: " + keyword;
}
// ➡ /search?keyword=Spring → keyword = "Spring"

@RequestParam(defaultValue = "guest") String name
@RequestParam(required = false) String email

//defaultValue: 값이 없을 때 기본값 설정
//required: 필수 여부 지정 (기본 true)
```

## @RequestBody
- HTTP 요청 Body(JSON, XML 등)를 Java 객체로 변환
- POST, PUT 요청에서 주로 사용
- 변환은 HttpMessageConverter가 담당 (Spring Boot 기본: Jackson)
```java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    return user;
}
// ➡ User 객체로 자동 변환
// 대규모 API에서는 DTO(Data Transfer Object)로 받는 것이 안전
```
### 잠깐만? 자동으로 객체 변환?
- 내부적으로는 이런게 되는거랑 비슷한데 스프링이 자동으로 해줌
```java
ObjectMapper mapper = new ObjectMapper();
User user = mapper.readValue(jsonString, User.class);
```
- Spring Boot에서는 이 변환 과정을 HttpMessageConverter가 담당
- 기본적으로 Jackson 라이브러리가 포함되어 있어서, JSON ↔ Java 객체 변환이 자동
- 그래서 가능하면 아래처럼 DTO랑 같이 쓰면 좋다
```java
// DTO 클래스
public class CreateUserRequest {
    @NotNull
    private String name;

    @Email
    private String email;

    // Getter/Setter....도 있다고 치고
}

// Controller
@RestController
@RequestMapping("/users")
public class UserController {

    @PostMapping
    public User createUser(@RequestBody CreateUserRequest request) {
        // DTO → Entity 변환
        User user = new User(request.getName(), request.getEmail());

        // DB 저장 로직...
        return user;
    }
}
```
