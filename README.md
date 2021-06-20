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
  
  
# 1.2.1 Soft Parsing vs Hard Parsing
 - library cache : SQL 파싱, 최적화, 로우 소스 생성 과정을 거쳐 생성한 내부 프로시저를 반복 재사용할 수 있도록 캐싱해 두는 메모리 공간
 - Soft Parsing : SQL을 캐시에서 찾아 곧바로 실행단계로 넘어감
 - Hard Parsing : SQL을 캐시에서 찾는데 실패해 최적화 및 로우 소스 성성단계까지 모두 거침
 - 왜 최적화 과정은 Hard Parsing 인가 ?
  - 옵티마이저가 SQL을 최적화할 때 순식간에 많은 연산 수행 => 라이브러리 캐시로 저장하는 이유

# 1.2.2 바인드 변수의 중요성
 - 공유 가능 SQL (example)
 ```
 로그인 ID를 파라미터로 받는 프로시저 하나를 공유하면서 재사용
  create procedure LOGIN (login_id in varchar2) {...}
 ```
 => 이처럼 파라미터 driven 방식으로 SQL을 작성하는 방법 제공 (=바인드 변수) if 수많은 로그인 요청이 들어와도 라이브러리 캐시에는 
 ```
 select * from customer where login_id = :1
 ```
 하나만 발견

# 1.3.1 SQL이 느린 이유
 - 80~90% 디스크 I/O가 원인 => 프로세스는 디스크에서 데이터를 읽어야 할 땐 CPU를 OS에 반환하고 잠시 wait queue에서 waiting 상태로 I/O가 완료되길 기다림

# 1.3.2 데이터베이스 저장 구조 
 - 테이블 스페이스 : segment를 담는 container
  - segment(table) : 데이터 저장공간이 필요한 오브젝트
   - 익스텐트 : 공간을 확장하는 단위, 연속된 블록 집합
    - 블록 : 데이터를 읽고 쓰는 단위
  - segment(index)
  - segment(partition)
  - segment(LOB) 

# 1.3.3 시퀀셜 액세스 vs 랜덤 액세스
 - 시퀀셜 액세스 : 논리적 또는 물리적 연결 순서에 따라 차례대로 블록일 읽는 방식
 - 랜덤 액세스 : 레코드 하나를 읽기 위해 한 블록씩 접근하는 방식

# 1.3.4 논리적 I/O vs 물리적 I/O
 - DB 버퍼 캐시 : 디스크에서 읽은 데이터 블록을 캐싱해 둠으로써 같은 블록에 대한 반복적인 I/O call을 줄일 수 있음
 - 논리적 I/O : SQL문을 처리하는 과정에 메모리 버퍼캐시에서 발생한 총 블록 I/O
 - 물리적 I/O : 디스크에서 발생한 총 블럭 I/O => SQL 처리중 읽어야 할 블록을 버퍼캐시에서 찾지 못할 경우에만 디스크에 액세스 => 논리적 I/O의 일부
 - 버퍼캐시 히트율(BCHR) = (캐시에서 곧바로 찾은 블록 수 / 총 읽은 블록 수) * 100
                     = ( (논리적 I/O - 물리적 I/O) / 논리적 I/O ) * 100
                     = ( (1 - 물리적 I/O) / 논리적 I/O ) * 100
                     => 전체 블록 중에서 물리적인 디스크 I/O를 수반하지 않고 곧바로 메모리에서 찾은 비율
 - 평균적으로 99% 히트율 달성해야 함
 - 실제 SQL 성능을 향상하려면 논리적 I/O를 줄여야 함 => 물리적 I/O는 시스템 상황에 의해 결졍되는 통제 불가능한 외생변수
 => 논리적 I/O는 읽는 총 블록 개수를 줄여야 함 => 논리적 I/O를 줄여 물리적 I/O를 줄이는 것이 곧 SQL 튜닝
 
 # 1.3.5 Single Block I/O vs Multiblock I/O
  - Single Block I/O : 한번에 한 블록씩 요청하여 메모리에 적재하는 방식 => 소량의 데이터 읽을 때 효율적 (ex Index)
  - Multiblock I/O : 한번에 여러 블록을 요청하여 메모리에 적재하는 방식 => 테이블 전체를 스캔할 때 효율적
 
 # 1.3.6 Table Full Scan vs Index Range Scan
  - Table Full Scan : 테이블 전체를 스캔해서 읽는 방식
  - Index Range Scan : 인덱스를 이용해서 읽는 방식

 # 1.3.7 메모리 공유자원에 대한 엑세스 직렬화
  - 하나의 블록을 두 개 이상의 프로세스가 동시에 접근할 때 블록 정합성에 문제 발생 가능성 => 한 프로세스씩 순처적으로 접근하도록 구현하기 위해 직렬화 메커니즘 필요 (=Latch)
  - 버퍼 블록 자체에도 직렬화 메커니즘 존재 => 버퍼 Lock
  - 이런 직렬화 메커니즘에 의한 캐시 경합을 줄이려면, SQL 튜닝을 통해 논리적 I/O를 줄여야 한다.
   
