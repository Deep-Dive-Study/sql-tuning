# 1장 SQL 처리 과정과 I/O

## 1.1 SQL 파싱과 최적화

### 1.1.1 구조적, 집합적, 선언적 질의 언어
- SQL : **`Structured Query Language`**
  - SQL은 `구조적`이고 `집합적`이고 `선언적`인 질의 언어
- 원하는 결과 집합을 구조적, 집합적으로 선언하지만, 그 결과 집합을 만드는 과정은 절차적일 수 밖에 없음
  - 즉, **프로시저(실제 실행 코드)가 필요함**
- 결과 집합을 만들어 내는 역할을 **`옵티마이저`** 가 수행 (프로그래밍을 대신 해준다고 보면 된다.)
- DBMS 내부에서 프로시저를 작성하고 컴파일해서 실행 가능한 상태로 만드는 전 과정을 **`SQL 최적화`** 라고 함

  <img width="422" alt="CleanShot 2024-07-03 at 00 32 30@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/52d316c1-374a-4353-928c-38cb37f5b133">

### 1.1.2 SQL 최적화
1. **`SQL 파싱`**
  - 사용자로부터 SQL을 전달 받으면 가장 먼저 SQL 파서(Parser)가 파싱을 진행함 
    - 파싱 트리 생성 : SQL문을 이루는 개별 구성요소를 분석하여 파싱 트리 생성
    - Syntax 체크 : 문법적 오류가 없는지 확인(SQL 문법에 맞는지 체크)
    - Semantic 체크 : 의미상 오류가 없는지 확인(테이블, 컬럼, 인덱스 등이 존재하는지 체크)
2. **`SQL 최적화`**
  - 옵티마이저가 역할을 수행하는 부분
  - 옵티마이저는 미리 수집한 시스템 및 오브젝트 통계정보를 바탕으로 다양한 실행 경로를 생성해서 비교 후 가장 효율적인 경로를 선택함
3. **`로우 소스 생성`** 
  - 로우 소스 생성기가 역할을 수행하는 부분
  - 실행 가능한 코드 또는 프로시저 형태로 포맷팅하는 단계

### 1.1.3 SQL 옵티마이저
- SQL 옵티마이저 : `사용자가 원하는 작업을 가장 효율적으로 수행할 수 있는 최적의 데이터 액세스 경로를 선택해주는 DBMS의 핵심 엔진`
- 전달받은 쿼리를 수행하기 위한 실행계획들을 찾아냄
- 데이터 딕셔너리에 미리 수집해둔 오브젝트에 통계 및 시스템 통계 정보를 이용해 각 실행계획의 예상 비용 산정
- 최저 비용을 나타내는 실행계획 선택

  <img width="406" alt="CleanShot 2024-07-03 at 00 33 10@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/296350dd-259d-4fea-a9bd-8358f2dd1b53">

### 1.1.4 실행계획과 비용
- DBMS에서 실행 계획을 미리 확인할 수 있는 기능이 있음 → **`실행계획(Execution Plan)`**
- 실행하는 실행계획은 옵티마이저가 비용을 근거로 선택한 것
- 비용은 쿼리를 실행하는 동안 발생할 것으로 `예상되는` `I/O 횟수, 예상 소요 시간`을 표현한 것

  <img width="453" alt="CleanShot 2024-07-03 at 00 33 49@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/83029b4f-48e6-45ef-bd25-d4bf1e5dd051">

#### (참고)옵티마이저가 실행계획을 세우기 위해 고려하는 수많은 정보들
    - 테이블, 컬럼, 인덱스 구조에 관한 기본 정보
    - 오브젝트 통계: 테이블 통계, 인덱스 통계, (히스토그램을 포함한) 컬럼 통계
    - 시스템 통계: CPU 속도, Single Block I/O 속도, Mutliblock I/O 속도 등
    - 옵티마이저 관련 파라미터

### 1.1.5 옵티마이저 힌트
- 옵티마이저가 항상 최선의 선택을 하는 것은 아님
- 따라서, 힌트를 통해서 사용자가 의도적으로 옵티마이저가 다른 실행계획을 선택할 수 있도록 유도할 수 있음
  - 상황이나 서비스 특성을 고려
- 상황에 따라 사용하거나 사용하지 않을 수 있음
- MySQL에서 힌트를 사용하기 위해서 주석 기호에 +를 붙여 작성한다

예시: `SELECT /*+ Index(PK) */`
  
  <img width="210" alt="CleanShot 2024-07-03 at 00 37 24@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/930c22ed-2da8-41a8-ab4f-bfc98371ad33">

#### 주의사항
- 힌트 안에 인자를 나열할 땐 `','` 를 사용할 수 있지만, 힌트와 힌트 사이에 사용하면 안됨

  <img width="319" alt="CleanShot 2024-07-03 at 01 03 13@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/95145af3-7370-4728-9a66-248a7b438296">

- 테이블을 지정할 땨 아래와 같이 스키마명까지 명시하면 안됨 

  <img width="214" alt="CleanShot 2024-07-03 at 01 03 44@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/f8e9fbaf-982b-4622-91fc-3224f0df0308">

- FROM 절에 테이블명 옆에 ALIAS를 지정했다면, 힌트에도 반드시 ALIAS를 사용해야함

  <img width="174" alt="CleanShot 2024-07-03 at 01 04 39@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/737abba3-f407-4932-a39f-beb9c8281c5d">

- 기왕에 힌트를 쓸거면, 빈틈없이 기술해야함

## 1.2 SQL 공유 및 재사용

### 1.2.1 소프트 파싱 vs 하드 파싱
- 캐시되어 있지 않는 SQL 쿼리의 경우 SQL 파싱, 최적화, 로우 소스 생성 등을 거쳐 내부 프로시저를 생성하는데 이를 `하드 파싱`이라 하고, 캐시된 쿼리를 바로 찾아 쓰는 것을 `소프트 파싱`이라 함
  - `라이브러리 캐시(쿼리 캐시)`: SGA의 구성요소. SQL파싱, 최적화, 로우 소스 생성 과정을 거친 내부 프로시저는 라이브러리 캐시에 저장되어 재사용됨
    - `SGA`: 서버 프로세스와 백그라운드 프로세스가 공통으로 액세스하는 데이터와 제어 구조를 캐싱하는 메모리 공간

      <img width="354" alt="CleanShot 2024-07-03 at 00 43 41@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/ca65b2c7-aac5-43f8-81b8-5c14f41468a5">

  - `하드 파싱`: 파싱한 SQL이 라이브러리 캐시에 존재하지 않아서 최적화, 로우 소스 생성과정을 모두 거치는 것
  - `소프트 파싱`: 파싱한 SQL이 라이브러리 캐시에 존재하면 바로 실행

- 즉, DBMS가 SQL을 파싱한 후 해당 SQL이 라이브러리 캐시에 존재하는지 확인하여
  - 있는 경우 캐시에서 가져와서 실행 (소프트 파싱)
  - 없는경우 최적화, 로우소스 생성, 등의 모든 과정을 거치고 실행(하드 파싱)

  <img width="436" alt="CleanShot 2024-07-03 at 00 44 57@2x" src="https://github.com/Deep-Dive-Study/sql-tuning/assets/99165624/fd325ca7-b0ed-4211-b520-9b4395d824cc">

#### SQL 최적화 과정은 왜 하드(Hard)한가(재사용 하는 이유)
  - 최적화 과정에서 딕셔너리, 통계 정보를 읽어 효율성을 판단하는 과정은 많은 경우의 수를 계산해야 하기 때문에 CPU를 많이 사용하는 무거운 작업임

### 1.2.2 바인드 변수의 중요성
- 사용자정의 함수/프로시저, 트리거, 패키지 등은 생성할 때부터 이름을 가지며, 컴파일시 딕셔너리에 저장되어 별도로 사용자가 삭제하지 않는 한 영구적으로 보관됨
- 반면에, 라이브러리 캐시에 저장되는 SQL에는 이름이 따로 없음
  - `SQL 전체가 하나의 이름이 되기 때문에` 이름을 만들 필요가 없음
- **`SQL자체가 검색을 위한 key`** 가 됨
  - 대/소문자, 띄어쓰기 등 **약간만 달라져도 매번 하드 파싱하고 캐시에 저장을 함**
- DBMS에서 수행되는 SQL이 모두 완성된 SQL이 아니며, 일회성 쿼리도 많음.
- 이 모든 쿼리를 저장하려면 공간 낭비가 심해지기 때문에 상용 DBMS에서는 이러한 SQL을 영구 저장하지 않는 방향으로 감

#### 바인드 변수
- 파라미터 Driven 방식으로 SQL을 작성하는 방법

- `바인드 변수를 사용하지 않는 경우`

  ```sql
    SELECT * FROM CUSTOMER WHERE LOGIN_ID = 'ko'
    SELECT * FROM CUSTOMER WHERE LOGIN_ID = 'kwon'
    SELECT * FROM CUSTOMER WHERE LOGIN_ID = 'kim'
  ```

- `바인드 변수를 사용하는 경우`
  ```sql
  SELECT * FROM CUSTOMER WHERE LOGIN_ID = :1
  ```

- 바인드 변수를 사용하지 않은 경우, 수많은 비효율적인 쿼리가 모두 라이브 캐시에 저장되는 문제가 발생함
