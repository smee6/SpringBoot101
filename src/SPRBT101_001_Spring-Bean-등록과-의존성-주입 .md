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

  
