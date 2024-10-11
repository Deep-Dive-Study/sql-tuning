# 7.1 통계정보와 비용 계산 원리

## 7.1.1 선택도와 카디널리티

* 선택도 : 전체 레코드 중에서 조건절에 의해 선택되는 레코드 비율을 말한다. (1/NDV)

* 카디널리티 : 전체 레코드 중에서 조건절에 의해 선택되는 레코드 개수. (총 로우수/NDV)

한 컬럼에 값이 가, 나, 다, 라 네가지가 있을 때 한 컬럼에 대한 선택도는 25%, 카디널리티는 4/레코드 수 이다.

옵티마이저는 이렇게 카디널리티를 구하고 비용을 계산해서 테이블 액세스, 조인 방식 등을 결정한다.

선택도를 잘못 계산하면, 카디널리티와 비용도 잘못 계산하고, 결과적으로 비효율적인 액세스 방식과 조인 방식을 선택하게 된다.

## 7.1.2 통계정보

### 1. 테이블 통계

```sql
begin
	dbms_stats.gather_table_stats('scott', 'emp');
end;
```

![image](https://github.com/user-attachments/assets/e70151c2-d5cb-4a52-a592-c0f9eecb377e)

### 2. 인덱스 통계

```sql
begin
	dbms_stats.gather_index_stats ( ownname => 'scott', indname => '이름' );
end;
```

![image](https://github.com/user-attachments/assets/83a7e073-de35-4aa6-bee3-046c7ed3ea75)

### 3. 컬럼 통계

컬럼 통계는 테이블 통계시에 같이 수집된다.

![image](https://github.com/user-attachments/assets/e1c6da86-401d-4e1d-945e-e4e5b4331e29)

### 컬럼 히스토그램

= 조건에 대한 선택도는 1/NUM_DISTINCT 공식으로 구하거나, DENSITY 값을 이용하면 된다.

선택도를 잘못 구하면 데이터 액세스 비용을 잘못 산정하게 되고, 결국 최적이 아닌 실행계획으로 이어진다.

![image](https://github.com/user-attachments/assets/7bc4a12a-9abb-466c-8ca7-f857125a1a12)

```sql
begin
	dbms_stats.gather_table_stats('scott', 'emp', cascade=>false
		, method_opt=>'for columns ename size 10, deptno size 4');
end;
```

### 4. 시스템 통계

애플리케이션 및 하드웨어 성능 측정 표이며 주로 다음과 같은 지표가 있다.

- CPU 속도
- Single Block I/O 속도
- Multiblock I/O 속도
- block I/O 개수
- I/O 서브시스템 최대 처리량
- 병렬 Slave 평균 처리량

## 7.1.3 비용 계산 원리

비용모델은 I/O, CPU등으로 나뉘는데, 어떤 모델이냐에 따라 I/O콜 횟수, Single, Multiblock의 상대적인 수치일 수 있다.

- 인덱스 키값을 모두 = 조건으로 검색할 때
    - 비용 = BLEVEL + AVG_LEAF_BLOCKS_PER_KEY + AVG_DATA_BLOCKS_PER_KEY
- 인덱스 키값이 모두 = 조건이 아닐 때
    - 비용 = BLEVEL + LEAF_BLOCKS * 유효 인덱스 선택도 + CLUSTERING_FACTOR * 유효 테이블 선택도
