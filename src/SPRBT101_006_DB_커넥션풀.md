### 커넥션풀
- 커넥션 풀은 데이터베이스에 대한 연결을 미리 생성해두고, 요청 시 재사용하는 기술
- CPU 코어 수에 따라 생성할 수 있는 커넥션 수가 제한
- 자바 스프링에서는 hikari를 기본으로 사용하고 아래처럼 설정

```java
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
```

## 커넥션풀 크기 설정
PostgreSQL 프로젝트에서 제공하는 공식적인 계산식
```
connections = ((core_count * 2) + effective_spindle_count)
위 식은 이상적인 처리량을 위해 필요한 커넥션.
예를 들어, 4코어 CPU를 가진 서버와 1개의 하드 디스크가 있는 경우, 커넥션 풀의 크기는 9로 설정할 수 있음
```

## 풀 잠금
단일 쓰레드가 많은 커넥션을 획득하면서 발생할 수 있는 "풀 잠금" 문제도 고려해야 함.
이 문제를 피하기 위해 다음 공식
```
pool size = Tn x (Cm - 1) + 1
여기서 Tn은 최대 스레드 수, Cm은 단일 스레드가 보유할 수 있는 최대 커넥션 수를 나타냄
```
을 활용이 가능하다.

### 커넥션 풀 사이즈에 관한 가이드
[You want a small pool, saturated with threads waiting for connections](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)

