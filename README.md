# SQL_Tuning

# 1.1.1 SQL 최적화
  - (1) SQL 파싱
    - 1.1 파싱 트리 생성
    - 1.2 Syntax 체크
    - 1.3 Semantic 체크
  - (2) SQL 최적화 => 옵티마이저가 다양한 실행경로 중 가장 효율적인 하나 선택
  - (3) 로우 소스 생성

#
# 1.1.2 SQL 옵티마이저
  - (1) 실행계획 찾기
  - (2) 각 실행계획의 예상 비용 산정
  - (3) 최저 비용을 나타내는 실행계획 선택

#
# 1.1.3 실행계획과 비용 (example)

  ```
  SQL> create index t_1 on test(deptno, no)  // 인덱스 생성
  SQL> create index t_2 on test(deptno, job, no)  // 인덱스 생성
  ```
  
  ```
  SQL> exec dbms_stats.gather_table_stats(user, 'test');  // test 테이블 통계정보 수집 명령어
  ```

  ```
  SQL> set autotrace traceonly exp;  // autotrace 활성화
  ```
  
  ```
  결과 
  t_1 cost = 2
  t_2 cost = 19
  table full scan cost = 29
  ```
