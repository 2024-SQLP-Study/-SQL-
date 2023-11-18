# SQL 처리 과정과 I/O

## 세줄 요약(세줄 아님)
- SQL 기능은 절차적일 수밖에 없는데, 이를 최적화 해주는 것이 SQL 옵티마이저다.
- DBMS에는 데이터 캐싱 메커니즘이 필수다.
- 논리적 I/O을 줄임으로써 물리적 I/O을 줄이는 것이 SQL 튜닝이다.
- 인덱스 스캔을 맹신하지 말자. 풀 스캔이 더 효율적일 수 있다.


## 1.1 SQL 파싱과 최적화
- 두 개의 테이블
    - EMP 테이블
        - 사원번호(EMPNO)
        - 사원명(ENAME)
        - 직업(JOB)
        - 부서번호(DEPTNO)
    - DEPT 테이블
        - 부서번호(DEPTNO)
        - 부서명(DNAME)
        - 지역(LOC)

- 두 테이블을 부서 번호로 조인하고 사원명 순으로 정렬하는 SQL
```sql
SELECT E.EMPNO, E.ENAME, E.JOB, D.DNAME, D.LOC
FROM EMP E, DEPT D
WHERE E.DEPTNO = D.DEPTNO
ORDER BY E.ENAME;
```

### 1.1.1 구조적, 집합적, 선언적 질의 언어
- SQL은 구조적 질의 언어(Structured Query Language)이다.
    - 또한 집합적이고 선언적인 질의언어이기도 하다.
    - 원하는 결과의 집합을 구조적, 집합적으로 선언하지만 그 과정은 절차적일 수 밖에 없다.


### 1.1.2 SQL 최적화
- SQL 최적화 과정의 세분화
- SQL 최적화 과정의 세분화
    - SQL 파싱
        - 파싱 트리 생성 : SQL 문장을 분석하여 파싱 트리 생성(select 등)
        - 문법 체크 : SQL 문장이 문법적으로 올바른지 체크(오타, 누락 등)
        - 시맨틱 체크 : SQL 문장이 의미적으로 올바른지 체크(컬럼, 테이블 존재 여부 등)
    - SQL 최적화
        - 최적화 트리 생성 : 파싱 트리를 분석하여 최적화 트리 생성(실행 계획)
        - 통계 정보 확인 : 최적화 트리 생성에 필요한 통계 정보를 확인(인덱스 통계, 테이블 통계 등)
        - 비용 계산 : 최적화 트리의 비용을 계산(비용이 가장 적은 실행 계획 선택)
    - 로우 소스 생성
        - 옵티마이저가 선택한 실행 계획에 따라 로우 소스 생성(실제 실행 계획)

- DBMS에도 SQL 실행 경로 미리보기가 있다.
    - Oracle : EXPLAIN PLAN
    - MySQL : EXPLAIN
    - SQL Server : SHOWPLAN
```sql
EXPLAIN PLAN
-------------------------
0 SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=14 Bytes=560)
1 0 SORT (ORDER BY) (Cost=4 Card=14 Bytes=560)
2 1 NESTED LOOPS (Cost=3 Card=14 Bytes=560)
3 2 TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=2 Card=4 Bytes=80)
4 2 TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=1 Card=3 Bytes=114)
5 4 INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (INDEX) (Cost=1 Card=3)
```

- 옵티마이저가 특정 실행계획을 선정하는 근거
```sql
테스트 테이블 생성
SQL> create table t
    2  as
    3  select d.no, e.*
    4  from scott.emp e
    5     , (select rownum no from dual connect by level <= 1000) d;
    
인덱스 생성
SQL> create index t_x01 on t(deptno, no);
SQL> create index t_x02 on t(deptno, job, no);

t 테이블 통계 정보 확인
SQL> exec dbms_stats.gather_table_stats(user, 't');

```

### 1.1.5. 옵티마이저 힌트
- 옵티마이저 힌트의 사용법
    - /* 이 안에 힌트를 통해 데이터 엑세스 경로를 변경할 수 있다. */

```sql
SELECT /*+ INDEX_ASC(E EMP_DEPTNO_IDX) */ E.EMPNO, E.ENAME, E.JOB, D.DNAME, D.LOC
```

- 옵티마이저 힌트 목록
    - 최적화 목표
        - ALL_ROWS : 전체 처리속도 최적화
        - FIRST_ROWS : 최초 처리속도 최적화
    - 엑세스 방식
        - FULL : 풀 스캔 유도
        - INDEX : 인덱스 스캔 유도
        - INDEX_DESC : 인덱스 역순 스캔 유도
        - INDEX_FFS : 인덱스 풀 스캔 유도
        - INDEX_SS : 인덱스 스킵 스캔
    - 조인 순서
        - LEADING : 괄호에 기술한 순서대로 조인
        - ORDERED : FROM 절에 기술한 순서대로 조인
        - SWAP_JOIN_INPUTS : 해시 조인 시, 인풋을 명시적으로 선택
    - 조인 방식
        - USE_NL : 루프 조인 유도
        - USE_MERGE : 병합 조인 유도
        - USE_HASH : 해시 조인 유도
        - NL_SJ : NL 세미조인 유도
        - MERGE_SJ : MERGE 세미조인 유도
        - HASH_SJ : HASH 세미조인 유도
    - 서브쿼리 팩토링
        - MATERIALIZE : WITH 문으로 정의한 집합을 물리적으로 생성하도록 유도
        - INLINE : WITH 문으로 정의한 집합을 인라인 뷰로 변환하도록 유도
    - 쿼리 변환
        - ... 추가 작성 예정

## 1.2 SQL 공유 및 재사용
- 소프트 파싱과 하드 파싱의 차이점
    - SGA의 구성요소로 라이브러리 캐시가 있다.
    - 소프트 파싱
        - 라이브러리 캐시에 SQL 문장이 존재하면 곧바로 실행 단계로 진행
    - 하드 파싱
        - 라이브러리 캐시에 SQL 문장이 존재하지 않으면 최적화 단계 진행
        - 로우 소스 생성
        - 실행

- SQL 최적화 과정이 하드 파싱인 이유
  - SQL 최적화 과정에서 옵티마이저는 다양한 정보를 참조하므로 하나의 쿼리를 실행하는 것에도 무수히 많은 후보군을 참조할 수 있다.
  - 데이터베이스의 I/O에 집중, 하드 파싱은 많은 CPU 자원을 소모한다.
  - 이로 인해 라이브러리 캐시가 필요하다.

- 공유 가능 SQL
    - 라이브러리 캐시에서 SQL을 찾을 때 키 값은 SQL 문장 그 자체를 사용한다.
```sql
그래서 다음의 모든 쿼리들이 전부 별도의 공간을 차지한다.
SELECT * FROM EMP WHERE EMPNO = 7369;
select * from emp where empno = 7369;
select * From emp where empno = 7369;
```

  - 바인드 변수
    - 최적화에 따라 과도한 I/O의 병목을 막아줄 수 있다.
```java
public void login(String login_id) throw Exception {
    String SQLStmt = "SELECT * FROM CUSTOMER WHERE LOGIN_ID = '" + login_id + "'";
    Statement stmt = conn.createStatement();
    ResultSet rs = stmt.executeQuery(SQLStmt);
    if(rs.next()) {
        // 로그인 성공
        }
    rs.close();
    stmt.close();
}
```
login_id가 1,000,000개라면 1,000,000개의 SQL이 라이브러리 캐시에 쌓이게 된다.
```sql
SELECT * FROM CUSTOMER WHERE LOGIN_ID = '1';
SELECT * FROM CUSTOMER WHERE LOGIN_ID = '2';
SELECT * FROM CUSTOMER WHERE LOGIN_ID = '3';
...
```
위의 코드는 다음과 같이 바꿀 수 있다.
```java
public void login(String login_id) throw Exception {
    String SQLStmt = "SELECT * FROM CUSTOMER WHERE LOGIN_ID = ?";
    PreparedStatement pstmt = conn.prepareStatement(SQLStmt);
    pstmt.setString(1, login_id);
    ResultSet rs = pstmt.executeQuery();
    if(rs.next()) {
        // 로그인 성공
        }
    rs.close();
    pstmt.close();
}
```
이렇게 하면 라이브러리 캐시에는 하나의 SQL만 쌓이게 된다.
```sql
SELECT * FROM CUSTOMER WHERE LOGIN_ID = ?
```


## 1.3 데이터 저장 구조 및 I/O 메커니즘

### 1.3.1 SQL이 느린 이유
- SQL이 느린 이유는 대부분 I/O의 병목현상 때문이다.
- 디스크 I/O가 SQL의 성능을 좌지우지 함.

### 1.3.2 데이터 저장 구조

### 1.3.3 블록 단위 I/O
- 데이터 I/O의 단위가 블록이므로, 특정 레코드 하나를 읽는 것에도 해당 블록 전체를 읽어야 한다.

### 1.3.4 시퀀셜 엑세스 vs 랜덤 엑세스
- 시퀀셜 엑세스
    - 블록을 순차적으로 읽는 것
    - 블록을 읽을 때마다 디스크 헤더를 이동시키지 않아도 되므로 빠르다.
- 랜덤 엑세스
    - 블록을 랜덤하게 읽는 것

### 1.3.5 논리적 I/O vs 물리적 I/O
- DB 버퍼캐시
    - 데이터 캐시. 디스크 I/O를 줄이기 위해 사용.
- 논리적 I/O
    - 버퍼캐시에 존재하는 데이터를 읽는 것
    - 
- 물리적 I/O
    - 디스크에서 데이터를 읽는 것
    - DB 버퍼 캐시에서 블록을 찾지 못해 디스크를 읽으면 물리적 I/O가 발생한다.

### 1.3.6 Single Block I/O vs Multi Block I/O
- Single Block I/O
    - 소량의 데이터를 읽을 때 주로 사용
- Multi Block I/O
    - 대량의 데이터를 읽을 때 주로 사용
    - 블록을 순차적으로 읽는 것이 빠르므로 시퀀셜 엑세스를 사용한다.

### 1.3.7 Table Full Scan vs Index Scan
- Table Full Scan
    - 테이블의 모든 블록을 읽는 것
    - 테이블의 모든 블록을 읽어야 하므로 물리적 I/O가 많이 발생한다.
- Index Scan
    - 인덱스의 리프 블록을 읽는 것
    - 랜덤 엑세스와 Single Block I/O를 사용한다.
    - 소량의 데이터를 검색할 때 유용하다.