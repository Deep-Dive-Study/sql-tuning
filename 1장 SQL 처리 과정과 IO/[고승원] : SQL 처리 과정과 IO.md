# 1장 SQL 처리 과정과 I/O

# 1.1 SQL 파싱과 최적화

## 1.1.1 구조적, 집합적, 선언적 질의 언어

SQL : Structured Query Language의 줄임말로서 구조적, 집합적, 선언적 질의 언어이다.

## 1.1.2 SQL 최적화

SQL 실행 최적화 과정

### 1. SQL 파싱

사용자로부터 SQL을 전달받은 뒤 SQL 파서가 파싱한다.

- 파싱 트리 생성 : SQL문을 이루는 개별 구성요소를 분석해서 트리 생성
- Syntax 체크 : 문법적 오류가 없는지 확인하고, 올바르지 않은 키워드나 누락된 키워드 확인
- Semantic 체크 : 의미상 오류 확인 / 테이블 존재 유무, 오브젝트 권한 유무

### 2. SQL 최적화

옵티마이저는 미리 수집한 통계정보를 바탕으로 다양한 실행경로를 생성해서 가장 효율적인 하나를 선택한다.

데이터베이스 성능의 핵심은 옵티마이저이다.

### 3. 로우 소스 생성

SQL 옵티마이저가 선택한 실행경로를 실제 실행 가능한 코드 또는 프로시저 형태로 포매팅 한다.

로우 소스 생성기가 그 역할을 한다.

## 1.1.3 SQL 옵티마이저

옵티마이저의 최적화 단계

1. 사용자로부터 전달받은 쿼리 수행 후보 계획들을 찾아낸다.
2. 데이터 딕셔너리에 미리 수집해 둔 오브젝트 통계 및 시스템 통계를 이용해 각 실행계획의 예상비용을 산정한다.
3. 예상비용이 최저인 실행계획을 선택한다.

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/4bf59014-95aa-454d-8c04-7cfeabe15654)

## 1.1.4 실행계획과 비용

SQL 실행계획을 확인한 뒤 원하는대로 실행경로를 변경할 수 있다. (쿼리 앞에 Explain)

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/0047487f-5a57-4d11-8f82-f751f7f24f26)

옵티마이저가 실행계획을 선택하는 근거 = Cost (예측치)

## 1.1.5 옵티마이저 힌트

옵티마이저가 결정한 실행계획이 마음에 들지 않을 때 (인덱스를 안탄다거나) 힌트로 원하는대로 변경할 수 있다.

```java
SELECT /*+ INDEX(A PK) */
	컬럼, 컬럼, 컬럼
FROM 테이블 A
WHERE ID = ??
```

### 주의사항

- 힌트 안에 인자를 나열할 때 ,를 사용해도 되지만 힌트와 힌트 사이에는 안된다
    
    ```java
    /*+ INDEX(A A인덱스) INDEX(B, B인덱스) */ -> 모두 유효
    /*+ INDEX(C), FULL(D) */ -> C만 유효
    ```
    
- FROM 절 테이블에 ALIAS를 지정했다면, 힌트에도 ALIAS를 사용해야 한다.
    
    ```java
    SELECT /*+ FULL (EMP) */ -> 유효하지 않음
    FROM EMP E
    ```
    

### 자율 or 강제

- 자율 : 주요 힌트만 지정
    
    ```java
    SELECT /*+ INDEX(A (주문일)) */
    	컬럼, 컬럼, 컬럼
    FROM 주문 A, 고객 B
    WHERE A.주문일 = ??
    	AND A.고객ID = B.고객ID
    ```
    
- 강제 : 힌트부터 조인 방식과 순서 및 엑세스 방식까지 지정
    
    ```java
    SELECT /*+ LEADING(A) USE_NL(B) INDEX(A (주문일)) INDEX(B 고객PK) */
    	컬럼, 컬럼, 컬럼
    FROM 주문 A, 고객 B
    WHERE A.주문일 = ??
    	AND A.고객ID = B.고객ID
    ```
    

자율의 경우, 옵티마이저가 가끔 실수를 하게 되면 기업에 큰 손실을 끼치기도 한다. **힌트는 빈틈없이**

### 자주 사용하는 힌트 목록

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/387d8bd4-a3e2-4ead-a565-3dc4134e3a1f)

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/e2148184-4d23-4da3-8a26-e0a686a3ba52)

# 1.2 SQL 공유 및 재사용

SQL 최적화의 복잡성을 알게되면, 동시성이 높은 온라인 트랜잭션 처리에서 바인드 변수의 중요성을 알 수 있다.

## 1.2.1 소프트 파싱 vs 하드 파싱

* 라이브러리 캐시 : SQL 파싱, 최적화, 로우 소스 생성을 재사용 하기위해 캐싱한 메모리 공간 (SGA의 요소)

* SGA : System Global Area의 약자로 서버 프로세스와 백그라운드 프로세스가 공통으로 엑세스하는 데이터와 제어 구조를 캐싱하는 **메모리 공간**

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/9dd9c358-fbc9-4a40-81f2-8a0480e5a6fa)

1. 사용자가 SQL을 파싱한다.
2. 라이브러리 캐시에 존재 여부를 확인한다.
    1. 존재하면 실행 한다. (소프트 파싱)
3. 존재하지 않으면 최적화 및 로우 소스를 생성한다. (하드 파싱)

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/1ab13769-7e00-424f-b1a8-0f6900cb4dee)

### SQL 최적화 과정이 하드한 이유

여러 실행 계획을 고려하고, 최적의 경로를 선정하는 것은 생각보다 많은 일을 수행해야 한다.

- 테이블, 컬럼, 인덱스 구조 정보
- 오브젝트 통계 - 테이블 통계, 인덱스 통계, 컬럼 통계 (히스토그램 포함)
- 시스템 통계 - CPU, Single Block I/O, Multiblock I/O 속도 등
- 옵티마이저 관련 파라미터

하드 파싱은 CPU를 많이 소비하는 몇 안되는 작업 중 하나이다. 따라서 캐싱이 없다면, 매우 아까울 것이다.

## 1.2.2 바인드 변수의 중요성

### 이름 없는 SQL 문제

함수/프로시저, 트리거, 패키지 등은 생성할 때부터 이름을 가져 딕셔너리에 저장되고, 라이브러리 캐시에 적재도 된다. 반면에 SQL은 이름이 없어 딕셔너리에 저장하지 않는다. SQL 자체가 이름이 되어 라이브러리 캐시에 적재되긴 하지만, 공간이 부족하면 버려진다.

SQL도 영구 저장하는 방법이 있을까? (IBM DB2 제외하곤 그렇지 않고있음)

SQL은 이름이 없어 조그마한 수정만 일어나도 다른 객체가 탄생한다. Oracle 10g의 SQL ID를 사용해도 이름이 매번 변화하기에 소용없다.

### 공유 가능 SQL

```java
SELECT * FROM TABLE WHERE COLUMN = ?
```

위의 쿼리에서 ?에 들어갈 단어가 매번 바뀌면 모두 다른 SQL이 되어 라이브러리 캐시에 가득차게 된다.

그렇다면 파라미터를 받는 프로시저를 만들어 재사용하면 되지 않을까?

```java
create procedure SQL (variable in varchar2) { ... }
```

이처럼 파라미터 Driven 방식으로 사용하면 SQL에 대한 하드파싱은 줄어들고 재사용하게 된다.

→ 실제 우리회사에서 로그인/아웃 관련 SQL을 프로시저로 사용중인데 이거때문일까?

→ 그럼 왜 최근 기업들은 자바로 옮기려는 행동을 보이는걸까?


# 1.3 데이터 저장 구조 및 I/O 메커니즘

**I/O 튜닝이 곧 SQL 튜닝이다.**

## 1.3.1 SQL이 느린 이유

90%는 I/O 때문이다. 정확히는 디스크 I/O

OS 또는 서비스시템이 I/O를 처리하는 동안 프로세스는 일하지 않고 잠을 잔다.

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/3a77180e-2806-4194-8e29-a443e6daa18e)

I/O를 처리하는 동안 프로세스는 wait queue에 들어가게 되어 전체적인 처리속도(성능)가 느려진다.

 I/O call 속도

- Single Block : 평균 10ms
- SAN 스토리지 : 4~8ms
- SSD : 1~2ms

## 1.3.2 데이터베이스 저장 구조

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/eda20dea-7d4f-47dc-b93a-9bee888968c8)

테이블 스페이스 : N개의 세그먼트가 있으며, 여러 개의 데이터파일로 구성된다.

세그먼트

- N개의 익스텐트가 있으며, 테이블, 인덱스 처럼 데이터 저장공간이 필요한 오브젝트이다.
- 파티션 구조가 아니라면 테이블도 하나의 세그먼트이고, 인덱스도 하나의 세그먼트이다.
- 파티션 구조라면 각 파티션이 하나의 세그먼트가 된다.
- LOB 컬럼은 자체가 하나의 세그먼트를 구성해서 별도 공간에 값을 저장한다.

익스텐트 : 연속된 블록의 집합이며, 익스텐트 단위로 공간을 확장한다. / 한 테이블이 독점한다.

데이터 블록 : 사용자가 입력한 레코드를 실제로 저장하는 공간 이다.(page라고도 불린다)

세그먼트 공간이 부족해지면 테이블스페이스로부터 익스텐트를 추가로 할당받는다.

세그먼트에 할당된 모든 익스텐트가 같은 데이터파일에 위치하지 않을 수 있는데, DBMS가 파일 경합을 줄이기 위해 가능한 여러 파일로 분산하기 떄문이다.

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/9bf8d17a-490f-452d-a5eb-24d243360184)

익스텐트 내에선 연속되지만, 익스텐트끼리는 연속된 공간이 아니다.

오라클에서 세그먼트에 할당된 익스텐트 목록을 조회하는 방법

```sql
SELECT segment_type, tablespace_name, extent_id, file_id, block_id, blocks
FROM dba-extents
WHERE owner = USER
	AND segment_name = 'your_segment_name'
ORDER BY extent_id;
```

**Data Block Address**

- 모든 데이터 블록은 디스크 상에서 몇 번 데이터파일의 몇 번째 블록인지 본인의 주소값을 갖는다. 이를 DBA라고 한다.
- 인덱스를 이용해 테이블 레코드를 읽을 때 ROWID를 이용한다.
- 테이블 스캔은 테이블 세그먼트 헤더에 저장된 익스텐트 맵을 사용한다. 맵에선 첫 번째 익스텐트의 블록 DBA를 알 수 있는데, 첫 번째로부터 연속해서 블록을 읽으면 된다.

* ROWID : DBA + 블록 내 로우 번호

**용어 정리**

- 블록 : 데이터를 읽고 쓰는 단위
- 익스텐트 : 공간을 확장하는 단위, 연속된 블록 집합
- 세그먼트 : 데이터 저장공간이 필요한 오브젝트(테이블, 인덱스, 파티션, LOB 등)
- 테이블 스페이스 : 세그먼트를 담는 컨테이너
- 데이터파일 : 디스크 상의 물리적인 OS 파일

## 1.3.3 블록 단위 I/O

데이터 I/O 단위는 한 블록이다. 만일 내가 한 레코드를 읽고 싶어도, 한 블록을 통째로 읽는다.

테이블뿐만 아니라 인덱스도 블록 단위로 데이터를 읽고 쓴다. 

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/b4f950c6-b74f-4795-a742-450c13146fee)

블록 사이즈 쿼리

```sql
show parameter block_size -- 오라클은 보통 8KB
```

## 1.3.4 시퀀셜 엑세스 vs 랜덤 엑세스

시퀀셜 엑세스 : 논리 또는 물리적으로 연결된 순서에 따라 차례대로 블록을 읽는 방식

- 인덱스 리프 노드가 논리적으로 연결되어 있어 순차적으로 블록을 읽는다.
- 테이블 블록은 논리적 연결고리가 없어, 익스텐트 목록을 Map으로 관리하여 첫 번째 블록에 접근하고 순차적으로 접근한다. 이를 Full Table Scan이라고 한다.

랜덤 엑세스 : 레코드 하나를 읽기 위해 한 블록씩 접근하는 방식

- 주로 인덱스에서 테이블 블록에 접근할 때 사용된다.

## 1.3.5 논리적 I/O vs 물리적 I/O

### DB 버퍼캐시

디스크 I/O를 줄이기 위해 DB 버퍼 캐시도 존재한다. 데이터 블록을 캐싱해두는 목적이다.

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/9d1e9a03-0f0f-42e9-8377-bde497703716)

데이터 블록을 읽을 땐 항상 버퍼캐시부터 탐색하고, 디스크 I/O후에 버퍼 캐시에 적재한다

```sql
show sga; 
```

### 논리적 I/O vs 물리적 I/O

논리적 블록 I/O : SQL을 처리하는 과정에 발생한 총 블록 I/O 횟수 (메모리 I/O + DirectPath I/O)

물리적 블록 I/O : 디스크에서 발생한 총 블록 I/O (버퍼 non hit I/O)

위 둘의 속도는 약 10,000배 가량 차이나서 상당한 속도저하를 야기한다.

![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/58a1c881-089c-49ce-a789-5b6799e810d8)

### 왜 논리적 I/O?

논리적 I/O = SQL을 수행하며 읽은 총 블록 I/O

### 버퍼캐시 히트율 (BCHR)

히트율 = 캐시에서 찾은 블록 수 / 총 읽은 블록 수

= (논리적 I/O - 물리적 I/O) / 논리적 I/O

= (1 - 물리적 I/O) / 논리적 I/O

물리적 I/O = 논리적 I/O * (100% - BCHR) 이다.

물리적 I/O를 줄이는게 SQL 실행 시간을 줄이는 방법인데, 이를 위해선 BCHR를 늘리거나, 논리적 I/O를 줄여야 한다.

논리적 I/O는 SQL 튜닝을 통해 읽는 총 블록 개수를 줄이면 된다. 그러면 동시에 물리적 I/O도 줄게 될 것이다.

BCHR이 높다고 항상 좋은 것은 아니다. 같은 블록을 비효율적으로 반복해서 읽는 경우가 있어서다.

## 1.3.6 Single Block I/O vs Multiblock I/O

Single Block I/O : 한 번에 한 블록씩 요청해서 메모리에 적재하는 방식

- 인덱스를 I/O할 때
- 인덱스를 사용한 테이블 블록 I/O할 때

Multiblock I/O : 한 번에 여러 블록씩 요청해서 메모리에 적재하는 방식

- 인덱스 루트 블록을 읽을 때
- 인덱스 루트 블록에서 얻는 주소로 브랜치 블록을 읽을 때
- 인덱스 브랜치 블록에서 얻은 주소로 리프 블록을 읽을 때
- 인덱스 리프 블록에서 얻은 주소 정보로 테이블 블록을 읽을 때
- 풀 테이블 스캔

Multiblock I/O를 할 땐 한 번에 가능한 많은 블록을 가져오는게 좋다. (프로세스가 놀지 않기 위해)

또한 Multiblock I/O는 I/O 콜을 할 때 인접한 블록들을 한꺼번에 읽어 캐시에 미리 적재하는 기능이다. (보통 1MB단위이며 db_file_multiblock_read_count 파라미터로 정한다.)

예시) OS 레벨 I/O가 1MB, 오라클 레벨 I/O 단위가 8KB이면 8KB * 128 = 1MB 이므로 128개의 블록을 담게 된다. 단, 익스텐트의 경계를 넘지 못한다.

간혹 Multiblock I/O 중간에 Single Block I/O가 나타나는데, 이는 캐싱되지 않은 블록이 사이에 있을 때 그렇다.

## 1.3.7 Table Full Scan vs Index Range Scan

Table Full Scan : 테이블 전체를 스캔하는 방식 (테이블에 속한 블록 전체를 읽는다)

Index Range Scan : 인덱스를 이용한 테이블 엑세스 (ROWID로 테이블 레코드를 찾는다)

쿼리튜닝을 할 때 Table Full Scan이 항상 안좋은 것은 아니다. (배치에서 좋은 경우도 있음)

왜? → 대량 검색은 시퀀셜 엑세스 + Multiblock I/O가 더 효율적이기 때문이다. (랜덤 엑세스 + Single Block I/O 보단 빠름)

요약하자면, 대량 검색엔 Table Full Scan이, 소량 검색엔 Index Range Scan이 더 효율적이다.

## 1.3.8 캐시 탐색 메커니즘

Direct Path I/O를 제외한 모든 블록 I/O는 메모리 버퍼캐시를 경유한다.

- 인덱스 **루트 블록**을 읽을 때
- 인덱스 **루트 블록에서** 얻은 주소로 정보로 **브랜치 블록**을 읽을 때
- 인덱스 **브랜치 블록에서** 얻은 주소 정보로 **리프 블록**을 읽을 때
- 인덱스 **리프 블록**에서 얻은 주소로 **테이블 블록**을 읽을 때
- 테이블 블록을 **Full Scan**할 때

### 버퍼캐시 탐색 메커니즘

- 해시 함수 모듈러
    
    ![image](https://github.com/Deep-Dive-Study/sql-tuning/assets/85796588/a0e49e07-be1e-462d-8ee6-1982e375957b)
    
    모듈러 연산을 통해 버킷을 찾고, 리스트를 탐색한다. 없으면 메모리에 없는 것
    

### 메모리 공유자원에 대한 엑세스 직렬화

버퍼캐시는 SGA의 요소여서 모두 공유자원이다. 공유자원은 말 그대로 모두에게 권한이 있기 때문에 누구나 접근할 수 있다.

문제는 여기서 발생한다. 동시성 문제가 발생하는데, 직렬화가 필요하다.

SGA의 서브 캐시마다 별도의 래치가 존재하는데, 캐시버퍼 체인 래치, 캐시버퍼 LRU 체인 래치등이 있다.

래치에 의한 경합이 잦으면 캐시 I/O도 느려질 수 있다.

캐시버퍼가 아니라 버퍼 블록에는 버퍼Lock이 직렬화를 한다. 

버퍼 락은 캐시버퍼 체인 래치를 해제하기 전에 헤더에 Lock을 설정하여 버퍼 블록 자체에 대한 직렬화 문제를 해결한다.

결론적으로 캐시경합을 줄이기 위해선 앞에서도 언급했던 논리적 I/O 자체를 튜닝을 통해 줄여야 한다.
