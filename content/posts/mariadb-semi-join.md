---
title: "MariaDB semi-join과 구체화 뷰"
date: 2022-02-16T17:39:28+09:00
tags:
- mariadb
- relationaldb
---

[MariaDB Knowledge Base](https://mariadb.com/kb/en/) 다음 두 포스트를 따라하며 semi-join과 구체화 뷰 대해 이해한 점을 정리한다:
- https://mariadb.com/kb/en/semi-join-subquery-optimizations/
- https://mariadb.com/kb/en/semi-join-materialization-strategy/

한번씩 읽어야 이 글도 이해가 빠를것 같다.

원랜 회사에서 겪은 DB 이슈를 공유하고 싶었는데, 한번에 글로 정리가 안되어 쪼개어 쓴다(DB 이슈는 이 글을 참고해서 이어서 쓸 예정이다).

## 준비
우선 Homebrew로 받은 MariaDB 10.3.32에서 테스트했다:
```sh
❯ mysql --version
mysql  Ver 15.1 Distrib 10.3.32-MariaDB, for osx10.16 (x86_64) using readline 5.1
```

MariaDB Knowlege Base(kb)의 쿼리를 따라하기 위해 샘플 데이터를 받는다. [world](https://downloads.mysql.com/docs/world_x-db.tar.gz)라는 DB인데, 데이터가 kb에 써 있는것과 완전 같진 않지만(kb가 업데이트 안된걸 수도 있다), *얼추* 비슷하다. 발견한 과정은 [여기](https://dba.stackexchange.com/questions/307550/sample-data-for-mariadb-knowledge-base)에 자문자답했다.

다운 받고 압축을 풀었으면 root 계정으로 덤프한다. 파라미터 바꿀 일도 있으니 로컬에서 root 계정을 사용했다:
```sh
mysql -uroot < world.sql 
```

그리고 `city.Population`에 인덱스를 추가한다:
```sql
use world;
CREATE INDEX Population ON city (population);
```

그럼 실습에서 사용할 `city` 와 `country` 테이블 덤프는 다음과 같다:
```sql
CREATE TABLE `city` (
  `ID` int(11) NOT NULL AUTO_INCREMENT,
  `Name` char(35) NOT NULL DEFAULT '',
  `CountryCode` char(3) NOT NULL DEFAULT '',
  `District` char(20) NOT NULL DEFAULT '',
  `Population` int(11) NOT NULL DEFAULT 0,
  PRIMARY KEY (`ID`),
  KEY `CountryCode` (`CountryCode`),
  KEY `Population` (`Population`),
  CONSTRAINT `city_ibfk_1` FOREIGN KEY (`CountryCode`) REFERENCES `country` (`Code`)
) ENGINE=InnoDB AUTO_INCREMENT=4080 DEFAULT CHARSET=utf8mb4;

CREATE TABLE `country` (
  `Code` char(3) NOT NULL DEFAULT '',
  `Name` char(52) NOT NULL DEFAULT '',
  `Continent` enum('Asia','Europe','North America','Africa','Oceania','Antarctica','South America') NOT NULL DEFAULT 'Asia',
  `Region` char(26) NOT NULL DEFAULT '',
  `SurfaceArea` decimal(10,2) NOT NULL DEFAULT 0.00,
  `IndepYear` smallint(6) DEFAULT NULL,
  `Population` int(11) NOT NULL DEFAULT 0,
  `LifeExpectancy` decimal(3,1) DEFAULT NULL,
  `GNP` decimal(10,2) DEFAULT NULL,
  `GNPOld` decimal(10,2) DEFAULT NULL,
  `LocalName` char(45) NOT NULL DEFAULT '',
  `GovernmentForm` char(45) NOT NULL DEFAULT '',
  `HeadOfState` char(60) DEFAULT NULL,
  `Capital` int(11) DEFAULT NULL,
  `Code2` char(2) NOT NULL DEFAULT '',
  PRIMARY KEY (`Code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

이러면 이 포스트에서 참고하는 kb 쿼리를 따라하는데 문제가 없다!

## semi-join subquery
```sql
SELECT ... FROM outer_tables WHERE expr IN (SELECT ... FROM inner_tables ...) AND ...
```
위 쿼리에서 outer_tables의 관심사는 `WHERE IN` 내의 subquery(`SELECT ... FROM inner_tables ...`)에 일치하는것 뿐이다. 다른 조건이 없다면 outer_tables 에서 subquery의 결과와 join하여 간단하게 결과를 만들 수 있다.

준비에서 만든 `city`와 `country` 테이블을 사용한 예제를 살펴보면:
```sql
SELECT * from country 
WHERE
  continent='Europe' AND
  country.code IN (SELECT city.countrycode
                   FROM city 
                   WHERE city.population > 1*1000*1000);
```
이런 쿼리가 된다(kb의 쿼리와 달리 `city.country` 대신 **`city.countrycode`**를 쓰자). 여기서 subquery는 도시 인구가 1M이 넘는 국가코드들이다.

```sql
MariaDB [world]> EXPLAIN SELECT * from country 
    -> WHERE
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population > 1*1000*1000);
+------+--------------+-------------+--------+------------------------+--------------+---------+------+------+------------------------------------+
| id   | select_type  | table       | type   | possible_keys          | key          | key_len | ref  | rows | Extra                              |
+------+--------------+-------------+--------+------------------------+--------------+---------+------+------+------------------------------------+
|    1 | PRIMARY      | country     | ALL    | PRIMARY                | NULL         | NULL    | NULL |  239 | Using where                        |
|    1 | PRIMARY      | <subquery2> | eq_ref | distinct_key           | distinct_key | 12      | func |    1 |                                    |
|    2 | MATERIALIZED | city        | range  | CountryCode,Population | Population   | 4       | NULL |  237 | Using index condition; Using where |
+------+--------------+-------------+--------+------------------------+--------------+---------+------+------+------------------------------------+
```

`EXPLAIN` 2에 `MATERIALIZED` 타입이 있다. 이것을 `<subquey2>`라는 이름의 임시 테이블(`EXPLAIN` 결과의 두번째 레코드)로 넘겨주어 outer query에서 구체화 뷰로서 참조한다. 구체화 뷰는 임시 테이블을 매번 생성하지 않고 바깥 쿼리 실행동안 저장해두고 쓰게 된다(semi-join에서 사용하는 방법이라 뒤에 이어 설명한다).

즉 첫번째 kb 글의 다음 그림들:

![1](/images/mariadb-semi-join/1.png)
![2](/images/mariadb-semi-join/2.png)

에서 회색부분이 구체화 뷰(=서브쿼리의 결과인 임시테이블)이고, kb 두번째 글의 그림을 보면 더 명확하다:

![3](/images/mariadb-semi-join/3.png)

(두 글을 왔다갔다 보며 이해하는데 오래 걸렸다...)

내가 준비한 데이터 기준으론 3 국가보다 훨씬 많다(76개국). 서브쿼리는 `IN` 안에서 실행하니 결과 레코드를 `DISTINCT` 했다:
```sh
❯ mysql world -uroot -Nse "SELECT DISTINCT city.countrycode FROM city WHERE city.population > 1*1000*1000" | tr '\n'  ','
MEX,ITA,RUS,UKR,GBR,IND,JPN,CAN,MOZ,CHN,IRN,BRA,IDN,GHA,KOR,GIN,TUR,AUS,LBN,BGR,PAK,KAZ,USA,PHL,ARG,CZE,DEU,YUG,COL,GEO,URY,ARM,SYR,SDN,MYS,VEN,HKG,ZMB,CMR,BGD,VNM,ZWE,NGA,TWN,ESP,ECU,AUT,DOM,POL,BLR,LBY,TZA,AFG,AZE,HUN,ROM,AGO,SAU,UZB,FRA,DZA,EGY,CUB,KEN,ZAF,PRK,ETH,CIV,MAR,MMR,SGP,IRQ,CHL,COD,THA,PER
```

`city`(inner table)과 `country`(outer table)의 순서가 뒤집는게 가능하다는 것도 조금은 이해가 될 것이다. 구체화 뷰는 서브쿼리에서 만들지만, 옵티마이저는 이걸 먼저 만들고 양쪽 테이블에서 참조할 것이기 때문이다(이는 세미 조인 최적화와 관련 있기 때문에 뒤에서 자세히 설명한다).

### 조건
반대로 구체화 뷰를 양방향에서 (outer -> inner 그리고 inner -> outer) join 가능하지 않는다면 semi-join을 할 수 없다. 예시 쿼리는 서브쿼리인 ***대도시(인구 1M 초과)***와 표면적 0.1M(`100*1000`, 단위는 아마 제곱 km 같다)이 넘는 조건을 OR로 묶어준다.

```sql
SELECT * from country 
WHERE
  continent='Europe' AND
  (country.code IN (SELECT city.countrycode
                   FROM city 
                   WHERE city.population > 1*1000*1000)
	OR country.surfacearea > 100*1000);
```

이러면 `city`(inner) -> `country`(outer) 방향으로 구체화 뷰와 join 할 수 없다. `country.surfacearea`를 구체화 뷰 바깥의 ***땅이 넓은 나라***도 출력해야하기 때문이다.

![4](/images/mariadb-semi-join/4.png)

따라서 구체화 뷰를 만들지만 outer table에서 subquery로 참조하지 않는다(=semi-join하지 않는다):
```sql
MariaDB [world]> EXPLAIN SELECT * from country 
    -> WHERE
    ->   continent='Europe' AND
    ->   (country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population > 1*1000*1000)
    -> OR country.surfacearea > 100*1000);
+------+--------------+---------+-------+------------------------+------------+---------+------+------+-----------------------+
| id   | select_type  | table   | type  | possible_keys          | key        | key_len | ref  | rows | Extra                 |
+------+--------------+---------+-------+------------------------+------------+---------+------+------+-----------------------+
|    1 | PRIMARY      | country | ALL   | NULL                   | NULL       | NULL    | NULL |  239 | Using where           |
|    2 | MATERIALIZED | city    | range | CountryCode,Population | Population | 4       | NULL |  237 | Using index condition |
+------+--------------+---------+-------+------------------------+------------+---------+------+------+-----------------------+
```

### inner joins와 차이점
inner join은 조건에 맞는(`ON`) outer table의 레코드 수만큼 결과를 출력한다. semi-join은 조건에 맞는 레코드 하나만 출력한다.

인구 1M 이상의 대도시가 있는 유럽 국가는 총 15개국이다(독일이 하나 있는 것은 당연하다):
```sql
MariaDB [world]> SELECT * FROM (SELECT * from country 
    -> WHERE
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population > 1*1000*1000)) AS country_with_big_city WHERE name='Germany';
+------+---------+-----------+----------------+-------------+-----------+------------+----------------+------------+------------+-------------+------------------+--------------+---------+-------+
| Code | Name    | Continent | Region         | SurfaceArea | IndepYear | Population | LifeExpectancy | GNP        | GNPOld     | LocalName   | GovernmentForm   | HeadOfState  | Capital | Code2 |
+------+---------+-----------+----------------+-------------+-----------+------------+----------------+------------+------------+-------------+------------------+--------------+---------+-------+
| DEU  | Germany | Europe    | Western Europe |   357022.00 |      1955 |   82164700 |           77.4 | 2133367.00 | 2102826.00 | Deutschland | Federal Republic | Johannes Rau |    3068 | DE    |
+------+---------+-----------+----------------+-------------+-----------+------------+----------------+------------+------------+-------------+------------------+--------------+---------+-------+
1 row in set (0.003 sec)
```

하지만 inner join 한 결과는 각국의 도시의 수만큼 중복되어 많아진다. `DISTINCT`해야 semi-join한 결과와 같다(kb의 데이터와 달리 독일의 대도시는 93이나 된다. 아마 예전? 데이터 기준인듯..):
```sql
MariaDB [world]> SELECT COUNT(*) FROM country INNER JOIN city ON country.code = city.countrycode
    -> WHERE 
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population>1*1000*1000);
+----------+
| COUNT(*) |
+----------+
|      709 |
+----------+
1 row in set (0.001 sec)

MariaDB [world]> SELECT COUNT(*) FROM (SELECT country.name FROM country INNER JOIN city ON country.code = city.countrycode
    -> WHERE 
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population>1*1000*1000)) AS country_with_big_city WHERE name='Germany';
+----------+
| COUNT(*) |
+----------+
|       93 |
+----------+
1 row in set (0.005 sec)

MariaDB [world]> SELECT COUNT(DISTINCT (country.name)) FROM country INNER JOIN city ON country.code = city.countrycode
    -> WHERE 
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population>1*1000*1000);
+--------------------------------+
| COUNT(DISTINCT (country.name)) |
+--------------------------------+
|                             15 |
+--------------------------------+
1 row in set (0.010 sec)
``` 

## 구체화 전략
구체화 전략엔 lookup과 scan 두 가지가 있다. 각 전략은 구체화를 outer, inner 테이블 중 어디서 시작해 어느 방향으로 할 것인가와 관련 있다.

### Lookup
구체화 뷰를 lookup 테이블로 활용하는 전략이다. 위 대도시 국가 쿼리의 경우가 그렇다. `EXPLAIN`를 다시 보자:
```sql
MariaDB [world]> EXPLAIN SELECT * from country 
    -> WHERE
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population > 1*1000*1000);
+------+--------------+-------------+--------+------------------------+--------------+---------+------+------+------------------------------------+
| id   | select_type  | table       | type   | possible_keys          | key          | key_len | ref  | rows | Extra                              |
+------+--------------+-------------+--------+------------------------+--------------+---------+------+------+------------------------------------+
|    1 | PRIMARY      | country     | ALL    | PRIMARY                | NULL         | NULL    | NULL |  239 | Using where                        |
|    1 | PRIMARY      | <subquery2> | eq_ref | distinct_key           | distinct_key | 12      | func |    1 |                                    |
|    2 | MATERIALIZED | city        | range  | CountryCode,Population | Population   | 4       | NULL |  237 | Using index condition; Using where |
+------+--------------+-------------+--------+------------------------+--------------+---------+------+------+------------------------------------+
```

- id=2에 해당하는 `MATERIALIZED`는 모든 컬럼에 unique key constraint가 있는 임시 테이블이다.
- outer table 인 Country에서 semi-join할 수 있게 이를(table `<subquery2>`) 올린다. 
- `country` 테이블에서 구체화 뷰를 사용해 index loopkup하여(`eq_ref`) semi-join한다


## Scan
대도시의 기준을 더 깐깐히 해보자. 옵티마이저는 작아진 구체화 뷰를 full scan하는 전략을 쓴다(반대로 lookup을 outer table에서 하게 된다).
```sql
MariaDB [world]> EXPLAIN SELECT * from country 
    -> WHERE
    ->   continent='Europe' AND
    ->   country.code IN (SELECT city.countrycode
    ->                    FROM city 
    ->                    WHERE city.population > 7*1000*1000);
+------+--------------+-------------+--------+------------------------+------------+---------+------------------------+------+-----------------------+
| id   | select_type  | table       | type   | possible_keys          | key        | key_len | ref                    | rows | Extra                 |
+------+--------------+-------------+--------+------------------------+------------+---------+------------------------+------+-----------------------+
|    1 | PRIMARY      | <subquery2> | ALL    | distinct_key           | NULL       | NULL    | NULL                   |   14 |                       |
|    1 | PRIMARY      | country     | eq_ref | PRIMARY                | PRIMARY    | 12      | world.city.CountryCode |    1 | Using where           |
|    2 | MATERIALIZED | city        | range  | CountryCode,Population | Population | 4       | NULL                   |   14 | Using index condition |
+------+--------------+-------------+--------+------------------------+------------+---------+------------------------+------+-----------------------+
3 rows in set (0.000 sec)
```

구체화 뷰(`MATERIALIZED`)는 똑같이 `city`(inner table)에서 만든다. 구체화 전략을 간단하게 판단하는 법은 `EXPLAIN` 결과 table이 `<subquery2>`인 레코드의 type이 `ALL`일 경우 scan, `eq_ref`이면 lookup으로 판단할 수 있다.

## 시스템 변수
[`optimizer_switch`](https://mariadb.com/kb/en/server-system-variables/#optimizer_switch) 변수에 `materialization=on`과 `semijoin=on`으로 설정되어야 semi-join 구체화 전략을 사용할 수 있다.

옵션이 나눠져 있듯, 구체화와 semi-join은 독립적으로 적용될 수 있다(`EXPLAIN` 레코드가 나뉘어진걸 보고 유추할 수 있다). [semi-join을 하지 않는 구체화도 있다](https://mariadb.com/kb/en/non-semi-join-subquery-optimizations/#materialization-for-non-correlated-in-subqueries)고 한다.


## 정리
- semi-join은 subquery(from inner table)와 outer table을 join처럼 가볍게 연산하는 동작
- 이때 subquery를 구체화 뷰로 저장하여 씀
- 구체화 전략을 임시 테이블을 full scan할지 lookup 테이블로 쓸지 두가지가 있음
  - `EXPLAIN`의 `<subquery2>` 테이블 레코드의 type으로 구분하면 쉽다
  - scan: `ALL`
  - lookup: `eq_ref`

정리하는게 너무 어려웠다. 특히 구체화 전략 부분은 쿼리 결과에서 눈으로 보이는게 아니라 어려웠다. `EXPLAIN`을 잘 볼 줄 알아야 하는데, 자주 쓰지 않다보니 만날 때마다 짜릿하다... 그래도 type에 대한 해석은 조금 늘은거 같다.

구체화 뷰는, DDIA 읽었을 땐 OLAP를 위한 특수한 형태의 DB 테이블 또는 콜렉션이라고만 생각했는데, 이미 알고 있는 RDB 내부에서도 쓰인다는 사실이 새로웠다.

원랜 [in_predicate_conversion_threshold](https://mariadb.com/docs/reference/mdb/system-variables/in_predicate_conversion_threshold/)란 시스템 변수에 대해 정리하고, 이를 튜닝한 경험을 소개하고 싶었다. 정확히 이해하고 튜닝한게 아니라 튜닝해서 테스트하니 성능이 좋아서 프로덕션에 도입한거라 찜찜한 상태이다. 이 변수 역할을 이해하기 위해서 semi-join 이해가 먼저 필요할거 같아 정리했는데 너무 오래걸렸다. 회사 개발자들과도 공유하고 semi-join, 구체화 뷰와 전략 그리고 변수 튜닝에 대해 이야기 해봐야겠다.

DBA 되는줄 알았는데 어림도 없지. DBA 스택오버플로 가입하고 글쓴게 소득인가 싶다 ㅎㅎ;

## 참고
- https://engineering.linecorp.com/ko/blog/mysql-workbench-visual-explain-index/
