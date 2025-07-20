# 나는 왜 쓰레드가 필요한가? 

```other
상황 1: 순차적 크롤링 (쓰레드 없음)
온오프믹스 크롤링 (3초) → 이벤터스 크롤링 (4초) → 데브이벤트 크롤링 (2초)
총 소요시간: 9초

상황 2: 동시 크롤링 (쓰레드 사용)
온오프믹스 │ 이벤터스 │ 데브이벤트 (모두 동시에)
총 소요시간: 4초 (가장 오래 걸리는 작업 기준)
```

- 크롤링 작업에서 가장 큰 병목은 보통 네트워크 대기 시간 
- 예를 들어 3개의 사이트를 순차적으로 크롤링하면, 각 요청이 끝날 때까지 기다려야 하므로 시간이 누적된다.
- 그래서 ‘동시에 여러 일을 처리하는 구조’, 즉 쓰레드가 필요하다. 

# 쓰레드의 핵심 개념

- **쓰레드(Thread)** = 프로그램 실행의 최소 단위
- **프로세스(Process)** = 하나의 독립된 프로그램 (예: 크롬 브라우저)
- 하나의 프로세스 안에 여러 쓰레드가 있을 수 있고,

    → 이 덕분에 **크롬에서 탭 여러 개를 동시에 열 수 있는 것처럼**,

    → **여러 페이지를 동시에 크롤링**할 수 있음

# 내 코드의 예상 병목 (가설) 

### 네트워크 I/O (추정 2-3초)

```python
# 예상했던 주요 병목 지점
def get_events_from_page(page_num):
    # ... 설정 코드 ...
    
    # 🚫 예상 병목: HTTP 요청이 2-3초씩 걸릴 것
    response = requests.get(api_url, headers=headers)  # 예상 시간: 2-3초
    
    # ... 처리 코드 ...
```

- 온오프믹스는 DB 조회가 많은 복잡한 페이지
- 일반적인 웹 크롤링에서 네트워크가 주 병목
- 서울 ↔ 서버 간 물리적 거리와 처리 시간

### 순차 처리 구조

```python
# 예상했던 비효율적 구조
for page in range(1, 4):  # 1, 2, 3페이지
    events, total_cnt = get_events_from_page(page)  # 각각 2-3초 대기
    # 페이지 1 완료 → 페이지 2 시작 → 페이지 3 시작
    
# 예상 총 시간: 3페이지 × 3초 = 9초
```

### time.sleep 낭비

```python
if events:
    all_events.extend(events)
    time.sleep(1)  # 예상: 2초 낭비 (2번 실행)
```

### Claude API

```python
# Claude는 1개만 분석하니까 큰 문제 없을 것이라 예상
parsed_event = analyze_event_with_claude(first_event_html)  # 예상: 1-2초
```



# 성능 측정 후

### 실제 측정 결과

```python
# 실제 성능 측정 결과
실제_측정_결과 = {
    "네트워크_요청": 1.17,      # 3회, 평균 0.39초 (예상보다 7배 빠름!)
    "HTML_파싱": 0.105,        # 3회, 평균 0.035초
    "Claude_분석": 2.58,       # 1회 (예상보다 약간 느림)
    "time_sleep": 2.00,        # 2회 × 1초 (예상과 동일)
    "총_실제_시간": 5.87       # 예상 13초보다 2.2배 빨랐음
}
```



# 실제 병목 우선순위

### 1. Claude API (2.58초, 43.9%)

```python
def analyze_event_with_claude(event_html):
    # 🚫 실제 최대 병목: Claude API 호출
    claude_start = time.time()
    
    message = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=500,
        messages=[{"role": "user", "content": prompt}]
    )
    # ↑ 이 부분에서 2.58초 소요 (전체의 44%)
    
    claude_time = time.time() - claude_start
    print(f"📊 Claude 총 처리 시간: {claude_time:.2f}초")
```

- Claude API가 예상보다 느림 (2초 예상 → 2.58초 실제)
- 전체 시간의 거의 절반을 차지
- 1개만 분석하는데도 이 정도면, 여러 개 분석 시 더 큰 병목

### 2. time.sleep 대기 (2.00초, 34.1%)

```python
# 여전히 큰 낭비였음
for page in range(1, 4):
    events, total_cnt = get_events_from_page(page)
    
    if events:
        all_events.extend(events)
        if page < 3:  # 마지막 페이지가 아니면
            time.sleep(1)  # 🚫 1초씩 2번 = 2초 완전 낭비
```

- 예상과 동일하게 2초 낭비
- 전체 시간의 1/3을 차지하는 순수 낭비
- 네트워크 요청이 빨라서 상대적으로 더 큰 비중

### 3. 네트워크 요청 (1.17초, 19.9%)

```python
# 예상과 달리 매우 빨랐음
def get_events_from_page(page_num):
    network_start = time.time()
    response = requests.get(api_url, headers=headers)
    network_time = time.time() - network_start
    
    # 실제 결과:
    # 페이지 1: 0.38초 (예상: 2-3초)
    # 페이지 2: 0.38초 (예상: 2-3초)  
    # 페이지 3: 0.41초 (예상: 2-3초)
```

- 온오프믹스 서버가 예상보다 7-8배 빨랐음
- API 응답이 최적화되어 있음
- 여전히 순차 처리로 인한 시간 낭비는 있음

### 4. HTML 파싱 (0.105초, 1.8%)

```python
# 완전히 무시할 수 있는 수준
parsing_start = time.time()
soup = BeautifulSoup(html_content, 'html.parser')
events = soup.select('article.event_area.event_main')
parsing_time = time.time() - parsing_start

# 실제 결과: 평균 0.035초 (전체의 1.8%)
```

# 개선 전략

1. Claude 최적화(지금은 1개 행사만 분석하고 있지만, 비슷한 소요 시간으로 더 많은 데이터 분석할 수 있도록)

→ 하지만 어려워 보여서 일단 지금은 Pass!  

1. time.sleep 제거 (2초 단축, 즉시 효과)
2. 페이지 크롤링 병렬화 → 파이썬 라이브러리 사용

# 시도한 방법

| 항목         | Before         | After                                                                              |
| ---------- | -------------- | ---------------------------------------------------------------------------------- |
| Import     | 기본 라이브러리       | `concurrent.futures` 라이브러리 사용 `<br/>ThreadPoolExecutor(클래스)`, `as_completed(함수)`추가 |
| 실행 방식      | `for` 루프 순차 처리 | `ThreadPoolExecutor` 동시 처리                                                         |
| time.sleep | 2초 강제 대기       | 완전 제거                                                                              |

# 라이브러리 `concurrent.futures`  ([공식 문서](https://docs.python.org/ko/3/library/concurrent.futures.html))

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

# 1. 일꾼들 모으기 (3명)
with ThreadPoolExecutor(max_workers=3) as executor:
    
    # 2. 일 시키기
    futures = [executor.submit(함수이름, 인자) for 인자 in [1,2,3]]
    
    # 3. 끝나는 대로 결과 받기
    for future in as_completed(futures):
        결과 = future.result()
        print(결과)
```

### 기존 방식

```python
def 페이지_가져오기(페이지번호):
    print(f"{페이지번호}페이지 가져오는 중...")
    time.sleep(1)  # 1초 걸림
    return f"{페이지번호}페이지 완료!"

# 혼자서 하나씩
결과1 = 페이지_가져오기(1)  # 1초
결과2 = 페이지_가져오기(2)  # 1초  
결과3 = 페이지_가져오기(3)  # 1초
# 총 3초
```

### 새로운 방식 (일꾼들과) 

```python
from concurrent.futures import ThreadPoolExecutor

# 일꾼 3명과 함께!
with ThreadPoolExecutor(max_workers=3) as 일꾼들:
    # 각각 다른 일꾼에게 일 맡기기
    작업1 = 일꾼들.submit(페이지_가져오기, 1)
    작업2 = 일꾼들.submit(페이지_가져오기, 2) 
    작업3 = 일꾼들.submit(페이지_가져오기, 3)
    
    # 결과 받기
    결과1 = 작업1.result()
    결과2 = 작업2.result()
    결과3 = 작업3.result()
# 총 1초 (동시에 했으니까!)
```

### as_completed = "끝나는 대로 알려줘"

```python
# 일꾼들이 끝나는 순서가 다를 수 있어요
작업들 =이[작업1, 작업2, 작업3]

# 순서대로 기다리면 비효율적
for 작업 in 작업들:
    결과 = 작업.result()  # 1번이 늦으면 2,3번도 기다려야 함

# as_completed: 끝나는 대로 바로 처리
for 완료된작업 in as_completed(작업들):
    결과 = 완료된작업.result()  # 끝난 것부터 바로!
    print(f"완료: {결과}")
```

# 결과 

![giphy (3).gif](https://resv2.craft.do/user/full/ee20bc6b-d50e-9e91-21cc-c811d3afb760/doc/EFF2C2B9-AB4F-4B9B-9D37-66A677CECBC8/55A49AE7-D293-47AC-8F2F-D157113AC65A_2/ILHsdy3hgzaZEIp42rx97AGjH472YiKnGp30Y3BqOWAz/giphy%203.gif)

|       | Before | After             |
| ----- | ------ | ----------------- |
| 실행 시간 | 5.87초  | 3.03초 (1.9배 빨라짐)  |

- sleep이 진짜 주범이었다! ( 약 2초 절약 ) 
- 크롤링 페이지 병렬 처리(멀티쓰레딩)도 효과가 있었다. ( 0.6초 절약 ) 
- 만약에 
    - 크롤링 페이지가 늘어나면 늘어날 수록 효과는? WoW 
    - 지금은 Claude가 1개 행사만 분석하고 있는데, 수집해온 60개 모든 행사를 분석한다면? WoW

# 부록 

- 지금은 병렬화의 단위가 작고, CPU 연산이 거의 없기 때문에 ThreadPoolExecutor로 충분하지만 향후 크롤링 대상이 수십 개 이상으로 늘어나면? 쓰레드는 언제(?)까지 어떨 때까지 유효한가?  
- Claude 병렬 분석도 ThreadPoolExecutor로 실험해보기
- 그런데요. 저 고백할 거 있습니다. 저…FE 도전합니다. 2년만 기다려주세여 💜 



