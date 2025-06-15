# CPU Scheduling Algorithm

스케줄링 알고리즘은 하나의 CPU가 여러 인스턴스 중 어떤 인스턴스를 실행할지 결정하는 규칙 또는 전략이다.

하나의 프로세스만 실행하는 단일 작업 환경에서는 CPU가 해당 프로세스를 전적으로 담당하지만, 여러 프로세스가 동시에 실행되어야 하는 환경에서는 CPU 자원을 두고 경쟁이 발생하게 된다.
결국 CPU는 한 번에 하나의 프로세스만 실행할 수 있으므로, 어떤 프로세스를 우선 실행할지를 정하는 것이 필요하다.
이러한 선택 과정을 일관된 기준으로 정립한 것이 **CPU 스케줄링 알고리즘**이다.

# 어떤 경우에 스케줄링이 필요한가?
"스케줄링"은 CPU 프로세스뿐 아니라, 시스템 전반에서 여러 인스턴스 중 하나를 선택해야 하는 모든 상황에서 활용될 수 있다.

예를 들어:
- 다중 인스턴스 서버 구성 시, 특정 도메인으로 들어온 HTTP 요청을 어떤 서버 인스턴스로 전달할지
- DB 커넥션 풀 크기가 1000개일 때, 추가 요청이 들어오면 어떻게 처리할지
- 작업마다 중요도(우선순위)가 다를 경우, 어떤 작업을 먼저 처리할지

이처럼 리소스를 누구에게 배정할 것인지 결정하는 문제는 모두 스케줄링 알고리즘으로 접근할 수 있다.

# 고가용성 (High Availability)
예를 들어, 사진 업로드를 위한 mediaUploader 서버를 만들었다고 가정해보자.
이 서버가 단일 인스턴스로만 운영된다면, 서버에 문제가 발생할 경우 사진 업로드 기능 전체가 중단될 수 있다.
이러한 상황을 방지하고자, 서비스가 장애 상황에서도 중단되지 않고 동작할 수 있도록 만드는 것이 **고가용성(HA)**의 핵심이다.

## 접근 방법
### 비즈니스 로직에서 예외 처리
업로드 실패 시 기본 이미지를 사용하고, 이후 재업로드를 시도하도록 로깅 및 재요청 로직을 구성한다.
- 장점: 서버 증설 없이도 동작
- 단점: 사용자 입장에서 원하지 않는 이미지 노출 가능, 재요청 실패 시 지속적 추적 필요

### 서비스 자체를 고가용성 구조로 구성
여러 대의 mediaUploader 서버를 운영하고, 문제가 발생한 서버는 제외하여 다른 인스턴스로 요청을 우회
- 장점: 사용자 경험 유지
- 단점: 서버 운영 비용 증가, 전체 인스턴스에 장애 발생 시 한계 존재


# HaProxy 사용 방법 (고가용성을 위한 로드밸런싱)
[HaProxy Document](https://docs.haproxy.org/)

설치 확인
```bash
haproxy -v

# 출력 예시
HAProxy version 2.8.5-1ubuntu3.2 2024/12/02 - https://haproxy.org/
....
```

HTTP 요청/응답 구조
```bash
# HTTP Request 예시
GET /serv/login.php?lang=en&profile=2 HTTP/1.1
Host: www.mydomain.com
User-Agent: my small browser
Accept: image/jpeg, image/gif
Accept: image/png

# HTTP Response 예시
HTTP/1.1 200 OK
Content-Length: 350
Content-Type: text/html

```
이 정보들을 활용해서 proxy서버단에서 로깅에 활용할 수 있다.
[loggin format](https://docs.haproxy.org/3.1/configuration.html#8.2)


# Haproxy 기본 설정 (haproxy.cfg)
```haporxy
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy                                         // haproxy 디렉토리를 루트처럼 사용
        stats socket /run/haproxy/admin.sock mode 660 level admin       // /run/haproxy/admin.sock 경로에 소켓 생성, 권한 660, admin 수준 명령 가능
        stats timeout 30s
        user haproxy                                                    // 실행 사용자 및 그룹을 haproxy로 제한
        group haproxy
        daemon                                                          // HAProxy를 백그라운드 데몬 프로세스로 실행

        # Default SSL material locations                                // SSL 인증서 및 CA 경로 기본값
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ...                                    // HAProxy의 기본 SSL 보안 설정
        ssl-default-bind-ciphersuites ....
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets     // no-tls-tickets: TLS 세션 재사용을 비활성화 → 보안성 향상 

defaults
        log     global
        mode    http                                    // HTTP 트래픽 처리 모드로 동작 (vs. tcp 모드)
        option  httplog                                 // HTTP 요청을 포맷에 맞춰 자세히 로깅
        option  dontlognull                             // 요청이 없는 keep-alive 연결은 로그 기록 생략
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http      // HTTP 오류 발생 시 사용자에게 보여줄 에러 페이지 파일 경로 지정
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
```

### **참고) 로그 레벨 설명**

| 레벨 이름     | 값 | 의미               |
| --------- | - | ---------------- |
| `emerg`   | 0 | 시스템이 사용 불가 상태    |
| `alert`   | 1 | 즉시 조치가 필요한 상황    |
| `crit`    | 2 | 심각한 상태           |
| `err`     | 3 | 오류 발생            |
| `warning` | 4 | 경고 메시지           |
| `notice`  | 5 | 주의할 만한 정상 상태 메시지 |
| `info`    | 6 | 일반 정보 메시지        |
| `debug`   | 7 | 디버깅용 상세 메시지      |

# admin socket을 통해 할 수 있는 일
### 상태 확인
- show stat → 백엔드/프론트엔드/서버의 트래픽, 연결 수, 상태 등 확인
- show info → 전체 HAProxy 인스턴스의 정보 (버전, Uptime 등)

### 서버 제어
- disable server <backend>/<server> → 특정 백엔드 서버를 일시적으로 트래픽에서 제외
- enable server <backend>/<server> → 다시 활성화
- 롤링 배포나 장애 대응 시 유용 (스크립트 연동 예: 배포 파이프라인에서 특정 서버를 잠시 빼고 다시 넣는 로직 구현)

### 동적 설정 변경 (제한적)
- 서버 상태를 바꾸거나 일시적으로 연결 제한 같은 설정을 적용


# 사용자 설정 예시
```
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    log-format "%t frontend=%ft client_ip=%ci request=\"%r\" status=%ST duration=%Tt"

frontend http-in
    bind *:80                                   // 모든 ip에서 80포트로 들어오는 요청을 수신한다.
    default_backend my_backend_servers          // *:80 요청이 들어오는 경우 요청을 보낼 서버

backend my_backend_servers                      // 요청을 보낼 백엔드 서버들 정의
    balance roundrobin                          // 스케줄링 알고리즘
    option tcp-check
    server server1 cluster1:8000 maxconn 32     
    server server2 cluster2:8000 maxconn 32
    server server3 cluster3:8000 maxconn 32
    
local0.*     /var/log/haproxy-access.log        // facility별 로그 저장 위치 지정
local1.notice /var/log/haproxy-notice.log

```

# HAProxy의 주요 스케줄링 방식
| 알고리즘                   | 설명                                                                                |
|------------------------|-----------------------------------------------------------------------------------|
| **roundrobin** (기본값)   | 각 서버에 순차적으로 요청 분배. 가장 단순하고 균등한 부하 분산. 서버 성능이 비슷할 때 적합.                            |
| **leastconn** (최소 연결 수) | 현재 연결 수가 가장 적은 서버에 요청 분배. 장기 연결이 많은 환경(DB, 채팅 서버 등)에 유리 : 부하가 비슷하지 않은 서버 간에도 효율적. |
| **source** (IP 해시 기반)  | 클라이언트 IP 기반 해시하여 항상 동일한 서버로 요청 전달. 세션 유지가 필요한 서비스에 적합. 단, 서버 수가 변경되면 분산 깨짐.                       |
| **uri / url_param** (URI 기반 해시)  | 요청 URI나 파라미터 값을 해시 기준으로 사용. 정적 리소스 캐싱에 유리.                                        |
| **hdr(name)** (HTTP 헤더 기반)  | HTTP 헤더 값을 해시 기준으로 사용. 사용자 식별 쿠키, 토큰 등을 기반으로 분배 가능.                               |


# 요약 정리
- 스케줄링 알고리즘은 시스템 리소스를 효율적으로 할당하는 핵심 원리이다.
- HAProxy는 서버 장애 시에도 요청을 안정적으로 분산할 수 있는 로드밸런서로, 고가용성을 실현할 수 있다.
- 다양한 로드밸런싱 알고리즘을 통해 서비스 특성에 맞는 트래픽 분산 전략을 선택할 수 있다.