# AOP 적용했는데 코드가 더 늘어났습니다

# AOP가 무엇인가 - "코드 중복을 없애는 똑똑한 방법"

```bash
📍 AOP 한 줄 정의
"모든 함수에 공통으로 필요한 코드를 한 곳에 모아서 자동으로 적용하는 기법"

📍 간단한 Before/After
Before: 로깅 코드를 10개 함수에 복사-붙여넣기
After: @log 데코레이터 하나로 해결

📍 핵심 개념 2가지만
- 핵심 관심사: 함수가 진짜 하려는 일
- 횡단 관심사: 모든 함수에 공통으로 필요한 일
```

# 실제 개발할 때 겪는 문제

```bash
📅 1주차: "간단한 크롤러 만들어주세요"
def crawl_data():
    return requests.get("...")

😊 "쉽네!"

📅 2주차: 팀장 "로그 좀 남겨주세요"  
def crawl_data():
    print("시작")
    result = requests.get("...")
    print("완료")
    return result

😐 "음... 괜찮네"

📅 3주차: 팀장 "에러 처리도 해주세요"
def crawl_data():
    print("시작")
    try:
        result = requests.get("...")
        print("완료")
        return result
    except Exception as e:
        print(f"에러: {e}")
        return None

😕 "코드가 좀 복잡해지네..."

📅 4주차: 팀장 "성능 측정도 추가해주세요"
📅 5주차: 팀장 "재시도 로직도 넣어주세요"  
📅 6주차: 팀장 "캐싱도 해주세요"

😱 "20줄 중에 핵심 로직은 1줄뿐이야!"

📅 7주차: 팀장 "크롤러 10개 더 만들어주세요"

🤯 "이 코드를 10번 복사붙여넣기?!"

📅 8주차: 버그 발견
👨‍💼 "10개 파일 다 고쳐야 해요..."

😭 "지옥이다..."
```

# 이때 AOP를 알았다면?

```python
# 8주간의 고생이 이렇게 간단해짐
@log @retry @performance @cache
def crawl_data():
    return requests.get("...")

# 새 크롤러 추가도 2줄로 끝
@log @retry @performance @cache  
def crawl_new_site():
    return requests.get("...")
```

# 내 프로젝트에 적용한다면? 

```bash
🎯 데브캘 프로젝트 목표
"개발자 행사 정보를 여러 사이트에서 수집해서 개인 맞춤형 캘린더로 제공"

✅ 개발 흐름
1단계(현재): 온오프믹스 1페이지만 (20개 행사)
   ↓
2단계: 전체 페이지 크롤링 (20+개 행사, Playwright 도입)
   ↓ 
3단계: Claude로 자동 파싱
   ↓    
4단계: 이벤터스 추가
   ↓   
5단계: 링크드인 추가
   ↓   
마지지막: 여러 사이트 통합 플랫폼 완성

🚨 예상되는 문제들 
현재: 온오프믹스 크롤러 (requests만)
1. Playwright 도입 (😱 핵심 로직 1줄, 부가 코드 N줄!)
2. Claude API 추가 (😱 총 N줄 중 핵심 로직 여전히 1줄!)
3. 사이트 추가 (😱 동일한 코드에서 URL 3개만 다름!)
4. Playwright 에러 발견 (😱 버그 수정 지옥에 빠진다)
→ 개 파일 모두 수정해야 함
→ 하나라도 빠뜨리면 버그 남아있음
→ 일관성 깨짐
😭 "같은 버그를 3번 고쳐야 해요..."
```

# AOP 적용 시나리오

## 현재 코드 (requests만 사용)

```python
# onoffmix_crawler.py (현재 코드)
import requests
from bs4 import BeautifulSoup

# 온오프믹스 AI 관련 행사 검색 URL
url = "https://onoffmix.com/event/main?s=%EC%9D%B8%EA%B3%B5%EC%A7%80%EB%8A%A5%20ai%20chatgpt%20%EC%B1%97gpt"

# 브라우저인 척 하기 위한 헤더 설정
headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
}

# HTTP GET 요청으로 HTML 가져오기
response = requests.get(url, headers=headers)

# HTML을 BeautifulSoup으로 파싱 가능한 객체로 변환
soup = BeautifulSoup(response.text, 'html.parser')

# CSS 선택자로 모든 행사 카드 찾기
events = soup.select('article.event_area.event_main')

print(f"총 {len(events)}개의 행사를 찾았습니다!")

# 추출된 행사 정보를 저장할 리스트
all_events = []

# 각 행사 카드에서 정보 추출
for i, event in enumerate(events):
    try:
        # 행사 제목 추출
        title = event.find('h5', class_='title ellipsis').text.strip()
        
        # 가격 정보 추출 (무료/유료)
        payment = event.find('span', class_='payment_type').text.strip()
        
        # 행사 날짜 추출
        date = event.find('div', class_='date').text.strip()
        
        # 행사 장소/지역 추출
        place = event.find('span', class_='place').text.strip()
        
        # 상세 페이지 링크 추출 (상대 경로)
        link = event.find('a')['href']
        
        # 추출된 정보를 딕셔너리로 구조화
        event_data = {
            "id": i + 1,
            "title": title,
            "payment": payment,
            "date": date,
            "place": place,
            "link": "https://onoffmix.com" + link  # 절대 경로로 변환
        }
        
        # 리스트에 행사 정보 추가
        all_events.append(event_data)
        
        # 콘솔에 간단한 정보 출력
        print(f"{i+1}. {title} | {payment} | {place}")
        
    except Exception as e:
        # 특정 행사 파싱 실패 시 에러 로그
        print(f"행사 {i+1} 추출 실패: {e}")

# 최종 결과 출력
print(f"\n총 {len(all_events)}개 행사 추출 완료!")
```

**결과:**

- 약 20개 행사 (1페이지만)

# Playwright 도입 후 (AOP 없음)

```python
# onoffmix_playwright_crawler.py (Playwright 도입 후)
import requests
from bs4 import BeautifulSoup
from playwright.sync_api import sync_playwright

def crawl_with_playwright():
    # 온오프믹스 AI 관련 행사 검색 URL
    url = "https://onoffmix.com/event/main?s=%EC%9D%B8%EA%B3%B5%EC%A7%80%EB%8A%A5%20ai%20chatgpt%20%EC%B1%97gpt"
    
    # Playwright 브라우저 설정 시작
    with sync_playwright() as p:
        # 크로미움 브라우저 실행 (화면 안 보이게)
        browser = p.chromium.launch(headless=True)
        
        # 새 페이지 생성
        page = browser.new_page()
        
        # 브라우저 헤더 설정 (봇 차단 우회)
        page.set_extra_http_headers({
            'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
        })
        
        # 첫 페이지로 이동
        page.goto(url)
        
        # 페이지 로딩 완료까지 대기
        page.wait_for_load_state()
        
        # 모든 페이지의 행사를 저장할 리스트
        all_page_events = []
        
        # 페이지네이션 처리: 1-5페이지 순회
        for page_num in range(1, 6):
            try:
                print(f"📄 {page_num}페이지 처리 중...")
                
                # 2페이지부터는 페이지 버튼 클릭
                if page_num > 1:
                    # 페이지 번호 버튼 클릭
                    page.click(f'[data-page="{page_num}"]')
                    
                    # 페이지 로딩 대기 (2초)
                    page.wait_for_timeout(2000)
                    
                    # 페이지 로딩 완료까지 대기
                    page.wait_for_load_state()
                
                # 현재 페이지의 완전한 HTML 가져오기
                html_content = page.content()
                
                # HTML을 BeautifulSoup으로 파싱
                soup = BeautifulSoup(html_content, 'html.parser')
                
                # 현재 페이지의 모든 행사 카드 찾기
                events = soup.select('article.event_area.event_main')
                
                print(f"  → {len(events)}개 행사 발견")
                
                # 현재 페이지의 각 행사 정보 추출
                for i, event in enumerate(events):
                    try:
                        # 행사 제목 추출
                        title = event.find('h5', class_='title ellipsis').text.strip()
                        
                        # 가격 정보 추출
                        payment = event.find('span', class_='payment_type').text.strip()
                        
                        # 행사 날짜 추출
                        date = event.find('div', class_='date').text.strip()
                        
                        # 행사 장소 추출
                        place = event.find('span', class_='place').text.strip()
                        
                        # 상세 페이지 링크 추출
                        link = event.find('a')['href']
                        
                        # 행사 정보를 딕셔너리로 구조화
                        event_data = {
                            "id": len(all_page_events) + i + 1,  # 전체 리스트 기준 ID
                            "title": title,
                            "payment": payment,
                            "date": date,
                            "place": place,
                            "link": "https://onoffmix.com" + link,  # 절대 경로로 변환
                            "page": page_num  # 어느 페이지에서 가져왔는지 기록
                        }
                        
                        # 전체 행사 리스트에 추가
                        all_page_events.append(event_data)
                        
                    except Exception as e:
                        # 개별 행사 파싱 실패 시 에러 로그
                        print(f"    행사 추출 실패: {e}")
                
            except Exception as e:
                # 페이지 처리 실패 시 에러 로그 및 중단
                print(f"❌ {page_num}페이지 처리 실패: {e}")
                break
        
        # 브라우저 종료 및 자원 정리
        browser.close()
        
    # 모든 페이지의 행사 정보 반환
    return all_page_events

# 메인 실행 부분
if __name__ == "__main__":
    # Playwright 크롤링 실행
    all_events = crawl_with_playwright()
    
    # 최종 결과 출력
    print(f"\n🎉 총 {len(all_events)}개 행사 추출 완료!")
    
    # 페이지별 통계 계산
    page_stats = {}
    for event in all_events:
        page = event['page']
        if page not in page_stats:
            page_stats[page] = 0
        page_stats[page] += 1
    
    # 페이지별 수집 결과 출력
    for page, count in page_stats.items():
        print(f"  {page}페이지: {count}개")
```

문제점: 

- 기존 40줄 → 93줄로 증가 😱
- 핵심 로직(파싱)은 그대로인데 브라우저 관리 코드가 53줄 추가
- 복잡성 3.5배 증가

# AOP 적용 후

### 1. Aspect 생성

```python
# aspects/playwright_aspect.py
import functools
from playwright.sync_api import sync_playwright
from bs4 import BeautifulSoup

class PlaywrightAspect:
    @staticmethod
    def browser_management(func):
        """브라우저 생성/관리를 자동화하는 데코레이터"""
        @functools.wraps(func)
        def wrapper(url, *args, **kwargs):
            # Playwright 컨텍스트 시작
            with sync_playwright() as p:
                # 크로미움 브라우저 실행 (백그라운드)
                browser = p.chromium.launch(headless=True)
                
                # 새 페이지 생성
                page = browser.new_page()
                
                # 봇 차단 우회를 위한 User-Agent 자동 설정
                page.set_extra_http_headers({
                    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36'
                })
                
                try:
                    # 실제 함수 실행 (page 객체를 함수에 전달)
                    result = func(page, url, *args, **kwargs)
                    return result
                finally:
                    # 성공/실패 관계없이 브라우저 정리
                    browser.close()
                    print("🔒 브라우저 정리 완료")
        return wrapper
    
    @staticmethod
    def navigate_pages(max_pages=5):
        """페이지네이션 자동 처리 데코레이터"""
        def decorator(func):
            @functools.wraps(func)
            def wrapper(page, url, *args, **kwargs):
                # 첫 페이지로 이동
                page.goto(url)
                
                # 페이지 로딩 완료 대기
                page.wait_for_load_state()
                
                # 모든 페이지 결과를 저장할 리스트
                all_results = []
                
                # 지정된 페이지 수만큼 순회
                for page_num in range(1, max_pages + 1):
                    try:
                        print(f"📄 {page_num}페이지 처리 중...")
                        
                        # 2페이지부터는 페이지 버튼 클릭
                        if page_num > 1:
                            # 페이지 번호에 해당하는 버튼 클릭
                            page.click(f'[data-page="{page_num}"]')
                            
                            # JavaScript 실행 및 로딩 대기
                            page.wait_for_timeout(2000)
                            
                            # 페이지 로딩 완료 대기
                            page.wait_for_load_state()
                        
                        # 현재 페이지 처리 (실제 함수 호출)
                        result = func(page, page_num, *args, **kwargs)
                        
                        # 결과를 전체 리스트에 추가
                        all_results.extend(result)
                        
                    except Exception as e:
                        # 페이지 처리 실패 시 에러 로그 및 중단
                        print(f"❌ {page_num}페이지 실패: {e}")
                        break
                
                # 모든 페이지 결과 반환
                return all_results
            return wrapper
        return decorator
```

### 2. 적용된 크롤러

```python
# onoffmix_aop_crawler.py (AOP 적용)
from bs4 import BeautifulSoup
from aspects.playwright_aspect import PlaywrightAspect

# Playwright 관련 모든 작업을 Aspect가 자동 처리
@PlaywrightAspect.browser_management
@PlaywrightAspect.navigate_pages(max_pages=5)
def crawl_onoffmix_events(page, page_num):
    """핵심 로직: 현재 페이지에서 행사 정보 추출"""
    
    # 현재 페이지의 완전한 HTML 가져오기
    html_content = page.content()
    
    # HTML을 BeautifulSoup으로 파싱
    soup = BeautifulSoup(html_content, 'html.parser')
    
    # 현재 페이지의 모든 행사 카드 찾기
    events = soup.select('article.event_area.event_main')
    
    print(f"  → {len(events)}개 행사 발견")
    
    # 현재 페이지의 행사 정보를 저장할 리스트
    page_events = []
    
    # 각 행사 카드에서 정보 추출 (기존 로직과 동일)
    for i, event in enumerate(events):
        try:
            # 행사 제목 추출
            title = event.find('h5', class_='title ellipsis').text.strip()
            
            # 가격 정보 추출
            payment = event.find('span', class_='payment_type').text.strip()
            
            # 행사 날짜 추출
            date = event.find('div', class_='date').text.strip()
            
            # 행사 장소 추출
            place = event.find('span', class_='place').text.strip()
            
            # 상세 페이지 링크 추출
            link = event.find('a')['href']
            
            # 행사 정보를 딕셔너리로 구조화
            event_data = {
                "id": f"{page_num}-{i+1}",  # 페이지-순서 형태의 ID
                "title": title,
                "payment": payment,
                "date": date,
                "place": place,
                "link": "https://onoffmix.com" + link,  # 절대 경로로 변환
                "page": page_num  # 페이지 번호 기록
            }
            
            # 현재 페이지 행사 리스트에 추가
            page_events.append(event_data)
            
        except Exception as e:
            # 개별 행사 파싱 실패 시 에러 로그
            print(f"    행사 추출 실패: {e}")
    
    # 현재 페이지의 모든 행사 정보 반환
    return page_events

# 메인 실행 부분
if __name__ == "__main__":
    # 온오프믹스 AI 관련 행사 검색 URL
    url = "https://onoffmix.com/event/main?s=%EC%9D%B8%EA%B3%B5%EC%A7%80%EB%8A%A5%20ai%20chatgpt%20%EC%B1%97gpt"
    
    # AOP가 적용된 크롤링 실행 (브라우저 관리, 페이지네이션 자동 처리)
    all_events = crawl_onoffmix_events(url)
    
    # 최종 결과 출력
    print(f"\n🎉 총 {len(all_events)}개 행사 추출 완료!")
    
    # 페이지별 통계 계산
    page_stats = {}
    for event in all_events:
        page = event['page']
        if page not in page_stats
```

# 결과 비교 

### **코드량 비교**

```other
현재 (requests): 66줄
Playwright 도입 (AOP 없음): 124줄 (+58줄)
Playwright 도입 (AOP 적용): 80줄 + 79줄 = 159줄 (+35줄)

😱 AOP 적용했는데 35줄이 더 늘어났네요!
```

### **복잡도 비교**

```other
AOP 없음: 브라우저 관리 + 페이지네이션 + 파싱 로직 모두 섞임
AOP 적용: 파싱 로직만 집중, 나머지는 Aspect가 처리
```

### **유지보수성**

```other
AOP 없음: 브라우저 에러 시 70줄 코드 전체 디버깅
AOP 적용: 브라우저 에러 시 Aspect만 수정하면 됨
```

# AOP가 언제 의미 있는가? 

#### 1. **단일 사이트 크롤러**

```other
사이트 1개만 크롤링 → 코드 재사용 없음
→ AOP 도입 = 불필요한 복잡성 추가
```

#### 2. **작은 프로젝트**

```other
총 코드 100줄 미만 → 구조화할 필요 없음
→ AOP = 과도한 아키텍처
```

# 하지만! 사이트가 늘어나면...? 🚀

#### **AOP 없음**

```other
온오프믹스: 124줄
이벤터스: 124줄 (거의 동일한 코드 복사)
링크드인: 124줄 (또 거의 동일한 코드 복사)
총합: 372줄 (중복 코드 천국)
```

#### **AOP 적용**

```other
Aspect: 80줄 (한 번만 작성)
온오프믹스: 79줄
이벤터스: 79줄  
링크드인: 79줄
총합: 317줄 (55줄 절약)
```

### **5개 사이트 추가 시**

```other
AOP 없음: 124 × 5 = 620줄
AOP 적용: 80 + (79 × 5) = 475줄 (145줄 절약, 23% 절약)
```

# AOP의 현실적인 한계

```other
🚨 솔직한 고백:
"1개 사이트만 크롤링한다면 AOP는 오버엔지니어링입니다!"

✅ AOP가 유용한 경우:
- 3개 이상 사이트 크롤링
- 팀 프로젝트 (여러 개발자)
- 장기 유지보수 필요

❌ AOP가 불필요한 경우:  
- 1-2개 사이트만
- 일회성 프로젝트
- 개인 프로젝트
```

# **적정 기술 선택의 중요성**

```other
💭 개발자의 고민:
"지금 AOP를 도입할까? 나중에 도입할까?"

📈 현실적인 판단 기준:
- 현재: 1개 사이트 → AOP 불필요
- 2-3개 사이트 계획 → AOP 고려
- 5개+ 사이트 계획 → AOP 필수
```

# 결론

- “내 프로젝트에서 크롤링할 사이트가 몇 개인가?”
- 사이트 개수 먼저 파악하기 (현재 계획 + 미래 확장 계획 = 총 예상 사이트 수)

