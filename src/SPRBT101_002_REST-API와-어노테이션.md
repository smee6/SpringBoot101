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

