# 6장 소트 튜닝

## 6.1 기본 DML 튜닝
- 지금까지 배운 인덱스와 조인 튜닝을 DML 문에도 그대로 적용이 가능함
- 다만, DML 성능에 영향을 주는 다른 요소와 튜닝 방법도 따로 있음

### 6.1.1 DML 성능에 영향을 미치는 요소
- DML 성능에 영향을 미치는 요소
  - 인덱스
  - 무결성 제약
  - 조건절
  - 서브쿼리
  - Redo 로깅
  - Undo 로깅
  - Lock
  - 커밋 

#### 인덱스
- 인덱스가 늘어나면 Wirte(Insert, Update, Delete) 작업 시 부하가 늘어남
  - 특히 Update 시에는 변경과 인덱스 수조 내에서 정렬을 다시해야 하는 이중적인 부하가 발생함

    ![Google Chrome 2024-09-28 02 20 13](https://github.com/user-attachments/assets/dcca53aa-83e0-4414-b471-bd532d0d0a85)

    ![Google Chrome 2024-09-28 02 20 31](https://github.com/user-attachments/assets/00fa08c5-1bdf-48d6-ba42-ea2ad9b8012c)

- 불필요한 인덱스는 제거하자.
  - 100만 건 삽입시 8배 차이 

#### 무결성 제약
- DB 에는 논리적으로 의미 있는 자료만 저장되게 하는 데이터 무결성 규칙이 있음
  - 개체 무결성
  - 참조 무결성
  - 도메인 무결성
  - 사용자 정의 무결성 

- 이러한 규칙은 어플리케이션으로 구현 가능하지만, DBMS에서 PK, FK, Check, Not Null 같은 제약(Constraint)을 설정하면 더 완벽하게 데이터 무결성을 지킬 수 있음
- 무결성 제약 중 PK/FK 제약은 Check / Not Null 보다 성능에 더 큰 영향을 줌
  - 해당 제약은 실제 데이터를 조회해봐야 알기 때문 

- 예시 테스트 결과

  ![Goodnotes 2024-09-28 02 24 02](https://github.com/user-attachments/assets/154dc939-90de-49eb-a37b-32b8106af39e)

#### 조건절
- SELCT 문과 실행 계획이 다르지 않기 때문에, 이들 DML 문에는 2, 3장에서 학습한 인덱스 튜닝 원리를 그대로 적용 가능

  ![CleanShot 2024-09-28 at 02 25 23](https://github.com/user-attachments/assets/b704aec1-2a48-4bf6-b43d-c59d5d234149)

#### 서브쿼리
- 이또한, SELCT 문과 실행 계획이 다르지 않기 때문에, 이들 DML 문에는 4장에서 학습한 조인 튜닝 원리를 그대로 적용 가능(서브쿼리 조인 - 4장 4절)

  ![CleanShot 2024-09-28 at 02 25 44](https://github.com/user-attachments/assets/0c1ac8d6-cd49-48a3-9bf4-7e697b07848b)

#### Redo 로깅
- 오라클은 데이터 파일과 컨트롤 파일에 가해지는 모든 변경 사항을 Redo 로그에 기록함
  - 트랜잭션 데이터가 유실된 경우, 트랜잭션을 재현하여 유실 이전 상태로 복구하는데 사용됨
- DML 수행 시마다 Redo 로그 생성해야하기 때문에 DML에 성능에 영향을 끼침
  - INSERT 작업에 대해 Redo 로깅 생략 기능을 제공하는 이유 

#### Undo 로깅
- Redo와 반대로 Undo는 트랜잭션을 롤백함으로써 현재를 과거 상태로 되돌리는데 사용

  ![CleanShot 2024-09-28 at 02 30 08](https://github.com/user-attachments/assets/cd30be0f-b5a6-4996-a218-d477d7a7cf06)

- DML 수행 시마다 Undo 로그 생성하기 때문에 DML에 성능에 영향을 끼침
  - Undo를 생략하는 기능은 제공하지 않음(반드시 생성이 필요함)
  - 책에 상세한 내용이 나오기 때문에 참고하기 바람(MVCC, 등)

#### Lock
- Lock을 필요 이상으로 자주, 길게 사용하거나 레벨을 높일 수록 DML 성능은 느려짐
  - 즉, 성능과 데이터품질은 트레이드오프 관계

#### 커밋
- DML을 끝내려면 커밋까지 완료해야함으로 서로 밀접한 관련이 있음
  - 특히 DML이 Lock에 의해 블록킹(Blocking) 된 경우, 커밋은 성능과 직결됨

- 커밋은 결코 가벼운 작업이 아님(DBMS에서 Fast Commit을 구현하여 빠르게 처리될뿐)

- 트랜잭션 저장 과정

  ![CleanShot 2024-09-28 at 02 36 44](https://github.com/user-attachments/assets/528d331e-1a8d-4395-9e46-ade7c62181a2)
 
- DB 버퍼캐시
  - 서버 프로세스(DBWR)는 버퍼캐시를 통해 데이터를 일괄 처리한다
- Redo 로그버퍼
  - Redo 성능을 위해 Redo 파일 중간에 버퍼 위치(LGWR)
- 트랜잭션 데이터 저장 과정
  - DML 문을 실행하면 Redo 로그버퍼에 변경사항 기록
  - 버퍼블록에서 데이터 변경
  - 커밋
  - Redo 로그버퍼 파일에 일괄 저장
  - 버퍼블록을 데이터파일에 일괄 저장
- 커밋
  - 커밋을 오래동안 안하는 것도 안좋지만 너무 자주하는 것도 IO를 발생시키기 때문에 좋지 않다

### 데이터베이스 Call과 성능
- SQL Call 세 단계
  - Parse call: SQL 파싱과 최적화 수행하는 단계
  - Execute call: SQL 실행하는 단계
  - Fetch call: 데이터를 읽어서 사용자에게 결과집합 전송 과정(전송할 데이터가 많은 경우 여러번의 call 발생)

    ![CleanShot 2024-09-28 at 02 39 15](https://github.com/user-attachments/assets/87630aa9-d8e0-4a41-a5ec-c782be78b036)

- 발생 시점에 따른 Call 분류
  - User call: DBMS 외부로부터 인입되는 call(WAS)
  - Recursive call: DBMS 내부에서 발생하는 call(함수/프로시저/트리거, 등)

   ![CleanShot 2024-09-28 at 02 40 30](https://github.com/user-attachments/assets/7607d210-ec44-4162-82e9-9d51a2fe8dee)

  - User call 이든 Recursive call이든 SQL을 실행할때 위의 3단계의 call을 거침
  - call이 많은 경우 성능에 안좋음. 특히 네트워크를 경유하는 User Call이 성능에 미치는 영향은 매우 큼

- 절차적 루프 처리
  - 절차적으로 반복되는 call일 경우 그나마 recursive 콜일 경우는 나음
  - user call은 느려질 수밖에 없음
  - call 한번 당 커밋을 하게되면 성능이 훨씬 느려짐(너무 오래 커밋을 안하는 것도 문제지만 너무 자주해도 안 좋음)

- One SQL의 중요성
  - 로직이 복잡하지 않으면 되도록 하나의 SQL로 끝나는 게 성능에 좋음
  - One SQL 구현시 유용한 구문 활용법
    - Insert Into Select
    - 수정 가능 조인 뷰
    - Merge 문

### 6.1.3 Array Processing 활용
- 복잡한 업무 로직을 One SQL로 구현하는 것은 어려움, 이럴 때 Array Processing을 활용하면 One SQL로 구현하지 않고도 Call 부하를 획기적으로 줄일 수 있음

 ![CleanShot 2024-09-28 at 02 48 01](https://github.com/user-attachments/assets/d5ee112b-b2c4-4539-8f5e-b81dfb6466e0)
 
 ![CleanShot 2024-09-28 at 02 48 19](https://github.com/user-attachments/assets/33a4cd75-1f72-477e-93a5-05297e17ca97)

### 6.1.4 인덱스 및 제약 해제를 통한 대량 DML 튜닝
- 동시 트랜잭션이 없는 배치 환경에서는 PK 및 인덱스 제약을 해제를 통하여 성능을 향상 시킬 수 있음

### 6.1.5 수정가능 조인 뷰
- 다른 테이블과 조인이 필요할 때 전통적인 UPDATE 문을 사용하면 비효율을 완전히 해소할 수 없음
- 인라인 뷰가 포함된 update 쿼리문은 튜닝을 통하여 속도 향상이 가능함

### 6.1.6 Merge 문 활용
- DW(데이터 웨어하우스)에서는 두 시스템 간 데이터 동기화가 가장 흔히 발생하는 작업
- 이때 오라클9i에서 도입된 MERGE 문을 활용할 수 있음(DW 시스템으로 적재 - Loading)

## 6.2 Direct Path I/O
- 온라인 트랜잭션(OLTP)은 기준성 데이터, 특정 고객, 특정 상품, 최근 거래 등을 반복적으로 읽는 경우가 많기 때문에 버퍼 캐시가 성능 향상에 도움이됨.
- 반면에, **정보계 시스템(DW/OLAP)이나 배치 작업에서는 버퍼캐시 경유하지 않는 것이 성능이 더 좋을 수 있음**
  - 대량의 데이터를 처리하기 때문에 버퍼 캐시를 경유하는 I/O 메커니즘이 역효과를 줌
- 그래서 오라클은 버퍼 캐시를 경유하지 않고 바로 데이터 블록을 읽고 쓰는 기능을 제공함. 이를, **`Direct Path I/O`** 라고 함

### 6.2.1 Direct Path I/O
- 일반적인 블록 I/O는 버퍼 캐시를 경유해서 데이터를 처리함(변경시에도) 
- 자주 읽는 블록에 대한 반복적인 I/O Call을 줄여주기 때문에 일반적인 경우에는 성능에 도움이 됨
- 다만, 대량의 데이터를 읽고 쓸 때 건마다 버퍼 캐시를 탐색한다면 개별 프로그램 성능에는 오히려 성능 저하의 원인이됨
  - 대량 블록을 건건이 디스크로부터 버퍼 캐시에 적재하고 나서 읽어야 하는 부담도 큼
  - 또한, 적재한 블록을 재사용할 가능성 또한 낮은 편, 이러한 데이터가 버퍼 캐시를 점유하면 다른 프로그램 성능에도 나쁜 영향을 줌 

- 오라클에서는 이러한 경우에 버퍼 캐시를 경유하지 않고 곧바로 데이터 블록을 읽고 쓸 수 있는 **`Direct Path I/O`** 기능을 제공함

- 아래 경우에 사용함

1. 병렬 쿼리로 Full Scan울 수행 할 때
2. 병렬 DML을 수행할 때
3. Direct Path Insert를 수행할 떄
4. Temp 세그먼트 블록들을 읽고 쓸 때
5. direct 옵션을 지정하고 export를 수행할 때
6. nocache 옵션을 지정한 LOB 컬럼을 읽을 때

### 6.2.2 Direct Path Insert
- 일반적인 INSERT가 느린 이유(동작 방식)
  - 데이터를 입력할 수 있는 블록을 Freelist에서 찾는다.
    - 테이블 HMW(High-WaterMark) 아래쪽에 있는 블록 중 데이터 입력이 가능한(여유 공간이 있는) 블록을 목록으로 관리하는데, 이를 Freelist 라고함
  - Freelist에서 할당 받은 블록을 버퍼캐시에 적재한다.
  - 버퍼캐시에 없으면, 데이터 파일에서 읽어 버퍼캐시에 적재한다.
  - INSERT 내용을 Undo 세그먼트에 기록한다.
  - INSERT 내용을 Redo 로그에 기록한다.

- Direct Path Insert 방식을 사용하는 경우 대량 데이터를 일반적인 INSERT 보다 빠르게 입력이 가능함
  - **`입력 방법`**
    - INSERT ... SELECT 문에 append 힌트 사용
    - parallel 힌트를 이용해 병렬 모드로 INSERT
    - direct 옵션을 지정하고 SQL*Loder(sqlldr)로 데이터 적재
    - CATS(create table ... as select) 문 수행
 
  - **`빠른 이유`**
    - Freelist를 참조하지 않고 HWM 바깥 영역에 데이터를 순차적으로 입력한다.
    - 블록을 버퍼캐시에서 탐색하지 않는다.
    - 버퍼캐시에 적재하지 않고, 데이터파일에 직접 기록한다.
    - Undo 로깅을 안한다.
    - Redo 로깅을 안하게 할 수 있다.

- Direct Path Insert 방식을 사용할 때 주의할 점
  - **`성능은 빨라지지만 Exclusive 모드 TM Lock이 걸린다.`**
    - 따라서, 커밋하기 전까지 다른 트랜잭션은 해당 테이블에 DML을 수행하지 못함(트랜잭션이 빈번한 주간 운영 중에는 사용X)
    - Lock 에 대해서는 DML 테이블 Lock 부분에서 다시 설명
 
  - **`Freelist를 참조하지 않고 HWM 바깥 영역에 입력하므로 테이블에 여유 공간이 있어도 재활용하지 않는다.`**
    - 따라서, 이 방식으로만 계속 INSERT하는 테이블은 사이즈가 줄지 않고 계속 늘어만 가기 때문에 주의 필요
    - Range 파티션 테이블이면 과거 데이터를 삭제할때 DELETE 대신 DROP 방식으로 지워야 공간 반환이 제대로 이루어짐(일반 INSERT를 사용하더라도)
    - 비 파티션 테이블이면 주기적으로 Reorg 작업을 수행해주어야함 

### 6.2.3 병렬 DML
- INSERT의 경우에는 append 힌트를 통해 Direct Path Write 방식으로 유도가 가능
- 반면에, UPDATE, DELETE 는 기본적으로 Direct Path Write가 불가능하기 때문에 병렬 DML 로 유도해야함
  - 병렬 처리는 대용량 데이터가 전제이므로 오라클은 병렬 DML에 항상 Direct Path Write 방식을 사용함
 
- DML을 병렬로 처리하려면, 병렬 DML 기능을 활성화 해야함
  - 힌트에는 기술하고, 병렬 DML 기능을 비활성화 한 경우는 대상 레코드를 찾는 작업은 병렬로 진행하지만, 추가/변경/삭제는 QC가 혼자 담당하므로 병목이 생김
    - QC란, Query Coordinator 의 줄임말로, 최초 DB 접속해서 SQL을 수행한 프로세스  

  ```sql
  
  alter session enable parallel dml;

  ```

  ![CleanShot 2024-09-28 at 01 59 12](https://github.com/user-attachments/assets/ecaac8ab-f735-457b-9a83-afa361244f6d)

- 병렬 INSERT는 append 힌트를 지정하지 않더라도 Direct Path Insert 방식을 사용함
  - 하지만, 병렬 DML이 작동하지 않을 경우를 대비하여 아래와 같이 append 힌트를 같이 사용하는게 좋음(혹시 모를 사태에도 일정 성능 보장 가능) 

    ![CleanShot 2024-09-28 at 02 13 30](https://github.com/user-attachments/assets/2c1c533b-5ca4-45bd-b686-63609df15229)
 
- 오라클 12c부터는 enable_parallel_dml 힌트도 지원함

  ![CleanShot 2024-09-28 at 02 14 13](https://github.com/user-attachments/assets/78c8b7b9-ca05-420c-8f38-1ec02907dc82)

- 다만, **`병렬 DML도 Exclusive 모드 TM Lock이 걸리기 때문에 주의가 필요함`**

- 병렬 DML이 잘 실행되는지 실행 계획을 통해 확인 가능함
  - 정상 작동 중

    ![CleanShot 2024-09-28 at 02 00 49](https://github.com/user-attachments/assets/ab74468d-10d0-4bb1-acd0-35a3bac7e8c4)

  - 비정상 작동(병렬 처리X - QC 처리)
    
    ![CleanShot 2024-09-28 at 02 01 12](https://github.com/user-attachments/assets/1144b9c5-73ac-47ae-8522-ea04c004b6aa)
