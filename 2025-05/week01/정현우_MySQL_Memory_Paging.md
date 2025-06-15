## Intro

이번 주제는 Memory & Paging입니다! 이번 주제로는 무엇을 해볼까 고민하다, 저번 주 발표자료인 DB와 Lock에 대해 준비하며 DB의 세부 정책에 대해 파악하는 것의 중요성에 대해 떠올렸습니다. 그래서 이번에도 DB와 연관지어 설명하면 좋겠다 싶었습니다.

그래서 고민한 오늘의 주제는 “MySQL의 메모리 구조와 Buffer Pool의 Paging”입니다! 먼저 왜 수 많은 RDBMS 중 MySQL이냐? 그 이유는 바로 제가 ‘Real MySQL’이라는 유명한 책을 사놓고 제대로 안읽었기 때문이죠… 이렇게 퀄리티 좋고 자세한 자료가 있는데, 이걸 활용하는게 좋지 않을까 싶어 MySQL로 DB를 정했습니다. ㅎㅎㅎ 제가 주로 쓰는 건 PostgreSQL이지만, MySQL을 자세히 알면 PostgreSQL를 이해하는 것도 훨씬 수월할테니까요. 자 그럼 지금부터 MySQL의 전체 구조 및 메모리 구조 파악부터 시작해봅시다!

---

## 1️⃣ **MySQL 아키텍처**

MySQL은 가장 널리 사용되는 관계형 데이터베이스 관리 시스템(RDBMS) 중 하나입니다.

하지만 MySQL 역시 하나의 프로그램이기 때문에, 내부적으로 여러 역할을 나누어 담당하는 **모듈**과 **구성 요소**들이 존재합니다.

이를 크게 나누면 다음과 같은 두 가지 층으로 구분할 수 있습니다.

1. **MySQL Engine Layer**
2. **Storage Engine Layer**

<p align="center">
  <img src="https://github.com/user-attachments/assets/0ba12d83-ed36-4a11-af0e-2b8286006c29" width="500">
</p>

### 🧠 1. **MySQL Engine** Layer – 질문을 해석하고 계획을 세우는 뇌

MySQL Engine Layer는 사용자가 보낸 SQL 문장을 **해석하고 실행 계획을 세우는 역할**을 합니다.

‘*질문을 받고 어떻게 답할지 계획을 세우는 **뇌***’와 같은 역할을 한다고 볼 수 있죠.

예를 들어 `SELECT * FROM users WHERE id = 1;`라는 쿼리를 보냈다면, SQL Layer는 다음과 같은 작업을 수행합니다.

- **Connection handler**: 사용자의 연결 요청을 받아 세션을 생성하고 유지
- **Parser**: 문법 검증
- **Optimizer**: 어떤 인덱스를 사용할지 결정
- **Query Cache**: 이미 캐시된 쿼리 결과가 있는지 확인

👉 즉, SQL Layer는 **연결 관리부터 쿼리 실행 계획까지**의 모든 과정을 처리하며, **실제 데이터를 다루기 전 단계**를 담당합니다.

### 🦾 2. Storage Engine Layer – 계획된 일을 실제로 수행하는 손과 발

MySQL Engine Layer가 “어떻게 처리할지 계획”을 세웠다면,

Storage Engine Layer는 그 계획에 따라 **“실제 데이터를 다루는 작업을 수행하는 손과 발”** 역할을 합니다.

즉, 데이터를 저장하고 꺼내오는 **실질적인 작업**이 일어나는 곳입니다.

MySQL은 여러 종류의 Storage Engine을 지원하며, 필요에 따라 그 중 하나를 선택해 사용할 수 있습니다.

(걸을 때는 발, 물건을 잡을 때는 손이 필요한 것처럼 말예요)

그중 가장 많이 사용되는 Storage Engine은 **InnoDB**입니다.

| Storage Engine | 주요 특징 |
| --- | --- |
| **InnoDB** | 트랜잭션 지원, ACID 보장, MySQL 기본 엔진 |
| MyISAM | 트랜잭션 미지원, 과거에 사용 빈도 높음 |

이번 주제에 대해 자세히 들어가기 위해선, 이 InnoDB가 뭐하는 녀석인지, 뭐 때문에 얼마나 중요한지 알아야 하지만 그건 다음 파트에서 다루도록 하겠습니다. 그 전에 지금까지 MySQL의 전체적인 아키텍쳐에 대해 보았으니, 이제 MySQL을 ‘Memory’의 관점에서 보도록 합시다.

---

## 2️⃣.  **MySQL 메모리 구조**

MySQL은 단순히 디스크에만 데이터를 저장하고 읽어오는 것이 아니라, **성능을 높이기 위해 메모리(RAM)를 적극적으로 활용**합니다. 데이터베이스 서버가 메모리를 잘 사용하지 못한다면, 매번 디스크에서 데이터를 읽어와야 하기 때문에 속도가 크게 느려질 것입니다.

이런 이유로 MySQL은 메모리를 크게 **두 가지 영역**으로 나누어 사용합니다.

1. **Global Memory (공유 메모리)**
2. **Session Memory (세션 메모리)**

<p align="center">
  <img src="https://github.com/user-attachments/assets/77494927-d133-4f20-ab56-01bb6a0ecf83" width="500">
</p>

### **🗂️ 1. Global Memory (공유 메모리)**

Global Memory는 **MySQL 서버 전체가 공유하는 메모리 영역**입니다.

서버가 시작할 때 초기화되며, 모든 클라이언트 세션이 함께 사용하는 메모리입니다.

쉽게 말해, **여러 사람이 함께 사용하는 공용 창고 같은 공간**이라고 비유할 수 있습니다.

Global Memory에는 다음과 같은 예시가 포함됩니다.

- **InnoDB Buffer Pool**: 디스크 데이터를 메모리에 캐시하여 빠른 조회를 가능하게 함
- Query Cache (현재는 기본적으로 비활성화됨)
- Key Buffer (MyISAM 전용 인덱스 캐시)

이 중에서도 특히 중요한 것은 **InnoDB Buffer Pool**입니다. 왜 제일 중요한지는 다음 단계에서 다뤄보겠습니다

### **🙋‍♂️ 2. Session Memory (세션 메모리)**

Session Memory는 **각 클라이언트 연결(Session)마다 개별적으로 할당되는 메모리**입니다.

쉽게 말해, **각 사용자가 독립적으로 사용하는 개인 책상**이라고 생각할 수 있습니다.

사용자가 쿼리를 실행할 때 필요한 임시 메모리 공간으로,

쿼리의 복잡성이나 처리해야 하는 데이터 양에 따라 크기가 달라질 수 있습니다.

Session Memory에는 다음과 같은 예시가 있습니다.

1. **Sort Buffer** (sort_buffer_size)
    - ORDER BY, GROUP BY, DISTINCT 등 모든 정렬 작업에 필수
    - 가장 빈번하게 사용되는 세션 버퍼
    - 성능 영향도 매우 큼
2. **Tmp Table Size** (tmp_table_size)
    - 복잡한 쿼리에서 임시 테이블 생성 시 사용
    - 크기 부족 시 디스크로 전환되어 성능 급격히 저하
    - 집계/분석 쿼리에서 매우 중요
3. **Join Buffer** (join_buffer_size)
    - 테이블 조인 시 사용
    - 인덱스 미사용 조인에서 성능에 큰 영향
    - 대용량 처리나 OLAP 환경에서 특히 중요
4. **Thread Stack** (thread_stack)
    - 모든 쿼리 실행의 기본이 되는 스택
    - 기본값으로도 충분한 경우가 많음
    - 재귀 쿼리나 깊은 중첩에서만 조정 필요
5. **Read Buffer** (read_buffer_size)
    - 순차 테이블 스캔 시 사용
    - 특정 상황에서만 성능에 영향
    - InnoDB보다 MyISAM에서 더 유용
6. **Query Cache** (query_cache_min_res_unit)
    - MySQL 8.0부터 제거됨
    - 현재는 사용되지 않는 기능

이 메모리는 각 세션별로 할당되므로, **동시 접속자 수가 많아질수록 총 메모리 사용량이 커질 수 있습니다.**

따라서 Session Memory 설정 시에는 **최대 접속자 수와 쿼리 복잡성을 고려해 적절한 값을 설정하는 것이 중요합니다.**

MySQL의 메모리 구조를 본다는게 너무 길어졌네요. 그래도 그만큼 Session Memory는 상황에 따라 튜닝에 큰 영향을 줄 수 있다고 하니 무엇이 있는지 정도는 알아두는게 좋겠습니다.

이제 MySQL의 핵심, InnoDB에 대해서 살펴봅시다.

---

## 3️⃣ **InnoDB란?**

### 1. 개념

InnoDB는 MySQL에서 **데이터를 저장하고 관리**하는 **스토리지 엔진(Storage Engine)**입니다.

MySQL은 여러 가지 스토리지 엔진을 지원하지만, MySQL 5.5 버전부터 InnoDB가 기본 엔진으로 채택되었으며, 현재 가장 널리 사용되는 엔진입니다.

InnoDB는 단순히 데이터를 저장하는 것에 그치지 않고,

**트랜잭션, 외래 키 제약조건, 장애 복구 기능 등을 지원**하여 데이터 무결성과 안정성을 높여줍니다.

### **2. InnoDB의 주요 특징**

InnoDB는 다음과 같은 주요 특징을 가집니다.

| 특징 | 설명 |
| --- | --- |
| 트랜잭션 지원 (ACID) | 데이터 일관성과 무결성을 보장하는 트랜잭션 기능 |
| 외래 키 지원 | 테이블 간 참조 관계를 자동으로 관리 |
| MVCC (다중 버전 동시성 제어) | 여러 사용자가 동시에 같은 데이터를 접근할 수 있도록 지원 |
| 크래시 복구 기능 | 장애 발생 시 데이터 손실을 최소화하고 복구 가능 |

딱 보시면 InnoDB에서 제공하는 기능이 없다면 MySQL이 DB로서 기능할 수 있을까 싶죠?

이처럼 **“MySQL을 사용한다 = InnoDB를 사용한다”**라고 해도 과언이 아닙니다.

✅ **MySQL에서 InnoDB의 중요성**

- MySQL 기본 스토리지 엔진 (대부분의 테이블이 InnoDB 사용)
- 데이터 무결성, 트랜잭션, 성능의 균형
- 실무 환경에서 필수적인 기능 제공
- 은행, 쇼핑몰, ERP 같은 **데이터 신뢰성이 중요한 서비스에서 표준 엔진으로 사용**

### 3. InnoDB의 구조

InnoDB는 **디스크와 메모리를 함께 사용하는 하이브리드 구조**로 설계되었습니다.

데이터는 디스크에 영구적으로 저장되지만, **성능을 위해 메모리 캐시를 적극적으로 활용**합니다.

아래 그림처럼 InnoDB의 구조는 크게 **메모리 영역(In-Memory Structures)**와 **디스크 영역(On-Disk Structures)**으로 나눌 수 있습니다.

<p align="center">
  <img src="https://github.com/user-attachments/assets/816fa14d-e729-4b2d-95cf-b5d40c6b5896" width="500">
</p>

이제 각 영역의 역할을 간단히 살펴보겠습니다.

**1. 메모리 구조 (In-Memory Structures)**

✅ **Buffer Pool** 👉 InnoDB 성능의 핵심 요소

- Buffer Pool은 **디스크 데이터를 페이지(Page) 단위로 메모리에 캐시해두는 공간**입니다.
- 자주 사용하는 데이터를 미리 메모리에 올려두어 디스크 I/O 없이 빠르게 접근할 수 있게 도와줍니다.

✅ **Change Buffer**

- Change Buffer는 **비활성 인덱스(Secondary Index)에 대한 변경 작업을 임시 저장**하는 공간입니다.
- 작은 변경을 모아뒀다가 나중에 한 번에 디스크에 반영하여 디스크 I/O를 줄입니다.

✅ **Log Buffer**

- 트랜잭션 로그(Redo Log)를 디스크에 기록하기 전에 **임시로 보관하는 메모리 버퍼**입니다.

**2. 디스크 구조 (On-Disk Structures)**

InnoDB의 디스크 구조에는 다양한 **Tablespace**가 있습니다.

Tablespace는 **데이터와 인덱스를 저장하는 논리적 저장 공간**으로,

하나의 큰 저장소일 수도 있고, 테이블별로 파일이 따로 생성될 수도 있습니다.

✅ **기본 Tablespace (System Tablespace)**

- InnoDB가 기본적으로 사용하는 저장소

✅ **File-Per-Table Tablespace**

- 각 테이블마다 별도의 파일(.ibd)로 데이터를 저장

✅ **Undo Tablespace, Redo Log, 임시 Tablespace** 등

- 데이터 무결성과 복구, 임시 작업을 위한 공간 존재
- 여러 Tablespace가 존재하지만, 이번 글의 주제인 Buffer Pool과 직접 연결된 부분은 데이터 페이지를 캐시하는 역할과 관련이 있습니다.
- 따라서 Tablespace의 구체적 세부 사항은 심화 학습 시 다루면 충분하며, 여기서는 “데이터가 저장되는 다양한 공간이 존재한다”는 정도로 이해하면 됩니다.

InnoDB의 Buffer Pool은 MySQL 전체 메모리 사용량의 70~80%을 차지하고, 디스크 병목을 예방하는 역할을 하기에 제일 중요하다고 볼 수 있습니다. 다음으로는 Buffer Pool에 대해서 자세히 살펴보도록 하겠습니다.

---

## 4️⃣ **InnoDB Buffer Pool**

InnoDB는 디스크에 저장된 데이터를 빠르게 처리하기 위해 **Buffer Pool이라는 메모리 캐시**를 사용합니다.

Buffer Pool은 InnoDB의 성능을 좌우하는 가장 중요한 메모리 공간입니다.

**✅ Buffer Pool의 데이터 관리 단위: Page**

Buffer Pool은 데이터를 **Page 단위(기본 16KB)**로 관리합니다.

각 Page에는 여러 개의 데이터 행(Row)이 포함될 수 있으며,

InnoDB는 데이터를 한 줄씩이 아니라 **페이지(블록) 단위로 가져오고 저장**합니다.

### 1. Buffer Pool의 내부 구조

Buffer Pool 안에서도 Page는 단순히 쌓여 있는 것이 아니라,

**세 가지 주요 리스트(List)**를 통해 관리됩니다.

이 리스트들은 **어떤** Page**를 더 오래 메모리에 유지할지, 어떤** Page**를 디스크로 내보낼지 결정**하는 역할을 합니다.

1. **Buffer Pool List**
    - Page를 캐싱하고 관리하는 리스트입니다
    - Buffer Pool의 핵심 List이기에 아래서 자세히 살펴보겠습니다.
2. **Flush List**
    - Flush List는 **변경된(DIRTY) 페이지를 추적하는 리스트**입니다.
    - InnoDB는 데이터를 수정해도 즉시 디스크에 기록하지 않고, 우선 Buffer Pool 안에서 변경된 상태로 유지하다가 나중에 디스크에 한꺼번에 기록(Flush)합니다.
    - Flush List는 **어떤 페이지가 디스크에 기록되어야 하는지 관리하는 역할**을 합니다.
3. **Free List**
    - Free List는 **Buffer Pool 안에서 비어 있는 페이지를 관리하는 리스트**입니다.
    - 새로운 데이터를 올릴 때 Free List에 있는 공간부터 사용합니다.

### 2. Buffer Pool List

Buffer Pool List는 두 개의 영역으로 나뉘고, Paging을 통해 리스트를 관리합니다.

<p align="center">
  <img src="https://github.com/user-attachments/assets/bb9782c5-63f7-42fd-aadf-d10cf94b910d" width="500">
</p>

✅ **New Sublist (MRU 영역)**

- 최근 접근된 페이지가 들어가는 영역
- 전체 리스트의 상위 **5/8** 비율 차지
- 사용자가 자주 접근하는 데이터가 여기에 머무릅니다.

✅ **Old Sublist (LRU 영역)**

- 한동안 접근하지 않은 페이지가 내려오는 영역
- 전체 리스트의 하위 **3/8** 비율 차지
- 이 영역의 끝에 가까워질수록 페이지는 **메모리에서 제거(evict)**될 가능성이 커집니다.
- 새로운 페이지가 Buffer Pool로 들어올 때는 **중간 지점(midpoint)**에 삽입됩니다.

위와 같은 Paging 정책 덕분에 Buffer Pool은 **자주 사용하는 데이터를 메모리에 유지하고, 불필요한 디스크 I/O를 줄일 수 있습니다.**

### 3. Buffer Pool과 성능

Buffer Pool의 크기와 관리 방식은 InnoDB의 성능에 큰 영향을 줍니다.

- Buffer Pool이 충분히 크면 **자주 사용하는 데이터를 메모리에 오래 유지 → 빠른 응답**
- Buffer Pool이 너무 작으면 **자주 Paging이 일어나 → 디스크 I/O 증가 → 성능 저하**

실무에서는 `innodb_buffer_pool_size`, `innodb_old_blocks_pct`, `innodb_old_blocks_time` 같은 설정값을 조정해 Buffer Pool의 효율을 높이기도 합니다.

## 5️⃣ 결론

이번 글에서는 MySQL의 메모리 구조, 특히 InnoDB의 Buffer Pool과 그 내부 구조, 그리고 페이지 관리 방식(Paging)에 대해 알아보았습니다.

또한 MySQL은 단순히 디스크에 데이터를 저장하고 읽어오는 시스템이 아니라, **성능을 높이기 위해 메모리를 적극적으로 활용하는 시스템**임을 확인했습니다.

그리고 그 중심에 있는 것이 바로 **InnoDB Buffer Pool**입니다.

✅ Buffer Pool은 **자주 사용하는 데이터를 메모리에 캐시하여 디스크 I/O를 줄이고 성능을 높이는 공간**입니다.

✅ Buffer Pool은 **Buffer Pool List라는 복합 리스트 구조(LRU + MRU)**를 통해 페이지를 효율적으로 관리합니다.

✅ Buffer Pool은 메모리 공간의 한계 때문에 **Paging(페이지 교체)**을 통해 오래된 페이지를 제거하고 새 페이지를 수용합니다.

이 모든 구조는 결국 **디스크 접근을 최소화하고, 메모리 사용 효율을 극대화하여 데이터베이스 성능을 끌어올리기 위한 것**입니다.

**💡 느낀점: CS 기본 개념의 중요성**

이번 글에서 다룬 **“Paging”이라는 개념은 운영체제(OS)에서 메모리 관리의 핵심 개념**으로 배웠던 바로 그 Paging과 같은 원리입니다.

즉, Paging은 OS뿐 아니라 우리가 실무에서 다루는 데이터베이스(MySQL InnoDB)에서도

**메모리와 디스크를 효율적으로 관리하기 위해 필수적으로 사용되는 개념**입니다.

👉 결국, **CS의 기본 개념은 교과서 안에만 머무는 것이 아니라,
실제로 우리가 사용하는 시스템과 소프트웨어에 깊게 적용되어 있다는 사실**을 보여줍니다.

따라서 개발자로서 **CS 핵심 개념을 탄탄하게 학습하는 것이 실무 문제를 이해하고 해결하는 데 매우 중요함**을 다시 한 번 상기할 수 있습니다.


## 출처

https://dev.mysql.com/doc/

https://product.kyobobook.co.kr/detail/S000001766482