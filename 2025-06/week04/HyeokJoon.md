# AOP + 넥서스 패키지 배포를 선택하는 이유

## 1. AOP란?

**AOP (Aspect-Oriented Programming, 관점 지향 프로그래밍)**은 핵심 비즈니스 로직과 부가적인 공통 기능을 분리하여 모듈화할 수 있도록 도와주는 프로그래밍 패러다임이다.


### 1-1. Spring AOP의 핵심 개념

- **Join Point**: Advice가 적용될 수 있는 지점 (예: 메서드 호출)
- **Advice**: 공통 기능 코드 (Before, After, Around 등)
- **Pointcut**: Advice를 적용할 Join Point를 필터링하는 표현식
- **Aspect**: Advice와 Pointcut의 결합
- **Weaving**: 코드에 Aspect를 적용하는 과정


### 1-2. AOP의 넓은 의미
#### *관심사의 분리*
AOP는 Spring의 @Aspect만을 의미하는 것이 아니다. **횡단 관심사(Cross-cutting Concerns)**를 분리하는 모든 접근 방식이 AOP에 해당한다.

```java
// 전통적인 방식 - 비즈니스 로직과 부가 기능이 섞임
public class UserService {
    public String getUserInfo(Long id) {
        // 로깅 관심사
        log.info("사용자 조회 시작: {}", id);
        
        // 시간 변환 관심사
        LocalDateTime now = LocalDateTime.now();
        String formattedTime = now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"));
        
        // 핵심 비즈니스 로직
        User user = userRepository.findById(id);
        
        // 응답 포맷팅 관심사
        return String.format("[%s] 사용자: %s", formattedTime, user.getName());
    }
}

// AOP 적용 - 관심사 분리
public class UserService {
    @Loggable("사용자 조회")
    @TimeFormatted(pattern = "yyyy-MM-dd HH:mm:ss")
    public User getUserInfo(Long id) {
        // 순수한 비즈니스 로직만 남음
        return userRepository.findById(id);
    }
}
```
---

## 2. 어노테이션 vs Aspect: 역할 분담의 핵심

### 2-1. 어노테이션의 역할: 메타데이터 마킹

어노테이션은 **"여기에 특정 기능을 적용해달라"**는 **표시판** 역할

```java
// 단순 마킹 역할 - 실제 로직 없음
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Loggable {
    String value() default "";
    LogLevel level() default LogLevel.INFO;
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackTime {
    boolean includeArgs() default false;
}
```

### 2-2. Aspect의 역할: 실제 로직 구현

Aspect는 **"실제로 무엇을 할지"**를 구현하는 **실행자** 역할

```java
@Aspect
@Component
public class LoggingAspect {

    @Around("@annotation(loggable)")
    public Object logExecution(ProceedingJoinPoint joinPoint, Loggable loggable) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String logMessage = loggable.value().isEmpty() ? methodName : loggable.value();
        
        System.out.println("[" + loggable.level() + "] Start: " + logMessage);
        
        try {
            Object result = joinPoint.proceed();
            System.out.println("[" + loggable.level() + "] Success: " + logMessage);
            return result;
        } catch (Exception e) {
            System.out.println("[ERROR] Failed: " + logMessage + " - " + e.getMessage());
            throw e;
        }
    }
}

@Aspect
@Component
public class TimeTrackingAspect {

    @Around("@annotation(trackTime)")
    public Object trackTime(ProceedingJoinPoint joinPoint, TrackTime trackTime) throws Throwable {
        long start = System.currentTimeMillis();
        
        if (trackTime.includeArgs()) {
            System.out.println("Args: " + Arrays.toString(joinPoint.getArgs()));
        }
        
        Object result = joinPoint.proceed();
        long executionTime = System.currentTimeMillis() - start;
        
        System.out.println(joinPoint.getSignature().getName() + " executed in " + executionTime + "ms");
        return result;
    }
}
```

### 2-3. 동작 흐름: 어노테이션 → Aspect 연결

```java
@Service
public class UserService {
    
    @Loggable(value = "사용자 조회", level = LogLevel.DEBUG)
    @TrackTime(includeArgs = true)
    public User findUser(Long id) {
        return userRepository.findById(id);
    }
}
```

**실행 흐름:**
1. Spring이 `@Loggable`, `@TrackTime` 어노테이션 발견
2. 해당 어노테이션과 매칭되는 Aspect의 Pointcut 조건 확인
3. 조건에 맞는 Aspect의 Advice 실행
4. 실제 메서드 실행 전후로 부가 기능 적용

---

## 3. Pointcut 표현식: 어노테이션 vs 패키지 기반

### 3-1. 어노테이션 기반 Pointcut (권장)

```java
@Aspect
@Component
public class SecurityAspect {
    
    // 특정 어노테이션이 붙은 메서드에만 적용
    @Before("@annotation(com.example.annotation.RequireAuth)")
    public void checkAuthentication(JoinPoint joinPoint) {
        // 인증 체크 로직
    }
    
    // 어노테이션 파라미터 활용
    @Around("@annotation(cacheable)")
    public Object handleCache(ProceedingJoinPoint joinPoint, Cacheable cacheable) throws Throwable {
        String cacheKey = cacheable.key();
        // 캐시 로직
        return joinPoint.proceed();
    }
}
```

### 3-2. 표현식 기반 Pointcut

```java
@Aspect
@Component
public class MonitoringAspect {
    
    // 패키지 전체에 일괄 적용
    @Around("execution(* com.example.service.*.*(..))")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        // 모든 서비스 메서드 모니터링
        return joinPoint.proceed();
    }
    
    // 특정 클래스의 public 메서드에만 적용
    @Before("execution(public * com.example.controller.*Controller.*(..))")
    public void logControllerAccess(JoinPoint joinPoint) {
        // 컨트롤러 접근 로깅
    }
}
```

### 3-3. 어노테이션 방식의 장점

- **명시적 제어**: 개발자가 의도적으로 선택해서 적용
- **세밀한 설정**: 어노테이션 파라미터로 동작 커스터마이징
- **가독성**: 코드만 봐도 어떤 부가 기능이 적용되는지 명확
- **유지보수**: 불필요한 곳에 실수로 적용될 위험 없음

---

## 4. Spring AOP 동작 원리: 프록시 패턴

Spring AOP는 **프록시 패턴**을 기반으로 동작한다. 원본 객체를 직접 수정하지 않고, 원본 객체를 감싸는 프록시 객체를 생성하여 부가 기능을 제공하는 방식이다.

### 4-1. 프록시 생성 과정과 동작 원리

**런타임 프록시 생성**
Spring 컨테이너가 시작될 때, AOP가 적용된 빈들에 대해 프록시 객체를 동적으로 생성합니다. 이 프록시는 **원본 객체의 메서드 호출을 가로채어 Aspect의 로직을 실행한 후 원본 메서드를 호출**합니다.

```java
// 원본 서비스
@Service
public class UserService {
    @Loggable
    public User findUser(Long id) {
        return new User(id, "John");
    }
}

// Spring이 런타임에 생성하는 프록시 (개념적 표현)
public class UserService$$SpringProxy extends UserService {
    private LoggingAspect loggingAspect;
    
    @Override
    public User findUser(Long id) {
        // Before Advice
        loggingAspect.logBefore();
        
        try {
            // 실제 메서드 호출
            User result = super.findUser(id);
            
            // After Advice
            loggingAspect.logAfter();
            return result;
        } catch (Exception e) {
            // Exception Advice
            loggingAspect.logException(e);
            throw e;
        }
    }
}
```

### 4-2. 프록시 패턴의 한계점과 해결 방법

**내부 메서드 호출 문제**
프록시는 외부에서의 메서드 호출만 가로챌 수 있습니다. 같은 클래스 내부에서의 메서드 호출은 프록시를 거치지 않아 AOP가 적용되지 않습니다.

```java
@Service
public class UserService {
    
    @Loggable
    public User findUser(Long id) {
        // 내부 메서드 호출 - AOP 적용 안됨!
        return this.getUserFromCache(id);
    }
    
    @Loggable  // 이 어노테이션은 무시됨
    private User getUserFromCache(Long id) {
        return cache.get(id);
    }
}
```

**해결 방법:**
```java
@Service
public class UserService {
    
    @Autowired
    private UserService self;  // 자기 자신의 프록시 주입
    
    @Loggable
    public User findUser(Long id) {
        // 프록시를 통한 호출로 AOP 적용됨
        return self.getUserFromCache(id);
    }
    
    @Loggable
    public User getUserFromCache(Long id) {
        return cache.get(id);
    }
}
```

**성능 고려사항**
- 프록시 생성 비용: 애플리케이션 시작 시 추가 시간 소요
- 메서드 호출 오버헤드: 프록시를 거치는 추가 호출 비용
- 메모리 사용량: 원본 객체 + 프록시 객체

---

## 5. AOP의 다양한 구현 방식

### 5-1. 커스텀 유틸리티를 통한 AOP

- 시간 포맷팅(yyyy-MM-dd HH:mm:ss)
- 데이터 변환(반올림 로직)
- 유효성 검사

---

## 6. AOP + 넥서스 패키지 배포를 선택하는 이유

### 6-1. 관리 지점의 최소화

#### 문제 상황: 분산된 관리 지점
```java
// 프로젝트 A
public class UserServiceA {
    public User findUser(Long id) {
        log.info("[UserService] 사용자 조회 시작: {}", id);
        long start = System.currentTimeMillis();
        
        User user = userRepository.findById(id);
        
        long duration = System.currentTimeMillis() - start;
        log.info("[UserService] 사용자 조회 완료: {}ms", duration);
        return user;
    }
}

// 프로젝트 B
public class OrderServiceB {
    public Order findOrder(Long id) {
        logger.debug("주문 조회 시작: {}", id);  // 다른 로그 레벨
        long startTime = System.nanoTime();     // 다른 시간 측정 방식
        
        Order order = orderRepository.findById(id);
        
        long elapsed = System.nanoTime() - startTime;
        logger.debug("주문 조회 종료: {}ns", elapsed);  // 다른 단위
        return order;
    }
}
```

#### 해결: AOP + 넥서스를 통한 중앙 집중 관리
```java
// 공통 AOP 패키지 (넥서스 배포)
@Aspect
@Component
public class CommonLoggingAspect {
    @Around("@annotation(Loggable)")
    public Object log(ProceedingJoinPoint joinPoint, Loggable loggable) throws Throwable {
        // 통일된 로깅 정책
        return logAndProceed(joinPoint, loggable);
    }
}

// 모든 프로젝트에서 동일하게 사용
@Service
public class UserService {
    @Loggable("사용자 조회")
    @TrackTime
    public User findUser(Long id) {
        return userRepository.findById(id);  // 순수 비즈니스 로직만
    }
}
```

**관리 지점 최소화 효과:**
- **단일 진실 공급원**: 로깅 정책 변경 시 한 곳에서만 수정
- **일관성 보장**: 모든 프로젝트에서 동일한 방식으로 동작
- **유지보수 비용 절감**: N개 프로젝트 → 1개 패키지 관리

### 6-2. 확장성 측면의 이점

#### 수평적 확장성
```java
// 새로운 기능 추가 시
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedTrace {
    String serviceName() default "";
}

@Aspect
@Component
public class DistributedTracingAspect {
    @Around("@annotation(distributedTrace)")
    public Object addTracing(ProceedingJoinPoint joinPoint, DistributedTrace trace) throws Throwable {
        // 분산 추적 로직 추가
        return joinPoint.proceed();
    }
}
```

**한 번의 패키지 업데이트로 모든 서비스에 새 기능 적용**

#### 수직적 확장성
```java
// 기존 기능 확장
@Loggable(
    value = "사용자 조회",
    level = LogLevel.INFO,
    includeRequestId = true,    // 새 기능
    maskSensitiveData = true    // 새 기능
)
public User findUser(Long id) {
    return userRepository.findById(id);
}
```

### 6-3. 개발 생산성 향상

#### Before: 반복적인 보일러플레이트 코드
```java
public class OrderService {
    public Order processOrder(OrderRequest request) {
        // 매번 반복되는 코드들
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);
        
        log.info("주문 처리 시작: {}", request.getOrderId());
        long start = System.currentTimeMillis();
        
        try {
            validateRequest(request);
            Order order = createOrder(request);
            sendNotification(order);
            
            long duration = System.currentTimeMillis() - start;
            log.info("주문 처리 완료: {}ms", duration);
            return order;
            
        } catch (Exception e) {
            log.error("주문 처리 실패: {}", e.getMessage());
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

#### After: 핵심 로직에만 집중
```java
public class OrderService {
    @Loggable("주문 처리")
    @TrackTime
    @DistributedTrace(serviceName = "order-service")
    @ValidateParams(required = {"orderId", "customerId"})
    public Order processOrder(OrderRequest request) {
        // 순수한 비즈니스 로직만
        validateRequest(request);
        Order order = createOrder(request);
        sendNotification(order);
        return order;
    }
}
```

### 6-4. 운영 관점에서의 이점

#### 모니터링 및 관찰 가능성
```java
// 운영팀이 필요한 메트릭을 AOP로 자동 수집
@Aspect
@Component
public class OperationalAspect {
    
    @Around("@annotation(MonitorPerformance)")
    public Object collectMetrics(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        
        // 메트릭 수집
        Timer.Sample sample = Timer.start(meterRegistry);
        Counter.builder("method.calls").tag("method", methodName).register(meterRegistry).increment();
        
        try {
            Object result = joinPoint.proceed();
            Counter.builder("method.success").tag("method", methodName).register(meterRegistry).increment();
            return result;
        } catch (Exception e) {
            Counter.builder("method.error").tag("method", methodName).tag("error", e.getClass().getSimpleName())
                   .register(meterRegistry).increment();
            throw e;
        } finally {
            sample.stop(Timer.builder("method.duration").tag("method", methodName).register(meterRegistry));
        }
    }
}
```

#### 정책 변경의 신속한 적용
```java
// 보안 정책 변경 시
@Aspect
@Component
public class SecurityAspect {
    
    @Before("@annotation(RequireAuth)")
    public void checkAuth(JoinPoint joinPoint, RequireAuth auth) {
        // 중앙에서 보안 정책 변경 시 모든 서비스에 즉시 적용
        if (isNewSecurityPolicyEnabled()) {
            enforceStrictAuthentication();
        } else {
            enforceBasicAuthentication();
        }
    }
}
```

### 6-5. 비용 효율성

| 구분 | 전통적 방식 | AOP + 넥서스 방식 |
|------|-------------|-------------------|
| **개발 시간** | N개 프로젝트 × 개발시간 | 1개 패키지 개발 + 적용시간 |
| **유지보수** | N개 프로젝트 개별 수정 | 1개 패키지 수정 후 배포 |
| **테스트** | 각 프로젝트별 테스트 | 패키지 단위 테스트 |
| **일관성** | 수동 관리 (오류 가능성) | 자동 보장 |
| **확장성** | 선형적 증가 | 로그적 증가 |

---

## 7. 학습하며 생각했던 내용들

1**AOP의 범위**: 어디까지를 AOP라고 볼 수 있을까? 단순 유틸리티 함수와 AOP의 경계는?

2. **어노테이션 vs 표현식**: 언제 어노테이션 기반 Pointcut을 사용하고, 언제 표현식 기반을 사용해야 할까?

3. **프록시 패턴의 한계**: 내부 메서드 호출에서 AOP가 동작하지 않는 문제를 어떻게 해결할 수 있을까?

4. **성능 vs 기능**: AOP 적용으로 인한 성능 오버헤드와 개발 편의성 사이의 트레이드오프는?

    **성능 오버헤드** 
      - 프록시 생성: 애플리케이션 시작 시 ~10-20% 증가 
      - 메서드 호출: 호출당 ~1-5μs 추가 비용 
      - 메모리: 원본 객체 대비 ~20-30% 추가 사용 
    
    **개발 편의성 이득**
      - 개발 시간: 보일러플레이트 코드 ~70% 감소
      - 유지보수: 중앙 집중 관리로 ~80% 비용 절감
      - 일관성: 휴먼 에러 ~90% 감소

5. **패키지 배포 전략**: 공통 AOP 유틸을 패키지로 배포할 때의 버전 관리 및 호환성 유지 전략은?
    - 1.0.0 - 초기 버전 
    - 1.1.0 - 새 기능 추가 (하위 호환)
    - 1.1.1 - 버그 수정 
    - 2.0.0 - Breaking Change (하위 호환 불가)
---

## 참고 자료
- [Spring AOP 공식 문서](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop)
- [AspectJ 문서](https://www.eclipse.org/aspectj/doc/released/progguide/index.html)
- [Spring Boot AOP Starter](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-aop)