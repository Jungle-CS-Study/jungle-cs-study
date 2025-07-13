# Java Spring에서 Queue Depth 문제 발생 시 대응 매뉴얼

Java Spring 기반 서버에서 `Queue Depth` 문제(요청 대기열 과부하) 발생 시, 원인 분석과 해결 방법을 정리한 가이드입니다.

---

## 1. Queue Depth란?

**Queue Depth**는 서버가 동시에 처리할 수 있는 요청을 초과하여 대기열에 요청이 쌓이는 상황을 의미합니다.

### 증상
- HTTP 503 (Service Unavailable)
- `acceptCount` 초과 시 요청 거절
- 응답 지연 / 타임아웃
- `java.util.concurrent.RejectedExecutionException`
- 로드밸런서에서 Health Check 실패

---

## 2. 문제 발생 원인

| 유형 | 설명 | 예시 |
|------|------|------|
|  스레드 수 부족 | `maxThreads` 이상 요청이 들어와 처리 불가 | 200개 스레드로 300개 동시 요청 처리 시도 |
|  요청 처리 지연 | I/O 또는 CPU 바운드 연산으로 처리 지연 | DB 쿼리 5초, 외부 API 호출 10초 |
|  큐 용량 부족 | `acceptCount` 제한 초과 | 대기열 100개 초과 시 신규 요청 drop |
|  DB / 외부 API 병목 | 커넥션 풀 부족 또는 외부 의존성 응답 지연 | HikariCP 10개, 동시 DB 요청 50개 |
|  GC 문제 | Full GC로 인한 애플리케이션 일시 정지 | Old Generation 메모리 부족 |

---

## 3. Spring Boot 환경에서 관련 설정 확인

```yaml
# application.yml
server:
  tomcat:
    max-threads: 200         # 동시에 처리 가능한 요청 수
    min-spare-threads: 10    # 최소 유지 스레드 수
    accept-count: 100        # 요청 대기열 최대 크기
    max-connections: 8192    # 최대 커넥션 수
    connection-timeout: 20000 # 커넥션 타임아웃 (ms)
```
**임계점**: 요청 수 > max-threads + accept-count일 경우 요청이 drop되거나 503 에러 발생

---

## 4. 진단 방법 
### 4-1. 실시간 스레드 상태 확인

```bash
# 1. Java 프로세스 PID 확인
jps -v

# 2. 스레드 덤프 생성
jstack [PID] > thread_dump.txt

# 3. 스레드 상태 분석
grep -A 5 -B 5 "BLOCKED\|WAITING" thread_dump.txt
```

**스레드 덤프 분석 포인트:**
```
"http-nio-8080-exec-1" #25 daemon prio=5 os_prio=0 tid=0x... nid=0x... waiting on condition
   java.lang.Thread.State: WAITING (parking)
   at sun.misc.Unsafe.park(Native Method)
   at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
   at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
```
- `BLOCKED`: 동기화 블록 대기 (데드락 가능성)
- `WAITING`: 다른 스레드의 작업 완료 대기
- `TIMED_WAITING`: 시간 제한이 있는 대기

### 4-2. Spring Actuator 를 사용해서 메트릭 수집

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,threaddump,httptrace
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
```

**주요 메트릭 확인:**
```bash
# 톰캣 스레드 상태
curl http://localhost:8080/actuator/metrics/tomcat.threads.busy
curl http://localhost:8080/actuator/metrics/tomcat.threads.current

# HTTP 요청 메트릭
curl http://localhost:8080/actuator/metrics/http.server.requests

# JVM 메트릭
curl http://localhost:8080/actuator/metrics/jvm.threads.states
```

### 4-3. 부하 테스트로 문제 재현

```java
// 테스트용 컨트롤러
@RestController
public class LoadTestController {
    
    @GetMapping("/slow")
    public String slowEndpoint() throws InterruptedException {
        Thread.sleep(5000); // 5초 지연 시뮬레이션
        return "Completed after 5 seconds";
    }
    
    @GetMapping("/cpu-intensive")
    public String cpuIntensive() {
        // CPU 집약적 작업 시뮬레이션
        long result = 0;
        for (int i = 0; i < 100_000_000; i++) {
            result += Math.sqrt(i);
        }
        return "CPU work completed: " + result;
    }
    
    @GetMapping("/db-heavy")
    public List<User> dbHeavy() {
        // DB 집약적 작업 시뮬레이션
        return userService.findAllWithComplexQuery();
    }
}
```

**Apache Bench로 부하 테스트:**
```bash
# 동시 100개 요청, 총 1000개 요청
ab -n 1000 -c 100 http://localhost:8080/slow

# 결과 분석 포인트
# - Time per request (평균 응답시간)
# - Failed requests (실패한 요청 수)
# - Requests per second (처리량)
```

### 4-4. 워크로드 유형 진단 (I/O vs CPU vs DB 바운드)

```java
// 성능 프로파일링을 통한 병목 지점 식별
@Component
public class PerformanceProfiler {
    
    private final MeterRegistry meterRegistry;
    
    @Around("@annotation(ProfilePerformance)")
    public Object profileMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        long startCpuTime = getThreadCpuTime();
        
        Object result = joinPoint.proceed();
        
        long totalTime = System.currentTimeMillis() - startTime;
        long cpuTime = getThreadCpuTime() - startCpuTime;
        long ioWaitTime = totalTime - cpuTime;
        
        double cpuRatio = (double) cpuTime / totalTime;
        String methodName = joinPoint.getSignature().getName();
        
        // 워크로드 유형 판단
        if (cpuRatio > 0.8) {
            log.warn("CPU 바운드 감지 - 메서드: {}, CPU 사용률: {}%", 
                    methodName, cpuRatio * 100);
            meterRegistry.counter("workload.cpu.bound", "method", methodName).increment();
        } else if (ioWaitTime > totalTime * 0.7) {
            log.warn("I/O 바운드 감지 - 메서드: {}, I/O 대기시간: {}ms", 
                    methodName, ioWaitTime);
            meterRegistry.counter("workload.io.bound", "method", methodName).increment();
        }
        
        return result;
    }
    
    private long getThreadCpuTime() {
        ThreadMXBean threadMX = ManagementFactory.getThreadMXBean();
        return threadMX.getCurrentThreadCpuTime() / 1_000_000; // 나노초를 밀리초로
    }
}
```

**워크로드 유형별 진단 기준:**

| 유형 | CPU 사용률 | I/O 대기시간 | DB 커넥션 사용률 | 대응 전략 |
|------|------------|--------------|------------------|----------|
| **CPU 바운드** | > 80% | < 20% | < 50% | 병렬 처리, 스케일 아웃 |
| **I/O 바운드** | < 30% | > 70% | < 50% | 비동기 처리, 커넥션 풀 증가 |
| **DB 바운드** | < 50% | > 50% | > 90% | 쿼리 최적화, 읽기 전용 DB 분리 |
| **메모리 바운드** | < 50% | < 30% | < 50% | GC 튜닝, 힙 크기 조정 |

```bash
# 실시간 워크로드 모니터링
# CPU 사용률 확인
top -p [PID]

# I/O 대기 확인
iostat -x 1

# DB 커넥션 상태 확인
curl http://localhost:8080/actuator/metrics/hikaricp.connections.active
```

### 4-5. 모니터링 대시보드 구성

```java
// 커스텀 메트릭 추가
@Component
public class QueueDepthMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Gauge queueDepthGauge;
    
    public QueueDepthMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.queueDepthGauge = Gauge.builder("queue.depth")
                .description("Current queue depth")
                .register(meterRegistry, this, QueueDepthMetrics::getCurrentQueueDepth);
    }
    
    private double getCurrentQueueDepth(QueueDepthMetrics self) {
        // 실제 큐 깊이 측정 로직
        ThreadPoolExecutor executor = (ThreadPoolExecutor) 
            ((ThreadPoolTaskExecutor) taskExecutor).getThreadPoolExecutor();
        return executor.getQueue().size();
    }
}
```
---

## 5. 해결 전략

### 5-1. 스레드 수, 큐 크기 조정

```yaml
server:
  tomcat:
    max-threads: 300       # 스레드 수 증가
    accept-count: 200      # 대기열 증가
    max-connections: 10000 # 최대 커넥션 수 증가
```

**적정 스레드 수 계산:**
```
CPU 집약적: 스레드 수 = CPU 코어 수 + 1
I/O 집약적: 스레드 수 = CPU 코어 수 × (1 + 대기시간/처리시간)

예시: 4코어, I/O 대기 80%, 처리 20%
스레드 수 = 4 × (1 + 0.8/0.2) = 4 × 5 = 20
```

**주의사항**: 물리 자원(메모리/CPU) 한계를 고려한 조정 필요

### 5-2. 비동기 처리 적용 (I/O 바운드 시)

```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean(name = "ioTaskExecutor")
    public Executor ioTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);  // I/O 작업은 더 많은 스레드
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("IO-");
        executor.initialize();
        return executor;
    }
}
```

```java
@Service
public class AsyncService {
    
    @Async("ioTaskExecutor")
    public CompletableFuture<String> processExternalApi() {
        return CompletableFuture.supplyAsync(() -> {
            // 외부 API 호출
            return restTemplate.getForObject("http://external-api.com/data", String.class);
        });
    }
    
    @Async("taskExecutor")
    public CompletableFuture<Void> processBackground(String data) {
        return CompletableFuture.runAsync(() -> {
            // 백그라운드 처리
            backgroundProcessor.process(data);
        });
    }
}
```

### 5-3. 병렬성 확보 (CPU 바운드 시)
cpu 작업에서는 .parallelStream 을 사용하도록 한다 -> parallelStream 은 ForkJoinPool.commonPool() 사용해서 HTTP 스레드와 독립적으로 수행된다.

하지만 과도한 병렬 처리시에는 컨텍스트 스위칭 비용이 증가하기 때문에 주의!

### 5-4. 커넥션 풀 최적화

```yaml
# HikariCP 설정
spring:
  datasource:
    hikari:
      maximum-pool-size: 30
      minimum-idle: 5
      connection-timeout: 3000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000  # 커넥션 누수 감지
      
# Redis 커넥션 풀
  redis:
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5
        max-wait: 3000ms
```

```java
// RestTemplate 커넥션 풀 설정
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory = 
            new HttpComponentsClientHttpRequestFactory();
        
        factory.setConnectTimeout(5000);
        factory.setReadTimeout(10000);
        
        // 커넥션 풀 설정
        PoolingHttpClientConnectionManager connectionManager = 
            new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(100);
        connectionManager.setDefaultMaxPerRoute(20);
        
        CloseableHttpClient httpClient = HttpClients.custom()
                .setConnectionManager(connectionManager)
                .build();
        
        factory.setHttpClient(httpClient);
        return new RestTemplate(factory);
    }
}
```

### 5-5. 요청 제한 (Rate Limiting)

```java
// Bucket4j를 활용한 Rate Limiting
@Component
public class RateLimitingFilter implements Filter {
    
    private final Map<String, Bucket> cache = new ConcurrentHashMap<>();
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        String clientIp = getClientIp((HttpServletRequest) request);
        Bucket bucket = resolveBucket(clientIp);
        
        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response);
        } else {
            HttpServletResponse httpResponse = (HttpServletResponse) response;
            httpResponse.setStatus(429); // Too Many Requests
            httpResponse.getWriter().write("Rate limit exceeded");
        }
    }
    
    private Bucket resolveBucket(String key) {
        return cache.computeIfAbsent(key, k -> {
            // 분당 100개 요청 제한
            Bandwidth limit = Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1)));
            return Bucket.builder()
                    .addLimit(limit)
                    .build();
        });
    }
}
```

---

## 6. 모니터링 및 알림 설정

### 6-1. Prometheus + Grafana 대시보드

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-app'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
```

**주요 모니터링 지표:**
- `tomcat_threads_busy_threads` / `tomcat_threads_config_max_threads` > 0.8
- `http_server_requests_seconds{quantile="0.95"}` > 5초
- `jvm_threads_states_threads{state="blocked"}` > 10개
- `hikaricp_connections_active` / `hikaricp_connections_max` > 0.9

### 6-2. 알림 규칙 설정

```yaml
# alerting_rules.yml
groups:
  - name: queue_depth_alerts
    rules:
      - alert: HighThreadUtilization
        expr: tomcat_threads_busy_threads / tomcat_threads_config_max_threads > 0.8
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High thread utilization detected"
          
      - alert: QueueDepthHigh
        expr: queue_depth > 50
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Queue depth is critically high"
```

---

## 7. 장애 대응 체크리스트

### 🚨 즉시 대응 (1-5분)
- [ ] 현재 스레드 사용률 확인 (`/actuator/metrics/tomcat.threads.busy`)
- [ ] 에러 로그 확인 (503, RejectedExecutionException)
- [ ] CPU/메모리 사용률 확인
- [ ] 로드밸런서에서 해당 인스턴스 제외 고려

### 🔍 원인 분석 (5-15분)
- [ ] 스레드 덤프 생성 및 분석
- [ ] 느린 쿼리/API 호출 확인
- [ ] GC 로그 분석
- [ ] 커넥션 풀 상태 확인

### 🛠️ 임시 조치 (15-30분)
- [ ] 스레드 수 증가 (재시작 필요)
- [ ] 타임아웃 설정 단축
- [ ] 불필요한 기능 일시 비활성화
- [ ] 스케일 아웃 (인스턴스 추가)

### 🔧 근본 해결 (30분 이후)
- [ ] 병목 지점 코드 최적화
- [ ] 비동기 처리 도입
- [ ] 캐싱 전략 적용
- [ ] 아키텍처 개선 (마이크로서비스, 이벤트 기반)

---

## 8. 정리

| 상황 | 즉시 대응 | 근본 해결 |
|------|-----------|-----------|
| **I/O 병목** | 타임아웃 단축, 커넥션 풀 증가 | 비동기 처리, 캐싱, 외부 API 최적화 |
| **CPU 병목** | 스케일 아웃, 불필요 기능 비활성화 | 알고리즘 최적화, 병렬 처리, 분산 처리 |
| **메모리 부족** | GC 튜닝, 힙 크기 증가 | 메모리 누수 수정, 객체 생성 최소화 |
| **DB 병목** | 커넥션 풀 증가, 쿼리 타임아웃 | 인덱스 최적화, 읽기 전용 DB 분리 |
| **설정 한계** | max-threads, accept-count 조정 | 아키텍처 개선, 마이크로서비스화 |

**핵심 원칙**: 모니터링 → 측정 → 분석 → 최적화 → 검증의 반복 사이클을 통한 지속적 개선
