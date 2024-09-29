# 6.1 기본 DML 튜닝

## 6.1.1 DML 성능에 영향을 미치는 요소

- 인덱스
- 무결성 제약
- 조건절
- 서브쿼리
- Redo 로깅
- Undo 로깅
- Lock
- 커밋

### 인덱스와 DML 성능

테이블에 레코드를 입력/삭제/수정 하면 인덱스에도 입력해야 한다. 

- 테이블은 Freelist를 통해 입력할 블록을 할당받는다.
- 인덱스는 수직접 탐색을 통해 입력할 블록을 찾는다.

### 무결성 제약과 DML 성능

무결성 규칙

- 개체 무결성 (PK NULL 금지 및 중복 금지)
- 참조 무결성 (FK NULL or 기본키에 존재해야함)
- 도메인 무결성 (데이터는 정한 범위 내에 속해야함)
- 사용자 정의 무결성 (업무 제약 조건)

### 조건절과 DML 성능

DELETE, UPDATE 도 SELECT과 동일한 실행계획이 적용된다.

### 서브쿼리와 DML 성능

DELETE, UPDATE 도 SELECT과 동일한 실행계획이 적용된다.

### Redo 로깅과 DML 성능

Redo 로그 : 데이터와 컨트롤 파일에 가해지는 모든 변경사항을 기록하는 것.

(Insert를 할 때 생략하기도 한다.)

Redo 로그의 목적

- DB 복구
- 캐시 복구
- 빠른 커밋 (버퍼블록에만 반영해도 Redo 로그를 믿고 커밋처리)

### Undo 로깅과 DML 성능

Redo : 트랜잭션을 재현해서 과거 상태로 되돌린다.

Undo : 트랜잭션을 롤백해서 과거 상태로 되돌린다.

DML을 수행할 때마다 Undo를 생성해야 하므로 성능에 악영향을 미친다.

**Undo**

- 오라클은 데이터 Insert/Update/Delete 할 때마다 Undo 세그먼트에 기록을 남긴다.
- 해당 트랜잭션이 커밋되면 다른 트랜잭션이 재사용할 수 있는 상태로 바뀐다.
- 가장 오래된 트랜잭션 순으로 Overwritten 된다.

**Undo 데이터의 목적**

- Transaction Rollback
- Transaction Recovery (시스템 셧다운 복구)
- Read Consistency

**MVCC**

Current 모드 : 디스크에서 캐시로 적재된 원본 상태를 그대로 읽는 방법

Consistent 모드 : 쿼리가 시작된 이후 다른트랜잭션에 의해 변경된 블록을 만나면 복사본을 만들어 Undo 데이터를 적용해 되돌려 읽는다. (SCN 모드)

- SELECT는 항상 Consistent 모드로 읽는다.
- DML은 Consistent 모드로 대상 레코드를 찾고 Current 모드로 변경한다.

### Lock과 DML 성능

Lock을 너무 많이 사용하면 성능이 느려지고, 너무 적게 사용하면 데이터 품질이 나빠진다.

성능과 데이터 품질 모두 얻으려면 **동시성 제어**를 해야한다.

### 커밋과 DML 성능

Commit을 해야 Lock이 해제되어 DML 성능과 밀접하다고 볼 수 있다.

1. **DB 버퍼캐시**
    - 버퍼캐시를 통해 데이터를 읽고 쓴다.
    - 버퍼캐시를 일괄적용한다. (DBWR 프로세스)
2. **Redo 로그버퍼**
    - DBWR 프로세스가 데이터파일에 반영할 때 까지 Redo로그에 기록한다.
    - Redo 로그는 버퍼를 사용하는데, 이후 LGWR 프로세스가 버퍼를 Redo 파일에 일괄적용한다.
3. **트랜잭션 데이터 저장 과정**
    
    ![image](https://github.com/user-attachments/assets/bced2a10-8692-4915-a88b-b28d9f472c6b)
    
    1. DML을 실행하면 Redo 로그버퍼에 변경사항을 기록한다.
    2. 버퍼블록에서 데이터를 변경한다. (버퍼에 없으면 데이터파일에서 검색)
    3. 커밋
    4. LGWR가 Redo 로그버퍼 내용을 일괄 저장한다.
    5. DBWR 프로세스가 변경된 버퍼블록은 데이터파일에 일괄 저장한다.
    
    오라클은 데이터 변경 전에 항상 로그를 기록한다.
    
    메모리 버퍼캐시가 휘발성이어서 Redo 로그를 LGWR 프로세스가 커밋 전엔 데이터파일에 저장한다. (휘발되지 않게)
    
4. **커밋 = 저장버튼**
5. 커밋은 Disk I/O이기 때문에 너무 잦거나, 너무 자주 안해도 안좋다.
    
    

## 6.1.2 데이터베이스 Call과 성능

SQL Call 단계

- Parse Call : SQL 파싱과 최적화를 수행하는 단계 (캐시 사용하기도 함)
- Execute Call : SQL 실행 단계 (select는 fetch를 한다)
- Fetch Call : 데이터를 읽어 결과집합 전송 (select만 발생)

Call의 발생위치에 따라 종류를 나눌 수 있다.

- User Call : 네트워크를 통해 DBMS 외부로부터 인입되는 Call (사용자, WAS)
- Recursive Call : DBMS 내부에서 발생하는 Call (프로시저, 함수, 트리거 등)

**절차적 루프 처리**

- 네트워크를 경유하지 않는 Recursive Call는 그나마 빠르다.
- User Call은 네트워크를 경유하기 때문에 더 느리다.

**One SQL의 중요성**

- 절차적 루프 처리한 쿼리를 하나의 쿼리로 수행하면 엄청 빠르다.
- 아래 방법을 잘 사용하자
    - Insert Into Select
    - 조인 뷰
    - Merge

## 6.1.3 Array Processing 활용

복잡합 업무 로직을 포함할 때 Array Processing 기능을 활용해서 Call 부하를 줄일 수 있다.

```jsx
declare
	cursor 선언
	type 선언
	source 선언
	
	array size 선언
	
	procedure insert_target (변수) is
	begin
		...
	end
	
	begin
		...
	end
	
	commit;
end;
```

Array Processing은 Recursive, User Call 모두 성능적인 효과를 받을 수 있다.

## 6.1.4 인덱스 및 제약 해제를 통한 대량 DML 튜닝

인덱스와 무결성 제약조건은 DML 성능에 큰 영향을 끼친다.

배치 프로그램에서는 이 기능을 해제함으로써 큰 성능개선 효과를 얻을 수 있다.

인덱스와 무결성 제약조건을 Unusable 하면 속도가 매우 빨라진다.

`skip_unusable_indexs` 파라미터를 true로 해둬야 한다.

이후 다시 제약조건을 Usable할 때 제약조건에 대한 Validate를 자동 수행하는데 끌 수 도 있다.

## 6.1.5 수정가능 조인 뷰

```jsx
UDATE 테이블
SET A = (select ~ from ~)
	, B = (select ~ from ~)
	, C = (select ~ from ~)
;

UDATE 테이블
SET (A, B, C) = (select ~, ~, ~ from ~)
;
위 쿼리를 아래처럼 조회쿼리를 줄이는 튜닝을 할 수 있다.

쿼리를 줄일 때, 같은 값으로 갱신되는 비중이 높을수록 Redo 로그 발생량이 증가해 오히려 비효율적일 수 있다.
(WHERE 절이 안들어간 전통 UPDATE문은 해당 문제가 없다.)
```

### 수정가능 조인 뷰

조인 뷰 : FROM 절에 두 개 이상의 테이블을 가진 뷰

수정가능 조인 뷰 : 입력, 수정, 삭제가 허용되는 조인 뷰

수정가능 조인 뷰를 사용하면 참조 테이블과 두 번 조인하는 비효율을 없엘 수 있다.

```jsx
UPDATE
(selecct ~, ~, ~
from (select ~, ~, ~
			from ~
			where ~
			group by~ ) t
where ~
)
set ~ = ~
	, ~ = ~
	, ~ = ~
;
```

1쪽 집합에 PK, Unique 제약을 설정하지 않으면 수정가능 조인 뷰를 통한 입력/수정/삭제가 불가능하다.

### 키 보존 테이블

조인된 결과집합을 통해서 중복 값 없이 Unique하게 식별이 가능한 테이블

Unique한 1쪽 집합과 조인되는 테이블이어야 식별 가능하다.

키 보존 테이블 : 뷰에 rowid를 제공하는 테이블

### ORA - 01779 오류 회피

11g 이하 버전에서 Group By 한 컬럼키가 보존될 때 옵티마이저가 불필요한 제약을 가해 에러가 발생한다.

10g에서는 bypass_ujvc 힌트를 사용해 회피가 가능하다.

11g는 수정가능 조인 뷰를 사용해 회피가 가능하다.

## 6.1.6 MERGE문 활용

두 시스템 간 데이터 동기화를 위해 MERGE문이 생겼다.

1. 전일 발생한 변경 데이터를 기간계 시스템으로부터 추출
2. 1에서 만든 테이블을 DW 시스템으로 전송
3. DW 시스템으로 적재

![image](https://github.com/user-attachments/assets/c9c50f0f-281d-4a7c-aa57-1d99c37a2899)

- UPDATE/INSERT를 선택적으로 처리할 수 있다.
- ON절에 기술한 조인문 외에 추가 조건절을 기술할 수 있다.
- 이미 저장된 데이터를 조건에 따라 지울 수 있다.

MERGE문은 데이터가 존재하면 UPDATE 없으면 INSERT할 때 SQL 수를 한 번만 날리도록 고정한다.
