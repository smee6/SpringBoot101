# Heap과 Stack (JVM 메모리 구조)
Java 프로그램이 실행되면 JVM은 크게 Heap과 Stack 메모리 영역을 사용
- 힙 : 객체(인스턴스)가 저장되는 영역, 모든 스레드가 공유, new로 생성하면 여기에, GC가 관리 ... 클래스,인스턴스,필드,배열,동적 데이터, 좀 느림, 동기화 필요
- 스택 : 메서드 호출 시 생성되는 스레드별 실행 컨텍스트 저장 영역, 스레드마다 독립적, 지역변수,매개변수,리턴주소,메서드 종료시 제거됨, 빠르고 동기화 필요없음, 크기 제한있고 공유 X

### 멀티 스레딩환경에서 Visibility(가시성) — Java Memory Model(JMM)
- 멀티스레드 환경에서 Visibility는 "한 스레드가 변경한 값이 다른 스레드에서 언제 보이는가"를 의미
```java
class SharedData {
    boolean flag = false;

    void writer() {
        flag = true; // Thread-1
    }

    void reader() {
        while (!flag) {
            // Thread-2: flag 변경을 못 볼 수 있음
        }
        System.out.println("Flag changed!");
    }
}

원인: CPU 캐시와 JMM의 메모리 가시성 규칙
Thread-1이 flag를 변경해도 Thread-2의 CPU 캐시에 이전 값이 남아있을 수 있음
Heap의 최신 값이 즉시 반영되지 않음

<참고> Java Memory Model(JMM)

JVM은 스레드별로 변수 값을 CPU 캐시 또는 레지스터에 저장할 수 있음
변경 사항이 **메인 메모리(Heap)**에 즉시 반영되지 않으면 다른 스레드가 변경을 못 볼 수 있음
Visibility 문제는 동시성 버그의 주요 원인
```
### Visibility 이슈의 해결방법
```java
//1. volatile 키워드
//변수 읽기/쓰기 시 메인 메모리와 동기화
//모든 스레드가 항상 최신 값을 읽음
volatile boolean flag = false;
//단점: 원자성(atomicity) 보장 X → 복합 연산에는 synchronized 필요

//2. synchronized 블록
//진입 시 메인 메모리에서 최신 값 읽기
//종료 시 메인 메모리에 값 쓰기
synchronized void writer() { flag = true; }
synchronized void reader() { while (!flag) {} }

//3. java.util.concurrent 패키지
AtomicInteger, ConcurrentHashMap 등은 가시성과 원자성을 모두 보장
```
- 실무예시
```java
class TaskRunner {
    private volatile boolean running = true;

    public void run() {
        while (running) {
            // 작업 수행
        }
        System.out.println("작업 종료");
    }

    public void stop() {
        running = false; // 다른 스레드에서 즉시 반영됨
    }
}
// volatile 덕분에 stop() 호출 시 run() 루프가 즉시 종료됨
```
---  

# Excutor
- 인터페이스임. (public interface Executor ...)
- 스레드 실행을 직접 제어하지 않고, 작업 실행을 위임하기 위한 역할
```java
public interface Executor {
    void execute(Runnable command);
}
```
원래는 new Thread(()-> .... 어쩌고.start(); 이런식으로 해서 스레드를 만들었음  
근데 이건 스레드 생성비용도 크고, 개수 관리도 어렵고 재사용도 안됨  
- 해결: 스레드 생성과 관리를 Executor가 대신 처리하고 개발자는 작업을 그냥 Executor에 넘기기만 함  

### Executor 구현체
- Java에서는 Executor 인터페이스 기반으로 여러가지 구현체를 만들어 뒀음
- ExecutorService
```java
ExecutorService executor = Executors.newFixedThreadPool(5);

executor.execute(() -> {
    System.out.println(Thread.currentThread().getName() + " 작업 실행");
});

executor.shutdown();
```
- ScheduledExecutorService
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

scheduler.schedule(() -> {
    System.out.println("3초 후 실행");
}, 3, TimeUnit.SECONDS);
```

### 동작 흐름
개발자 → Runnable 작업 생성 → Executor에 전달  
Executor → 스레드 풀에서 빈 스레드 선택 → 작업 실행  
Executor → 스레드 재사용, 개수 관리, 종료 처리  

```java
import java.util.concurrent.*;

public class ExecutorExample {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            executor.submit(() -> {
                System.out.println("Task " + taskId + " 실행 by " + Thread.currentThread().getName());
            });
        }

        executor.shutdown();
    }
}

<출력 예시> = 스레드가 재사용됨 → 성능 효율 ↑
Task 1 실행 by pool-1-thread-1
Task 2 실행 by pool-1-thread-2
Task 3 실행 by pool-1-thread-3
Task 4 실행 by pool-1-thread-1
Task 5 실행 by pool-1-thread-2
```

# ThreadPool 스레드 풀
매번 new Thread()로 스레드를 만들면 생성 비용과 GC 부담이 크기 때문에, 스레드를 재사용하는 방식으로 성능을 최적화
1. 애플리케이션 시작 시 N개의 스레드를 미리 생성
2. 작업(Runnable/Callable)을 큐에 넣음
3. 풀 안의 빈 스레드가 큐에서 작업을 꺼내 실행
4. 작업 완료 후 스레드는 다시 풀로 돌아감
5. 애플리케이션 종료 시 모든 스레드 종료

### ThreadPoolExecutor 라는게 있다!
ThreadPoolExecutor  
 ├─ corePoolSize       : 기본 유지 스레드 수  
 ├─ maximumPoolSize    : 최대 스레드 수  
 ├─ keepAliveTime      : 유휴 스레드 유지 시간  
 ├─ workQueue          : 작업 대기 큐  
 ├─ threadFactory      : 스레드 생성 전략  
 ├─ handler            : 작업 거부 정책 (RejectedExecutionHandler)  

```java
// Executor와 유사 코드
// ! Executors.newFixedThreadPool(), newCachedThreadPool() 등은 사실 내부적으로 ThreadPoolExecutor를 생성
Executors.newFixedThreadPool(n) → 고정 개수
Executors.newCachedThreadPool() → 필요 시 스레드 생성, 유휴 시 제거
Executors.newSingleThreadExecutor() → 단일 스레드
Executors.newScheduledThreadPool(n) → 예약 실행

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ThreadPoolExample {
    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(3); // 스레드 3개

        for (int i = 1; i <= 5; i++) {
            int taskId = i;
            pool.submit(() -> {
                System.out.println("Task " + taskId + " 실행 by " + Thread.currentThread().getName());
            });
        }

        pool.shutdown();
    }
}

출력 예시

Task 1 실행 by pool-1-thread-1
Task 2 실행 by pool-1-thread-2
Task 3 실행 by pool-1-thread-3
Task 4 실행 by pool-1-thread-1
Task 5 실행 by pool-1-thread-2

➡ 스레드가 재사용됨
```

# CompletableFuture
- 비동기 작업을 처리하고, 그 결과를 조합하거나 후속 작업을 연결할 수 있는 클래스  
- Java 8에서 도입되었으며, Future의 한계를 개선
- .supplyAsync() 이걸로 일단 비동기 작업을 시작

### Call back cahin method가 있음
```java
각 콜백은 새로운 CompletableFuture를 반환
이전 단계가 완료되면 다음 단계 실행
```
.thenApply(fn) → 결과 변환  
.thenAccept(fn) → 결과 소비  
.thenRun(fn) → 결과 사용 안 하고 실행  
.thenCombine(cf, fn) → 두 Future 결과 합치기  
.allOf(cf...) → 여러 Future 모두 완료 시 실행  
.exceptionally(fn) → 예외 처리  

### <기존 Future의 한계>
```java
Future<String> future = executor.submit(() -> "Hello");
String result = future.get(); // 블로킹 (결과 나올 때까지 기다림) 스레드가 블로킹됨
```

### 그런데 CompletableFuture 에선 다르다!

```java
CompletableFuture의 장점
  
논블로킹: 결과가 준비되면 콜백 실행
체이닝: .thenApply(), .thenAccept()로 후속 작업 연결
병렬 조합: .thenCombine(), .allOf()로 여러 작업 결과 합치기
예외 처리: .exceptionally()로 에러 핸들링

import java.util.concurrent.CompletableFuture;

public class CompletableFutureExample {
    public static void main(String[] args) {
        CompletableFuture.supplyAsync(() -> {
            System.out.println("비동기 작업 실행: " + Thread.currentThread().getName());
            return "Hello";
        }).thenApply(result -> result + " World")
          .thenAccept(System.out::println)
          .exceptionally(ex -> {
              System.out.println("에러 발생: " + ex.getMessage());
              return null;
          });

        System.out.println("메인 스레드 계속 실행");
    }
}

출력 예시
메인 스레드 계속 실행
비동기 작업 실행: ForkJoinPool.commonPool-worker-1
Hello World
➡ 메인 스레드가 블로킹 없이 계속 실행됨
```

### 하나 더 <ThreadPool + CompletableFuture>

```java
//대규모 비동기 작업을 효율적으로 처리하고, 스레드 관리까지 최적화 가능 (스레드 재사용 + 큐 관리로 성능·안정성 확보
// + 비동기 작업 체이닝, 병렬 조합, 예외 처리 가능)
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture<String> api1 = CompletableFuture.supplyAsync(() -> callApi("API1"), pool);
CompletableFuture<String> api2 = CompletableFuture.supplyAsync(() -> callApi("API2"), pool);

CompletableFuture.allOf(api1, api2)
    .thenRun(() -> {
        try {
            System.out.println(api1.get() + " & " + api2.get());
        } catch (Exception e) {
            e.printStackTrace();
        }
    });

pool.shutdown();

//API 병렬 호출 후 결과 합치기
```
## 자바 패키지 구조와 계층
- 스프링은 좀 이런식으로 쓰는듯
com.example.project  
 ├─ controller   → 요청 처리 (API 엔드포인트)  
 ├─ service      → 비즈니스 로직  
 ├─ repository   → DB 접근  
 ├─ model        → 엔티티/DTO  
 └─ config       → 설정 클래스  

### 패키지 선언 (package)
- Java 클래스 파일의 첫 줄에 package를 선언해서 클래스가 속한 네임스페이스를 지정
- 패키지 이름은 회사 도메인 역순 + 프로젝트명 + 역할 ... 로 주로 만들긴 함 (예: com.mycompany.myapp.service)
```java
package com.example.project.service;

public class UserService { ... }
//➡ com.example.project.service라는 폴더 구조에 이 클래스가 위치해야 함
```
- Java는 패키지 = 폴더 구조
- package 선언은 클래스의 고유 주소 역할을 함
- 같은 이름의 클래스라도 패키지가 다르면 구분 가능

### 그래서 패키지가 다르면 import해서 불러와야 함
```java
import com.example.project.model.User;

public class UserService {
    private User user;
}
//➡ User 클래스가 다른 패키지에 있으므로 import 필요
```
- 같은 패키지에 있는 클래스는 import 없이 사용 가능
- java.lang 패키지(예: String, Integer)는 자동 import
- import java.util.*; → 해당 패키지의 모든 클래스 import (권장 X, 필요한 것만 import)
