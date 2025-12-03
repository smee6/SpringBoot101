# 비동기 처리
- 메소드 호출되면 별도 쓰레드에서 실행되어서 호출 즉시 값 반환하는 방식
- 응답속도, 병렬처리, 리소스 효율 등

## 스프링 부트에서는 
- Spring Boot는 비동기 처리를 AOP 기반 프록시로 @Async 메서드를 감싸서 ThreadPoolTaskExecutor에 제출하는 형식으로 처리
1. 클라이언트 → Controller → Service 메서드 호출
2. @Async 붙은 메서드 → 프록시가 감지
3. 지정된 Executor(쓰레드풀)에 작업 제출
4. 메인 쓰레드는 즉시 반환, 작업은 별도 쓰레드에서 실행

### @EnableAsync ... 무슨 말이냐면 코드로 보자
- 먼저 @EnableAsync로 Spring의 비동기 메서드 실행 기능 활성화 함. 이거 그냥 @Configuration 클래스 같은거
```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "taskExecutor") // 중요 여기임 밑에 설명있
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);       // 기본 쓰레드 수
        executor.setMaxPoolSize(10);       // 최대 쓰레드 수
        executor.setQueueCapacity(100);    // 대기 큐 크기
        executor.setThreadNamePrefix("Async-");
        executor.initialize();
        return executor;
    }
}
```
- 중요: Executor Bean 이름을 지정하면 @Async에서 "taskExecutor"로 참조 가능
#### 잠깐
- 왜 하는거냐면 Executor에 할당된 해당 스레드풀 사용하려고 하는거
- 기능들끼리 따로 Executor 격리해서 사용하면 서로 영향이 없으니까! 그리고 쓰레드풀별로도 따로 모니터링 가능
```java
@Bean(name = "ioExecutor")
public Executor ioExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(20);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(200);
    executor.setThreadNamePrefix("IO-");
    executor.initialize();
    return executor;
}

@Bean(name = "cpuExecutor")
public Executor cpuExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(4);
    executor.setMaxPoolSize(8);
    executor.setQueueCapacity(50);
    executor.setThreadNamePrefix("CPU-");
    executor.initialize();
    return executor;
}

@Async("ioExecutor")
public void fetchDataFromApi() { ... }

@Async("cpuExecutor")
public void processImage() { ... }
```

### @Async
- 거기에다가 아래처럼 @Async 하는거
- @Async는 순서 보장 안 함 → 순서 필요한 경우 CompletableFuture 조합 또는 메시지 큐 사용
- 이건 아마 @Service, @Component 등의 메서드 에서 구현되어 있겠
```java
@Service
public class NotificationService {

    @Async("taskExecutor")
    public void sendEmail(String to) {
        // 이메일 발송 로직
    }
}
```
- 아까 즉시 반환 한다고 그랬는데 void → fire-and-forget
- Future<T> / CompletableFuture<T> → 결과를 비동기로 받음 -> 일단 Future를 뱉는다

### Public 메서드로 꼭 만들어야 한다
- Spring의 @Async는 프록시 기반 → 프록시는 인터페이스 또는 클래스의 public 메서드만 감쌈
- private/protected 메서드나 같은 클래스 내부 호출은 프록시를 거치지 않음 → 비동기 적용 안 됨

### 반드시 쓰레드 풀 ThreadPoolExecutor 설정 등이 필요
- 기본 설정은 SimpleAsyncTaskExecutor → 근데 이것은 쓰레드 재사용 없음, 성능 저하
- 그러니 반드시 ThreadPoolTaskExecutor 설정이 필수적

### 비동기 예외처리 (이전 설명과 동일)
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() { ... }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async error in method: {}", method.getName(), ex);
        };
    }
}
```
- @Async 메서드에서 발생한 예외는 호출자에게 전달되지 않음
- 이유: 호출자 쓰레드와 실행 쓰레드가 다름
- CompletableFuture 반환 후 .exceptionally()로 처리
- AsyncUncaughtExceptionHandler 구현

### 비동기 많이 쓰는 환경에서는 쓰레드풀 모니터링이 필수적임
- Actuator, JMX, Micrometer로 pool size, active count 확인
- 왜냐면 QueueCapacity 초과 시 → 쓰레드풀 정책에 따라 예외 발생 또는 호출 쓰레드에서 실행 될 수도 있
