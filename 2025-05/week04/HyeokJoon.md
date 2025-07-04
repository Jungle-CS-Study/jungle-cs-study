# HTTP
서버 간 통신을 위한 프로토콜로 웹 페이지, 텍스트 데이터, 이미지, 동영상 등 다양한 컨텐츠를 전송하는 표준 방법이다.

---
# HTTP의 버전
 
HTTP도 통신 프로토콜이기 때문에 점차 다양한 문제를 해결하고, 성능을 향상시키는 방향으로 개발되어왔다.

## HTTP 1.1

기존 HTTP 1.0의 문제점으로 요청마다 새로운 connection을 만들어야 했다.  
요청이 많은 경우 지속적인 3-way handshaking은 부담이 되었다.

[해결 방법]
- Keep-Alive 방식을 통해 timeout기간동안 연결을 유지할 수 있도록 변경

[Keep-Alive 방식의 문제]

Keep-Alive 방식은 TCP 연결을 재사용하지만, 여러 요청이 순차적으로 처리되기 때문에 앞선 요청이 지연되면 뒤의 요청이 블로킹되는 'Head-of-Line Blocking' 문제가 여전히 존재했다. 

[해결 방법]
- pipeline 방식을 사용하여 하나의 커넥션에서 여러 요청을 보낼 때, 응답을 기다리지 않고 순차적으로 요청을 보낸 후, 응답을 순차적으로 기다리는 방식을 사용하여 개선
---
## HTTP 2
HTTP 1.1은 결국 순차 처리방식을 사용하기 때문에(순차적으로 요청 전송, 순차적으로 응답 수신) pipeline을 사용한다고 해도 응답이 늦어지는 경우 전체가 컨넥션이 늦어지는 Head-of-line Blocking 문제가 있었다.

또한, 많은 요청이 들어오는 경우 동일한 요청 헤더가 반복적으로 사용되어 네트워크 대역폭을 잡아먹었다.

[해결 방법]
- Multiplexing: 하나의 커넥션이 여러 stream을 가질 수 있도록 하고, 이 stream 들의 순서관계를 없앰으로 여러 요청을 동시 처리가 가능하도록 변경하였다.
- 헤더 압축: HPACK 압축 방식을 통해 헤더의 동일한 부분을 전송하지 않고 변경점만 전송하도록 하여 네트워크 대역폭을 절약하였다.

> HTTP/1.1의 Pipeline은 요청을 연속적으로 보내는 것이지만, 응답은 순서대로 돌아와야 하므로 순서 문제가 있다. 반면 HTTP/2의 Multiplexing은 하나의 TCP 연결에서 여러 Stream을 독립적으로 처리하여 요청/응답이 섞여도 문제가 발생하지 않는다.

[추가 기능]
- 서버 푸시 : 클라이언트의 요청이 없어도 서버에서 리소스를 전달할 수 있다.

---
## HTTP 3
HTTP/2 까지는 기본적으로 TCP 프로토콜 위에서 동작하기 때문에 하나의 커넥션에서 TCP의 신뢰성 로직이 동작하게 된다. 따라서 패킷 손실이 발생했을 때는 커넥션에 포함된 모든 stream이 정지하고 손실된 패킷을 복구하는 과정을 가진다.

HTTP/2의 서버푸시의 문제점으로 서버가 클라이언트가 이미 로드한 리소스를 거의 알지 못하고 동일한 리소스를 여러 번 전송하기 때문에 대역폭 낭비가 빈번하게 발생하며, 푸시되는 리소스가 요청된 리소스와 대역폭을 놓고 경쟁하는 경우 속도가 느려지는 문제가 발생했다.

[해결 방법]
- QUIC 프로토콜 : UDP를 기반으로 하고, 각각의 stream에서 신뢰성 로직을 적용해서 TCP의 한계를 해결했다.
- 서버 푸시 기능 삭제

> QUIC은 각 Stream 단위로 패킷 손실 복원을 처리하므로, 일부 패킷 손실로 전체 연결이 멈추는 문제(TCP의 HOL Blocking)를 해결했다.
---


### HTTP 프로토콜별 특징

| 항목           | HTTP/1.1                              | HTTP/2                                          | HTTP/3 (QUIC 기반)                                       |
|----------------|---------------------------------------|------------------------------------------------|--------------------------------------------------------|
| 출시 연도       | 1997                                  | 2015                                           | 2022 (IETF 표준 채택)                                      |
| 전송 방식       | TCP                                   | TCP (멀티플렉싱 지원)                             | **UDP** (QUIC: Quick UDP Internet Connections 프로토콜 사용) |
| 연결 구조       | 요청당 한 연결 (Keep-Alive 필요)          | 하나의 TCP 연결에서 멀티플렉싱 지원                  | 하나의 UDP 연결에서 멀티플렉싱 지원                                  |
| 요청 처리 방식  | **순차 처리** (HOL Blocking 발생)          | **동시 처리 (Multiplexing)** (HOL Blocking 해결)    | 동시 처리 + HOL Blocking 해결 (TCP 한계 완전 제거)                 |
| 헤더 압축       | 없음 (매번 전체 전송)                      | 있음 (HPACK)                                     | 있음 (QPACK)                                             |
| TLS 의무화      | 선택 (HTTP 또는 HTTPS)                      | 선택 (HTTPS 권장, 브라우저는 HTTPS만 지원)            | **필수 (암호화 강제)**                                        |
| 서버 Push       | 없음                                    | 있음                                           | 없음 (제거됨)                                               |
| 전송 효율성      | 낮음 (다중 요청 처리 제한, 헤더 중복)          | 좋음 (멀티플렉싱, 헤더 압축)                          | **매우 좋음** (멀티플렉싱 + 0-RTT + 패킷 손실 복원)                   |
| 모바일/고손실 네트워크 | 약함                                   | 강함                                           | **매우 강함 (패킷 손실에도 강인함)**                                |

---


## 언제 어떤 HTTP 버전을 선택해야 할까?

| 상황                                 | HTTP/1.1                                     | HTTP/2                      | HTTP/3                                                       |
|------------------------------------|----------------------------------------------|-----------------------------|--------------------------------------------------------------|
| 단일 대용량 파일 다운로드 (예: 영상)   | 충분 (큰 차이 없음)                                 | 큰 차이 없음 (멀티플렉싱 효과 제한적) | 큰 차이 없음                                                      |
| 작은 요청/응답 다수 (썸네일, API)    | <span style="color:#DD3311">**병목 발생**</span> | 매우 유리                     | <span style="color:#1166EE">**가장 유리**</span> (멀티플렉싱 + 손실 복원) |
| 모바일/불안정 네트워크 (4G/5G, 위성) | 불리함                                          | 일부 개선                     | <span style="color:#1166EE">**압도적 이점**</span> (QUIC, 패킷 손실 복원)                                  |
| 브라우저 요청 (웹사이트, API)        | 지원됨                                          | 권장                           | 브라우저 지원 점점 확대 중                                              |
| 사내 시스템/단일 연결 API           | 충분 (간단함)                                     | 필요할 수도 있음 (헤더 크기 고려)       | 필요성 낮음 (복잡성 증가)                                              |

---

## HTTP/2, HTTP/3 도입 전 고려사항

| 항목                     | HTTP/2/3 도입 시 주의 사항                                                                               |
|--------------------------|---------------------------------------------------------------------------------------------------|
| TLS 필요성                | HTTP/2/3는 대부분 TLS 필요 (HTTP/3는 필수)                                                                 |
| 서버/로드밸런서 지원 여부  | 서버, CDN, LB(Cloudflare, ALB 등)의 HTTP/2/3 지원 확인 필요<br/>(클라이언트-LB는 HTTP/2, LB-백엔드는 HTTP/1.1일 수도 있음) |
| Idle Timeout 관리          | 연결 유지로 자원 고갈 방지 (Timeout, 커넥션 관리 필요)                                                              |
| 클라이언트 지원           | HTTP/3는 일부 클라이언트에서만 지원 (브라우저는 최신 버전 대부분 지원)                                                       |
| 순서 보장 문제            | HTTP/2/3는 요청 순서 보장 X → 클라이언트/서버 로직 주의 필요 (업데이트 → 조회 등의 순서 의존성이 필요한 로직에서 클라이언트, 서버단에서 관리해야한다.)     |

---
## 마무리

HTTP는 웹의 근간이 되는 중요한 프로토콜이며, 각 버전은 성능과 네트워크 환경을 고려해 발전해왔다.
- HTTP/1.1은 여전히 단순한 요청 처리나 소규모 시스템에 적합 
- HTTP/2는 대규모 API 서버, 웹사이트에서의 효율적인 요청 처리를 위해 추천 
- HTTP/3는 최신 네트워크 환경(모바일, 무선, 위성 등)과 초저지연 요구 사항에 최적화된 선택

시스템의 요구사항과 네트워크 환경, 클라이언트 지원 상태를 고려하여 적절한 HTTP 버전을 선택하는 것이 중요하다. \
앞으로도 HTTP 프로토콜의 발전을 이해하고 적절히 적용하여 효율적이고 안전한 네트워크 서비스를 설계해 봐야 한다.