# AOP가 무엇일까
AOP(Aspect Oriented Programming)의 약자

~~(AOP라는 용어를 이번에 처음 들었다)~~

주된 목적은 핵심 로직과 부가적인 관심사(공통 기능, 로깅, 트랜잭션, 보안 등)를 분리해서 **코드를 더 깔끔하고 유지보수하기 쉽게 만드는 것**

# AOP 용어

* Aspect : 부가 기능(로깅, 트랜잭션)
* Advice : 언제 실행할지 정의(메서드 실행 전/후)
* Join Point : Advice가 적용될 지점
* Pointcut : 어떤 join point에 적용할지 조건 설정
* Weaving : 실제 코드에 Aspect를 끼워넣는 과정

# Streamlit의 예시

### Streamlit 이란?

A faster way to build and share data apps(데이터 어플리케이션을 만들고 공유하는 빠른 방법)

데이터를 보여주는 것에 초점을 맞춘 Python 기반 데모용 프레임워크라고 한다.

우리 회사의 경우 데이터를 보기 위해 만드는 애플리케이션이 대부분이기 때문에 진입장벽도 낮고, 결과물도 빨리 볼 수 있는 streamlit이 선호된다.

### Streamlit에 AOP를 적용하면 어떤 점이 달라질까?

로그들을 수집해 모니터링 시스템의 한 항목으로 보여주게 되면, 병목이 일어나는 구간을 알 수 있어 안그래도 느린 Streamlit을 조금이나마 빠르게 응답할 수 있도록 사용자 경험을 개선시킬 수 있지 않을까? 싶기도 하고, 만약 배포된 서비스에서 문제가 생겼는데 우리의 인프라 관리가 문제라며 책임을 전가하는 행위를 확실히 방어할 수 있을것이라 생각한다. 

로그를 보여주며 "이거 봐!!니가 짠 코드에서 이 함수를 호출 할 때 응답 속도가 오래걸려서 생기는 문제잖아!!" 라고 하면 할 말이 없을테니까 말이다. 하하하

### Streamlit의 예시

### 1. Aspect 정의 : 실행 시간 측정 기능

```
import time
import streamlit as st
from functools import wraps
from prometheus_client import Summary, start_http_server

FUNCTION_EXECUTION_TIME = Summary('streamlit_function_execution_seconds', '실행 시간 (초)', ['function_name'])

start_http_server(8000)  # http://localhost:8000/metrics

# Aspect: 실행 시간 측정
def log_execution_time(func): <- Advice + Weaving
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        FUNCTION_EXECUTION_TIME.labels(function_name=func.__name__).observe(duration)
        return result
    return wrapper
```

나는 Prometheus로 메트릭을 수집하여 Grafana에 시각화하는 구성을 생각해보았다.

prometheus_client 라이브러리를 사용하여, Streamlit에서 Prometheus가 이해할 수 있는 형식의 메트릭을 HTTP로 노출해야한다.

### 2. Pointcut : 어떤 함수에 적용할지 지정

위 데코레이션 함수를 

```
@log_execution_time <- **pointcut**
def load_dashboard(): <- **joint point**
    st.write("대시보드 로딩 중")
    time.sleep(2)
    st.write("로딩 완료")

@log_execution_time
def show_stats():
    st.write("통계 분석 중")
    time.sleep(1)
    st.write("분석 완료")

# ✅ Streamlit 앱 실행
st.title("streamlit app")

if st.button("대시보드 보기"):
    load_dashboard()

if st.button("통계 보기"):
    show_stats()
```

위처럼 설정할 경우, 함수가 실행되는 실행 시간을 측정할 수 있다.

## 내가 할 수 있는 것

여기서 끝내긴 아쉬워서 내가 실무에 적용할 수 있는 것들을 생각해보았다.

일단 비개발자들이 개발을 하는 것이니, 데코레이터 함수를 내가 작성하고, 함수의 바로 위에 붙여두면 된다는 가이드를 하고, 해당 메트릭들을 서버에서 수집한 다음 Grafana에 시각화해서 각 애플리케이션마다의 상태를 볼 수 있다면 어떤 문제 사항이나 불만이 접수되었을때 어떤 부분에서 응답 지연이 일어나는지를 체크할 수 있을 것이다.

실제로도 나에게 문의가 많이 들어온다. 본인의 서비스가 느린 이유를 모르겠다며...(...)
나에게 찾아달란 식의 요청이 있다. 그럼 나는 내가 쓴 코드도 아닌 것들을 죄다 읽어야 한다.

대충 여기서 늦어질것 같은데 싶은 부분들이 보이긴 하지만 수치로 확인할 수 있는 부분이 아니기 때문에 그냥 추측만 할 뿐이다.

하지만 이것들을 적용한다면 내 업무가 많이 줄어들고 사용자 경험도 개선할 수 있지 않을까?생각한다.