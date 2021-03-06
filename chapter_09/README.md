# Chapter 9. 옵티마이저와 힌트

## 쿼리 실행 절차
1. 사용자로부터 요청된 SQL 문장을 쪼개서 MySQL 서버가 이해할 수 있는 수준으로 분리(파스 트리)한다.
2. SQL의 파싱 정보(파스 트리)를 확인하면서 어떤 테이블부터 읽고 어떤 인덱스를 이용해 테이블을 읽을지 선택한다.
3. 두 번째 단계에서 결정된 테이블의 읽기 순서나 선택된 인덱스를 이용해 스토리지 엔진으로부터 데이터를 가져온다.

* 첫 번째 단계를 "SQL 파싱"이라고 하며, MySQL 서버의 "SQL 파서"라는 모듈로 처리한다.
  * SQL 문장이 문법적으로 잘못 됐다면 이 단계에서 걸러진다.
  * 이 단계에서 "SQL 파스 트리"가 만들어진다.
  * MySQL 서버는 SQL 문장이 아닌, SQL 파스 트리를 이용하여 쿼리를 실행한다.
* 두 번째 단계에서는 첫 번째 단계에서 만들어진 SQL 파스 트리를 참조하면서 다음과 같은 내용을 처리한다.
  * 불필요한 조건 제거 및 복잡한 연산 단순화
  * 여러 테이블의 조인이 있는 경우 어떤 순서로 테이블을 읽을지 결정
  * 각 테이블에 사용된 조건과 인덱스 통계 정보를 이용해 사용할 인덱스를 결정
  * 가져온 레코드들을 임시 테이블에 넣고 다시 한번 가공해야 하는지 결정
* 두 번째 단계는 "최적화 및 실행 계획 수립" 단계이며, MySQL 서버의 옵티마이저에서 처리한다.
  * 이 단계가 완료되면 실행 계획이 만들어진다.
* 세 번째 단계는 수립된 실행 계획대로 스토리지 엔진에 레코드를 읽어오도록 요청하고, MySQL 엔진에서는 스토리지 엔진으로부터 받은 레코드를 조인하거나 정렬하는 작업을 수행한다.
* 첫 번째 단계와 두 번째 단계는 거의 MySQL 엔진에서 처리하며, 세 번째 단계는 MySQL 엔진과 스토리지 엔진이 동시에 참여해서 처리한다.

### 옵티마이저의 종류
* 비용 기반 최적화(Cost-based optimizer, CBO)
  * 현재 대부분의 RDMS가 선택하고 있는 옵티마이저 최적화 방법이다.
  * 쿼리를 처리하기 위한 여러 가지 가능한 방법을 만들고, 각 단위 작업의 비용 정보와 대상 테이블의 예측된 통계 정보를 이용해 실행 계획별 비용을 산출한다.
  * 산출된 실행 방법별로 비용이 최소로 소요되는 처리 방식을 선택해 최종적으로 쿼리를 실행한다.
* 규칙 기반 최적화(Rule-based optimizer, RBO)
  * 대상 테이블의 레코드 건수나 선택도 등을 고려하지 않고 옵티마이저에 내장된 우선순위에 따라 실행 계획을 수립하는 방식이다.
  * 통계 정보를 조사하지 않고 실행 계획이 수립되기 때문에 같은 쿼리에 대해서는 거의 항상 같은 실행 방법을 만들어 낸다.
  * 오래전부터 많은 RDBMS에서 거의 사용하지 않는다.

## 풀 테이블 스캔과 풀 인덱스 스캔

### 풀 테이블 스캔
* 인덱스를 사용하지 않고 테이블의 데이터를 처음부터 끝까지 읽어서 요청된 작업을 처리한다.
  * 테이블의 레코드 건수가 너무 적어서 인덱스를 통해 읽는 것보다 풀 테이블 스캔이 더 빠른 경우에 풀 테이블 스캔을 사용한다.
  * WHERE 절이나 ON 절에 인덱스를 이용할 수 있는 적절한 조건이 없는 경우에 풀 테이블 스캔을 사용한다.
  * 인덱스 레인지 스캔을 사용할 수 있는 쿼리라고 하더라도 옵티마이저가 판단한 조건 일치 레코드 건수가 너무 많은 경우(인덱스의 B-Tree를 샘플링해서 조사한 통계 정보 기준)에 풀 테이블 스캔을 사용한다.
* 일반적으로 테이블의 전체 크기는 인덱스보다 훨씬 크기 때문에 테이블을 처음부터 끝까지 읽는 작업은 대량의 디스크 읽기 작업을 필요로 한다.
* InnoDB 스토리지 엔진은 특정 테이블의 연속된 데이터 페이지가 읽히면 백그라운드 스레드에 의해 리드 어헤드(Read ahead) 작업이 자동으로 시작된다.
* 리드 어헤드란 어떤 영역의 데이터가 앞으로 필요해지리라는 것을 예측하여 요청이 오기 전에 미리 디스크에서 읽어 InnoDB의 버퍼 풀에 캐싱해 두는 것을 의미한다.
  * 즉, 풀 테이블 스캔이 실행되면 처음 몇 개의 데이터 페이지는 포그라운드 스레드가 페이지 읽기를 실행하지만 특정 시점부터는 읽기 작업을 백그라운드 스레드로 넘긴다.
* 리드 어헤드는 풀 인덱스 스캔에서도 동일하게 사용된다.

## 병렬 처리
* MySQL 8.0 버전부터는 용도가 한정되어 있긴 하지만 쿼리의 병렬 처리가 가능해졌다.
* 여기서 말하는 병렬 처리는 하나의 쿼리를 여러 스레드가 작업을 나누어 동시에 처리하는 것을 의미한다.
* 책의 쓰여질 당시에는 쿼리를 여러 개의 스레드에서 병렬 처리하도록 하는 힌트나 옵션은 없다고 한다.
* 병렬 처리용 스레드 개수를 아무리 늘리더라도 서버에 장착된 CPU의 코어 개수를 넘어서는 경우에는 오히려 성능이 떨어질 수 있으니 주의해야 한다.

## ORDER BY 처리 (Using filesort)
* 정렬을 처리하는 방법은 인덱스를 이용하는 방법과 쿼리가 실행될 때 "Filesort"라는 별도의 처리를 이용하는 방법으로 나눌 수 있다.
* 인덱스를 사용하지 않고 별도의 정렬 처리를 수행했는지는 실행 계획의 Extra 컬럼에 "Using filesort" 메시지가 표시되는지 여부로 판단할 수 있다.

### 인덱스를 이용하는 방법
* 장점
  * INSERT, UPDATE, DELETE 쿼리가 실행될 때 이미 인덱스가 정렬되어 있어서 순서대로 읽기만 하면 되므로 빠르다.
* 단점
  * INSERT, UPDATE, DELTE 작업 시 부가적인 인덱스 추가/삭제 작업이 필요하므로 느리다.
  * 인덱스로 인해 디스크 공간이 더 필요하다.
  * 인덱스의 개수가 늘어날수록 InnoDB의 버퍼 풀을 위한 메모리가 많이 필요하다.

### Filesort를 이용하는 방법
* 장점
  * 인덱스를 생성하지 않아도 되므로 인덱스를 이용할 때의 단점이 장점으로 바뀐다.
  * 정렬해야 할 레코드가 많지 않으면 메모리에서 Filesort가 처리되므로 충분히 빠르다.
* 단점
  * 정렬 작업이 쿼리 실행 시 처리되므로 레코드 대상 건수가 많아질수록 쿼리의 응답 속도가 느리다.

### 모든 정렬을 인덱스를 이용하도록 튜닝하기 어려운 이유 
* 정렬 기준이 너무 많아서 요건별로 모두 인덱스를 생성하는 것이 불가능한 경우
* GROUP BY의 결과 또는 DISTINCT 같은 처리의 결과를 정렬해야 하는 경우
* UNION의 결과와 같이 임시 테이블의 결과를 다시 정렬해야 하는 경우
* 랜덤하게 결과 레코드를 가져와야 하는 경우

### 소트 버퍼
* 정렬을 수행하기 위해서 별도의 메모리 공간을 할당받아서 사용하는데, 이 메모리 공간을 소트 버퍼(Sort buffer)라고 한다.
* 소프 버퍼는 정렬이 필요한 경우에만 할당되며, 버퍼의 크기는 정렬해야 할 레코드의 크기에 따라 가변적으로 증가하지만 최대 사용 가능한 소트 버퍼의 공간은 시스템 변수로 설정 가능하다.
* 소트 버퍼를 위한 메모리 공간은 쿼리의 실행이 완료되면 즉시 시스템으로 반납된다.
* MySQL은 정렬해야 할 레코드를 여러 조각으로 나눠서 처리하는데, 이 과정에서 임시 저장을 위해 디스크를 사용한다.
* MySQL은 글로벌 메모리 영역과 세션(로컬) 메모리 영역으로 나뉘어 있는데, 소트 버퍼는 세션 메모리 영역에 해당된다.

## 정렬 알고리즘
* 레코드를 정렬할 때 레코드 전체를 소트 버퍼에 담을지 또는 정렬 기준 컬럼만 소트 버퍼에 담을지에 따라 "싱글 패스"와 "투 패스" 2가지 정렬 모드로 나눌 수 있다.

### 싱글 패스 정렬 방식
* 소트 버퍼에 정렬 기준 컬럼을 포함해 SELECT 대상이 되는 컬럼 전부를 담아서 정렬을 수행하는 정렬 방식이다.
```sql
SELECT emp_no, first_name, last_name
FROM employees
ORDER BY first_name;
```
* 싱글 패스 정렬 방식에서 위 쿼리는 emp_no, first_name, last_name을 소트 버퍼에 모두 담고 정렬을 수행한다.

### 투 패스 정렬 방식
* 정렬 대상 컬럼과 프라이머리 키 값만 소트 버퍼에 담아서 정렬을 수행하고, 정렬된 순서대로 다시 프라이머리 키로 테이블을 읽어서 SELECT할 컬럼을 가져오는 정렬 방식이다.
* 싱글 패스 정렬 방식이 도입되기 이전부터 사용하던 방식이다.
* MySQL 8.0 버전에서도 여전히 특정 조건에서는 투 패스 정렬 방식을 사용한다.

### 싱글 패스 정렬 방식과 투 패스 정렬 방식
* 싱글 패스 정렬 방식은 더 많은 소트 버퍼 공간이 필요하다.
* 투 패스 방식은 테이블을 두 번 읽어야 하는 불합리가 있다.
* 최신 버전의 MySQL에서는 싱글 패스 정렬 방식을 주로 사용하지만 다음의 경우에는 싱글 패스 정렬 방식을 사용하지 못하고 투 패스 정렬 방식을 사용한다.
  * 레코드의 크기가 max_length_for_sort_data 시스템 변수에 설정된 값보다 클 때
  * BLOB이나 TEXT 타입의 컬럼이 SELECT 대상에 포함될 때
* 싱글 패스 방식은 정렬 대상 레코드의 크기나 건수가 작은 경우에 따른 성능을 보인다.
* 투 패스 방식은 정렬 대상 레코드의 크기나 건수가 상당히 많은 경우에 효율적이다.
* SELECT 쿼리에서 필요한 컬럼만 조회하지 않고 모든 컬럼(*)을 조회하도록 개발하는 방식은 성능이 중요한 기능이라면 지양해야 한다.
  * 정렬 버퍼를 비효율적으로 사용하기 떄문이다.
  * 임시 테이블이 필요한 쿼리에서도 영향을 미친다.

### 정렬 처리 방법
* 쿼리에 ORDER BY가 사용되면 반드시 다음 3가지 처리 방법 중 하나로 정렬이 되며, 아래쪽에 있는 정렬 방법으로 갈수록 느리다.

| 정렬 처리 방법                    | 실행 계획의 Extra 컬럼 내용                         |
|-----------------------------|--------------------------------------------|
| 인덱스를 이용한 정렬                 | 별도 표기 없음                                   |
| 조인에서 드라이빙 테이블만 정렬           | "Using filesort" 메시지가 표시됨                  |
| 조인에서 조인 결과를 임시 테이블로 저장 후 정렬 | "Using temporary; Using filesort" 메시지가 표시됨 |

* 일반적으로 조인이 수행되면서 레코드 건수와 레코드의 크기는 거의 배수로 불어나기 떄문에 가능하다면 드라이빙 테이블만 정렬한 다음 조인을 수행하는 방법이 효율적이다.
* 두 번째 방법보다는 첫 번째 방법이 더 효율적이다.

### 인덱스를 이용한 정렬
* 인덱스를 이용한 정렬을 위해서는 반드시 ORDER BY에 명시된 컬럼이 제일 먼저 읽는 테이블(조인이 사용된 경우에는 드라이빙 테이블)에 속하고, ORDER BY의 순서대로 생성된 인덱스가 있어야 한다. 또한 WHERE 절에 첫 번째로 읽는 테이블의 컬럼에 대한 조건이 있다면 그 조건과 ORDER BY는 같은 인덱스를 사용할 수 있어야 한다.
* 여러 테이블이 조인되는 경우에는 네스티드-루프(Nested-loop) 방식의 조인에서만 이 방식을 사용할 수 있다.
* 네스티드 루프 조인? (Nested Loop Join)
  * 드라이빙 테이블의 처리 범위를 하나씩 액세스 하면서 그 추출된 값으로 드리븐 테이블을 조인하는 방식이다. (중첩된 반복문과 유사한 방식)
  * 결과 행의 수가 적은 테이블을 조인 순서상 선행 테이블로 선택하는 것이 성능에 좋다.
  * 랜덤 액세스 방식으로 접근하기 때문에 처리 범위가 좁은 조건을 선택하는 것이 유리하다.
* 인덱스를 이용한 정렬을 수행하는 경우에는 실제 인덱스의 값이 이미 정렬돼 있기 때문에 인덱스의 순서대로 읽기만 하면 된다.
  * 실제로 MySQL 엔진에서 별도의 정렬을 위한 추가 작업을 수행하지 않는다.
  * ORDER BY가 있든 없든 같은 인덱스를 레인지 스캔해서 나온 결과는 같은 순서로 정렬되어 있다.
  * ORDER BY 절을 넣지 않아도 자동으로 정렬이 된다고 해서 ORDER BY 절을 제거하는 행동은 좋지 않다.
  * 어떤 이유로 쿼리의 실행 계획이 조금 변경되다면 ORDER BY가 명시되지 않은 쿼리는 기대했던 순서로 가져오지 못해서 어플리케이션 상에서 버그로 연결될 수 있기 떄문이다.
  * 인덱스로 정렬이 될 떄는 ORDER BY가 쿼리에 명시된다고 해서 작업량이 더 늘지 않으므로 명시적으로 처리하자.
* 네스티드-루프 방식으로 실행되기 때문에 조인으로 인한 드라이빙 테이블의 인덱스 읽기 순서는 바뀌지 않는다.
* 조인이 사용된 쿼리의 실행 계획에 조인 버퍼(Join buffer)가 사용되면 순서가 흐트러질 수 있기 때문에 주의해야 한다.

### 조인의 드라이빙 테이블만 정렬
* 이 방법으로 정렬이 처리되려면 조인에서 드라이빙 테이블의 컬럼만으로 ORDER BY 절을 작성해야 한다.

### 임시 테이블을 이용한 정렬
* 조인의 드라이빙 테이블을 이용한 정렬 외에 다른 패턴의 쿼리는 항상 조인의 결과를 임시 테이블에 저장해야 하고, 그 결과를 다시 정렬하는 과정을 거친다.
* 이 방법은 정렬해야 할 레코드 건수가 가장 많기 때문에 가장 느린 정렬 방법이다.
* ORDER BY 절의 정렬 기준 컬럼이 드리븐 테이블에 있는 컬럼인 경우에는 정렬이 수행되기 전에 드리븐 테이블을 읽어야 하므로 조인의 결과를 임시 테이블에 저장해야 한다.
* 실행 계획에서 살펴봤을 때 Extra 컬럼에 "Using temporary; Using filesort" 코멘트가 표시된다.
  * 이는 조인의 결과를 임시 테이블에 저장하고, 그 결과를 다시 정렬 처리했음을 의미한다.

### 정렬 처리 방법의 성능 비교
* 일반적으로 LIMIT은 테이블이나 처리 결과의 일부만 가져오기 때문에 MySQL 서버가 처리해야 할 작업량을 줄이는 역할을 한다.
* ORDER BY나 GROUP BY 같은 작업은 WHERE 조건을 만족하는 레코드를 LIMIT 건수만큼만 가져와서는 처리할 수 없다. 우선 조건을 만족하는 레코드를 모두 가져와서 정렬을 수행하거나 그루핑 작업을 수행해야만 LIMIT으로 건수를 제한할 수 있다.
* WHERE 조건이 아무리 인덱스를 잘 활용하도록 튜닝해도 잘못된 ORDER BY나 GROUP BY 때문에 쿼리가 느려지는 경우가 자주 발생한다.

## 쿼리가 처리되는 방법

### 스트리밍 방식
* 서버 쪽에서 처리할 데이터가 얼마인지에 관계없이 조건에 일치하는 레코드가 검색될 때마다 바로바로 클라이언트로 전송해주는 방식을 의미한다.
* 클라이언트는 쿼리를 요청하고 곧바로 첫 번째 레코드를 전달받는다.
* 가장 마지막 레코드는 언제 받을지 알 수 없다.
* 스트리밍 방식으로 처리되는 쿼리는 얼마나 많은 레코드를 조회하는지에 상관없이 빠른 응답 시간을 보장해주므로 OLTP 환경에 적합하다.
* LIMIT처럼 결과 건수를 제한하는 조건들은 쿼리의 전체 실행 시간을 상당히 줄여줄 수 있다.

### 버퍼링 방식
* ORDER BY나 GROUP BY 같은 처리는 쿼리의 결과가 스트리밍되는 것을 불가능하게 한다. (WHERE 조건에 일치하는 레코드를 가져온 후, 정렬하거나 그루핑해서 차례대로 보내야 하기 때문이다.)
* MySQL 서버에서 모든 레코드를 검색하고 정렬하는 동안 클라이언트는 응답을 기다려야 하므로 응답 속도가 느려진다.
* 버퍼링 방식은 LIMIT처럼 결과 건수를 제한하는 조건이 있어도 성능 향상에 별로 도움이 되지 않는다.

```text
스트리밍 처리는 어떤 클라이언트 도구나 API를 사용하느냐에 따라 그 방식이 달라진다.
JDBC 라이브러리를 이용해 SELECT * FROM bigtable 같은 쿼리를 수행하면 MySQL 서버는 레코드를 읽자마자 클라이언트로 전달하지만 JDBC는 응답받은 레코드를 일단 내부 버퍼에 모두 담아둔다.
그리고 마지막 레코드까지 모두 전달받으면 클라리언트 애플리케이션에 반환하게 된다.

MySQL 서버는 스트리밍 방식으로 처리하지만 클라이언트의 JDBC 라이브러리가 버퍼링하는 것이다.

JDBC 라이브러리가 자체적으로 레코드를 버퍼링하는 이유는 전체 처리(Throughput) 시간이 짧고 MySQL 서버와의 통신 횟수가 적어 자원 소모가 줄어들기 때문이다.

이러한 JDBC의 버퍼링 처리 방식은 기본 동작 방식이며, 대량의 데이터를 가져와야 할 때는 MySQL 서버와 JDBC 간의 전송 방식을 스트리밍 방식으로 변경할 수 있다.
```

* 인덱스를 사용한 정렬 방식만 스트리밍 형태의 처리이며, 나머지는 모두 버퍼링된 후에 정렬된다.
* 인덱스를 사용한 정렬 방식은 LIMIT으로 제한된 건수만큼만 읽으면서 바로바로 클라이언트로 전송할 수 있다.
* 인덱스를 사용하지 못하는 경우에는 필요한 모든 레코드를 디스크로부터 읽어서 정렬한 후에 LIMIT으로 제한된 건수만큼 잘라서 클라이언트로 전송할 수 있다.

## GROUP BY 처리
* GROUP BY 또한 ORDER BY와 같이 쿼리가 스트리밍된 처리를 할 수 없게 하는 처리 중 하나다.
* HAVING 절은 GROUP BY 결과에 대해 필터링 역할을 수행한다.
* GROUP BY에 사용된 조건은 인덱스를 사용해서 처리될 수 없으므로 HAVING 절을 튜닝하기 위해 인덱스를 생성하거나 다른 방법을 고민할 필요는 없다.
* GROUP BY 작업도 인덱스를 사용하거나 사용할 수 없는 경우가 있다.
  * 인덱스를 차례대로 읽는 인덱스 스캔
  * 인덱스를 건너뛰면서 읽는 루스 인덱스 스캔
  * 인덱스를 사용하지 못하면 임시 테이블을 사용

### 인덱스 스캔을 이용하는 GROUP BY (타이트 인덱스 스캔)
* 조인의 드라이빙 테이블에 속한 컬럼만 이용해 그루핑할 때 GROUP BY 컬럼으로 이미 인덱스가 있다면 그 인덱스를 차례대로 읽으면서 그루핑 작업을 수행하고 그 결과로 조인을 처리한다.
* GROUP BY가 인덱스를 사용해서 처리하더라도 그룹 함수 등의 그룹 값을 처리해야하면 임시 테이블이 필요할 수 있다.
* GROUP BY가 인덱스를 통해 처리되면 인덱스는 이미 정렬되어 있으므로 추가적인 정렬 작업이나 내부 임시 테이블은 필요하지 않다.
* Extra 컬럼에 별도로 GROUP BY 관련 코멘트("Using index for group-by")나 임시 테이블 사용 또는 정렬 관련 코멘트("Using temporary, Using filesort")가 표시되지 않는다.

### 루스 인덱스 스캔을 이용하는 GROUP BY
* 루스(Loose) 인덱스 스캔 방식은 인덱스의 레코드를 건너뛰면서 필요한 부분만 읽어서 가져오는 것을 의미한다.
* 옵티마이저가 루스 인덱스 스캔을 사용할 때는 실행 계획의 Extra 컬럼에 "Using index for group-by"가 표시된다.
* MySQL의 루스 인덱스 스캔 방식은 단일 테이블에 대해 수행하는 GROUP BY 처리에만 사용할 수 있다.
* 프리픽스 인덱스(Prefix index, 컬럼값의 앞쪽 일부만으로 생성된 인덱스)는 루스 인덱스 스캔을 사용할 수 없다.
* 인덱스 레인지 스캔에서는 유니크한 값의 수가 많을수록 성능이 향상되지만, 루스 인덱스 스캔에서는 유니크한 값의 수가 적을수록 성능이 향상된다.
* 루스 인덱스 스캔은 별도의 임시 테이블이 필요하지 않다.
* MIN()과 MAX() 이외의 집합 함수가 사용되면 사용이 불가능하다.
* GROUP BY에 사용된 컬럼이 인덱스 구성 컬럼의 왼쪽부터 일치하지 않으면 사용이 불가능하다.
* SELECT 절의 컬럼이 GROUP BY와 일치하지 않으면 사용이 불가능하다.
* MySQL8.0 버전부터는 루스 인덱스 스캔과 동일한 방식으로 동작하는 인덱스 스킵 스캔 최적화가 도입됐다.
  * 이전 버전에서는 GROUP BY 절의 처리를 위해서만 루스 인덱스 스캔이 사용됐지만, 8.0 버전부터는 인덱스 스킵 스캔이 도입되면서 옵티마이저가 필요로 하는 레코드를 검색하는 부분까지 루스 인덱스 스캔방식으로 최적화가 가능해졌다.
  * 인덱스 스킵 스캔은 루스 인덱스 스캔과 마찬가지로 인덱스의 선행 컬럼의 유니크한 값이 많을 수록 쿼리의 성능이 떨어진다. 그래서 인덱스 스킵 스캔에서도 선행 컬럼의 유니크한 값의 개수가 많으면 인덱스 스킵 스캔 최적화를 하지 않는다.

### 임시 테이블을 사용하는 GROUP BY
* GROUP BY의 기준 컬럼이 드라이빙 테이블에 있든 드리븐 테이블에 있든 관계없이 인덱스를 전혀 사용하지 못할 때 임시 테이블을 이용한다.
* MySQL 8.0 이전 버전까지는 GROUP BY가 사용된 쿼리는 그루핑되는 컬럼을 기준으로 묵시적인 정렬까지 수행했다.
* MySQL 8.0 버전부터는 묵시적인 정렬을 수행하지 않는다.
* MySQL 8.0 에서는 GROUP BY가 필요한 경우 내부적으로 GROUP BY 절의 컬럼들로 구성된 유니크 인덱스를 가진 임시 테이블을 만들어서 중복 제거와 집합 함수 연산을 수행한다.
  * 조인 결과를 한 건씩 가져와서 임시 테이블에서 중복 체크를 하면서 INSERT 또는 UPDATE를 실행한다.
* MySQL 8.0 버전에서도 GROUP BY와 ORDER BY가 같이 사용되면 명시적으로 정렬 작업을 실행한다. 이때는 Extra 컬럼에 "Using temporary"와 함께 "Using filesort"가 표시된다.

### DISTINCT 처리
* 집합 함수와 같이 DISTINCT가 사용되는 쿼리의 실행 계획에서 DISTINCT 처리가 인덱스를 사용하지 못할 떄는 항상 임시 테이블이 필요하다.
  * 하지만 Extra 컬럼에는 "Using temporary" 메시지가 출력되지 않는다.

#### SELECT DISTINCT ...
* 단순한 SELECT 쿼리에서 유니크한 레코드만 가져오고자 하면 SELECT DISTINCT 형태의 쿼리를 사용하는데, 이 경우에는 GROUP BY와 동일한 방식으로 처리된다.
* MySQL 8.0 버전부터는 GROUP BY를 수행하는 쿼리에 ORDER BY 절이 없으면 정렬을 사용하지 않기 때문에 다음의 두 쿼리는 내부적으로 같은 작업을 수행한다.
```sql
SELECT DISTINCT emp_no FROM salaries;
SELECT emp_no FROM salaries GROUP BY emp_no;
```
* DISTINCT는 SELECT하는 레코드를 유니크하게 SELECT하는 것이지, 특정 컬럼만 유니크하게 조회하는 것이 아니다.
```sql
SELECT DISTINCT first_name, last_name FROM employees;
-- 위 쿼리는 (first_name, last_name) 조합 전체가 유니크한 레코드를 가져온다.

SELECT DISTINCT(first_name), last_name FROM employees;
-- 위 쿼리는 MySQL 서버가 뒤의 괄호를 의미 없이 사용된 괄호로 해석하고 제거해 버린다. DISTINCT는 함수가 아니므로 그 뒤의 괄호는 의미가 없다.
```

#### 집합 함수와 함께 사용된 DISTINCT
* COUNT(), MIN(), MAX() 같은 집합 함수 내에서 DISTINCT 키워드가 사용될 수 있는데, 이 경우에는 집합 함수 내에서 인자로 전달된 컬럼 값이 유니크한 것들을 가져온다.
```sql
SELECT COUNT(DISTINCT s.salary)
FROM employees e, salaries s
WHERE e.emp_no = s.emp_no
AND e.emp_no BETWEEN 100001 AND 100100;
```
* 위 쿼리는 내부적으로 "COUNT(DISTINCT s.salary)"를 처리하기 위해 임시 테이블을 사용한다.
* 하지만 이 쿼리의 실행 계획에서는 임시 테이블을 사용한다는 메시지는 표시되지 않는다.
* 위 쿼리는 employees 테이블과 salaries 테이블을 조인한 결과에서 salary 컬럼의 값만 저장하기 위한 임시 테이블을 만들어서 사용한다.
  * 임시 테이블의 salary 컬럼에는 유니크 인덱스가 생성되기 때문에 레코드 건수가 많아진다면 상당히 느려질 수 있는 형태의 쿼리다.
* 만약 COUNT(DISTINCT e.last_name)를 하나 더 추가한다면 e.last_name 컬럼의 값을 저장하는 또 다른 임시 테이블이 필요하므로 전체적으로 2개의 임시 테이블을 사용한다.
* 인덱스된 컬럼에 대해 DISTINCT 처리를 수행할 수 있을 떄는 인덱스를 풀 스캔하거나 레인지 스캔하면서 임시 테이블 없이 최적화된 처리를 수행할 수 있다.

## 내부 임시 테이블 활용
* MySQL 엔진이 스토리지 엔진으로부터 받아온 레코드를 정렬하거나 그루핑할 때는 내부적인 임시 테이블을 사용한다.
* 여기서 말하는 임시 테이블은 "CREATE TEMPORARY TABLE" 명령으로 만든 임시 테이블과 다르다.
* 일반적으로 MySQL 엔진이 사용하는 임시 테이블은 처음에는 메모리에 생성됐다가 테이블의 크기가 커지면 디스크로 옮겨진다. (특정 예외 케이스에는 메모리를 거치지 않고 바로 디스크에 임시 테이블이 만들어지기도 한다.)
* MySQL 엔진이 내부적인 가공을 위해 생성하는 임시 테이블은 다른 세션이나 다른 쿼리에서 이용할 수 없으며, 쿼리의 처리가 완료되면 자동으로 삭제된다.

### 메모리 임시 테이블과 디스크 임시 테이블
* MySQL 8.0 버전부터 MEMORY 스토리지 엔진 대신 가변 길이 타입을 지원하는 TempTable 스토리지 엔진이 도입됐다.
  * 기존 MEMORY 스토리지 엔진은 가변 길이 타입을 지원하지 못했기 때문에 임시 테이블이 안들어지면 최대 길이만큼 메모리를 할당해서 사용했다. (메모리 낭비)
* MySQL 8.0 버전부터 MyISAM 스토리지 엔진 대신 트랜잭션 지원이 가능한 InnoDB 스토리지 엔진(또는 TempTable 스토리지 엔진의 MMAP 파일 버전)이 사용되도록 개선됐다.
* TempTable은 기본 값으로 1GB로 설정돼 있고, 임시 테이블의 크기가 1GB보다 커지는 경우 MySQL 서버는 메모리의 임시 테이블을 디스크로 기록한다.
  * 이때 MySQL 서버는 아래의 2가지 디스크 저장 방식 중 하나를 선택한다.
  * MMAP 파일로 디스크에 기록
  * InnoDB 테이블로 기록
* MySQL 서버는 기본 값으로 MMAP 파일로 기록하도록 설정되어 있다. TempTable 크기가 1GB를 넘으면 MMAP 파일로 전환한다.
* 메모리의 TempTable을 MMAP 파일로 전환하는 것은 InnoDB 테이블로 전환하는 것보다 오버헤드가 적다.

### 임시 테이블이 필요한 쿼리
* 다음과 같은 패턴의 쿼리는 MySQL 엔진에서 별도의 데이터 가공 작업을 필요로 하므로 내부 임시테이블을 생성한다. 인덱스를 사용하지 못할 때에도 내부 임시 테이블을 생성해야 할 때가 많다.
  * ORDER BY와 GROUP BY에 명시된 컬럼이 다른 쿼리
  * ORDER BY나 GROUP BY에 명시된 컬럼이 조인의 순서상 첫 번째 테이블이 아닌 쿼리
  * DISTINCT와 ORDER BY가 동시에 존재하는 경우 또는 DISTINCT가 인덱스로 처리되지 못하는 쿼리
  * UNION이나 UNION DISTINCT가 사용된 쿼리 (select type 컬럼이 UNION RESULT인 경우)
  * 쿼리의 실행 계획해서 select_type이 DERIVED인 쿼리
* 임시 테이블을 사용하는지는 Extra 컬럼에 "Using temporary"라는 메시지가 표시되는지 확인하면 된다.
* "Using temporary"가 한 번 표시됐다고 해서 임시 테이블이 하나만 사용했다는 것을 의미하지는 않는다.
* "Using temporary"라는 메시지가 표시되지 않을 때도 임시 테이블을 사용할 수 있는데, 위의 조건에서 마지막 3개 패턴이 그러한 예다.
* 첫 번째부터 네 번째까지의 쿼리 패턴은 유니크 인덱스를 가지는 내부 임시 테이블이 생성된다.
* 마지막 쿼리 패턴은 유니크 인덱스가 없는 내부 임시 테이블이 생성된다.
* 일반적으로 유니크 인덱스가 있는 내부 임시 테이블은 그러지 않은 쿼리보다 처리 성능이 상당히 느리다.

#### 임시 테이블이 디스크에 생성되는 경우
* UNION이나 UNION ALL에서 SELECT되는 컬럼 중에서 길이가 512바이트 이상인 크기의 컬럼이 있는 경우
* GROUP BY나 DISTINCT 컬럼에서 512바이트 이상인 크기의 컬럼이 있는 경우
* 메모리 임시 테이블의 크기가 (MEMORY 스토리지 엔진에서) tmp_table_size 또는 max_heap_table_size 시스템 변수보다 크거나 (TempTable 스토리지 엔진에서) temptable_max_ram 시스템 변수 값보다 큰 경우

## 고급 최적화

### MRR과 배치 키 액세스 (mrr & batched_key_access)
* MRR은 "Multi-Range Read"를 줄인 말이다. 메뉴얼에서는 DS-MRR(Disk Sweep Multi-Range Read)라고도 한다.
* MySQL 서버의 내부 구조상 조인 처리는 MySQL 엔진이 처리하지만, 실제 레코드를 검색하고 읽는 부분은 스토리지 엔진이 담당한다.
  * 드라이빙 테이블의 레코드 건별로 드리븐 테이블의 레코드를 찾으면 레코드를 찾고 읽는 스토리지 엔진에서는 아무런 최적화를 수행할 수 없다.
* 이러한 단점을 보완하기 위해서 MySQL 서버는 드라이빙 테이블의 레코드를 읽어서 드리븐 테이블과의 조인을 즉시 실행하지 않고 조인 대상을 버퍼링한다.
* 조인 버퍼에 레코드가 가득 차면 MySQL 엔진은 버퍼링된 레코드를 스토리지 엔진으로 한 번에 요청한다.
  * 스토리지 엔진은 읽어야 할 레코드들을 데이터 페이지에 정렬된 순서로 접근해서 디스크의 데이터 페이지 읽기를 최소화할 수 있게된다.
  * 데이터 페이지가 메모리(InnoDB 버퍼 풀)에 있다고 하더라도 버퍼 풀의 접근을 최소화할 수 있다.
  * 이러한 읽기 방식을 MRR이라고 한다.

### 블록 네스티드 루프 조인 (block_nested_loop)
* 조인의 연결 조건이 되는 컬럼에 모두 인덱스가 있는 경우에 사용되는 조인 방식이다.
* 네스티드 루프 조인과 블록 네스티드 루프 조인의 차이는 조인 버퍼가 사용되는지 여부와 조인에서 드라이빙 테이블과 드리븐 테이블이 어떤 순서로 조인되느냐다.
* Extra 컬럼에 "Using Join Buffer"가 표시되면 조인 버퍼를 사용한다는 것을 의미한다.
* 조인은 드라이빙 테이블에서 일치하는 레코드의 건수만큼 드리븐 테이블을 검색하면서 처리된다.
  * 드라이빙 테이블은 한 번에 쭉 읽지만, 드리븐 테이블은 여러 번 읽는다는 것을 의미한다.
* 어떤 방식으로도 드리븐 테이블의 풀 테이블 스캔이나 인덱스 풀 스캔을 피할 수 없다면 옵티마이저는 드라이빙 테이블에서 읽은 레코드를 메모리에 캐시한 후 드리븐 테이블과 이 메모리 캐시를 조인하는 형태로 처리한다.
  * 이때 사용되는 메모리의 캐시를 조인 버퍼라고 한다.
* 일반적으로 조인이 수행된 후 가져오는 결과는 드라이빙 테이블의 순서에 의해 결정되지만, 조인 버퍼가 사용되는 조인에서는 결과의 정렬 순서가 흐트러질 수 있다.
  * MySQL 8.0.18 버전부터는 해시 조인 알고리즘이 도입되었고, 8.0.20 버전부터는 블록 네스티드 루프 조인은 더이상 사용되지 않고 해시 조인 알고리즘이 대체되어 사용된다.
  * 따라서 Extra 컬럼에 더이상 "Using Join Buffer (block nested loop)" 메시지가 표시되지 않을 수도 있다.

### 인덱스 컨디션 푸시다운 (index_condition_pushdown)
* 기존에는 인덱스의 컬럼 조건에 LIKE '%sal'과 같이 체크 조건으로만 사용되는 조건이 있다면 해당 조건을 처리하기 위해 테이블 레코드를 읽어서 처리했다.
  * 이는 인덱스를 범위 제한 조건으로 사용하지 못하는 조건은 MySQL 엔진이 스토리지 엔진으로 아예 전달해주지 않았기 때문이다. 그래서 스토리지 엔진에서는 불필요한 테이블 읽기를 수행할 수밖에 없었던 것이다.
* 5.6 버전부터는 인덱스를 범위 제한 조건으로 사용하지 못한다고 하더라도 인덱스에 포함된 컬럼의 조건이 있다면 모두 같이 모아서 스토리지 엔진으로 전달할 수 있게 핸들러 API가 개선되었는데, 이를 인덱스 컨디션 푸시다운이라고 한다.
  * 인덱스를 이용해 최대한 필터링까지 완료해서 꼭 필요한 레코드에 대해서만 테이블 읽기를 수행할 수 있게 되었다.
* Extra 컬럼에는 "Using index codition"이 표시된다.

### 인덱스 확장 (use_index_extensions)
* use_index_extensions 옵티마이저 옵션은 InnoDB 스토리지 엔진을 사용하는 테이블에서 세컨더리 인덱스에 자동으로 추가된 프라이머리 키를 활용할 수 있게 할지를 결정하는 옵션이다.
* InnoDB의 프라이머리 키가 세컨더리 인덱스에 포함돼 있으므로 정렬 작업도 인덱스를 활용해서 처리될 수 있다는 장점이 있다.

### 인덱스 머지 (index_merge)
* 대부분의 옵티마이저는 테이블별로 하나의 인덱스만 사용하도록 실행 계획을 수립하지만, 인덱스 머지 실행 계획을 사용하면 하나의 테이블에 대해 2개 이상의 인덱스를 이용해 쿼리를 처리한다.
* 한 테이블에 대한 WHERE 조건이 여러 개 있더라도 하나의 인덱스에 포함된 컬럼에 대한 조건만으로 인덱스를 검색하여 작업 범위를 충분히 줄일 수 있는 경우라면 테이블별로 하나의 인덱스만 활용하는 것이 효율적이다.
* 쿼리에 사용된 각각의 조건이 서로 다른 인덱스를 사용할 수 있고 그 조건을 만족하는 레코드 건수가 많을 것으로 예상될 때 MySQL 서버는 인덱스 머지 실행 계획을 선택한다.
* 인덱스 머지 실행 계획은 다음과 같이 3개의 세부 실행 계획으로 나뉜다.
  * index_merge_intersection
  * index_merge_sort_union
  * index_merge_union

### 인덱스 머지 - 교집합 (index_merge_intersection)
* Extra 컬럼에 "Using intersect"가 표시된다.
* 쿼리가 여러 개의 인덱스를 각각 검색해서 그 결과의 교집합만을 반환한다.

### 인덱스 머지 - 합집합 (index_merge_union)
* Extra 컬럼에 "Using union"이 표시된다.
* WHERE 조건에 사용된 2개 이상의 조건이 각각의 인덱스를 사용하되 OR 연산자로 연결된 경우에 사용되는 최적화다.
* OR 조건의 경우 각 인덱스에서 조건을 만족하는 컬럼이 존재하여 중복 레코드가 존재할 것 같지만, 각 인덱스에는 프라이머리 키를 포함하고 있고 이를 기반으로 이미 정렬이 되어 있으므로 중복된 레코드를 정렬 없이 걸러낼 수 있다.
  * 정렬된 두 집합의 결과를 하나씩 가져와 중복 제거를 수행할 때 사용된 알고리즘을 우선순위 큐라고 한다.
```text
SQL에서 AND로 연결된 경우에는 두 조건 중 하나라도 인덱스를 사용할 수 있으면 인덱스 레인지 스캔으로 쿼리가 실행된다.
SQL에서 OR로 연결된 경우에는 둘 중 하나라도 제대로 인덱스를 사용하지 못하면 항상 풀 테이블 스캔으로밖에 처리하지 못한다.
```

### 인덱스 머지 - 정렬 후 합집합 (index_merge_sort_union)
* 인덱스 머지 작업을 하는 도중에 결과의 정렬이 필요한 경우 MySQL 서버는 인덱스 머지 최적화의 'Sort union' 알고리즘을 사용한다.

### 세미 조인 (semijoin)
* 다른 테이블과 실제 조인을 수행하지는 않고, 단지 다른 테이블에서 조건에 일치하는 레코드가 있는지 없는지만 체크하는 형태의 쿼리를 세미 조인이라고 한다.
```sql
SELECT *
FROM employee e
WHERE e.emp_no IN
    (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
```
* 일반적으로 dept_emp 테이블을 조회하는 서브 쿼리 부분이 먼저 실행되고 그다음 employee 테이블에서 일치하는 레코드만 검색할 것으로 예상하지만, 세미 조인 최적화가 없었을 때는 employees 테이블을 풀 스캔하면서 한 건 한 건 서브쿼리의 조건에 일치하는지 비교하게 된다.
* MySQL 서버 8.0 버전부터 세미 조인 쿼리의 성능을 개선하기 위한 최적화 전략은 아래와 같다.
  * Table Pull-out
  * Duplicate Weed-out
  * First Match
  * Loose Scan
  * Materialization
