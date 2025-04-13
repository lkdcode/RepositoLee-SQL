# 🎯 Tuning

쿼리 튜닝은 병목 현상을 해결하므로 성능 개선에 큰 도움을 준다.  
Java, Python 처럼 Application 계층에서도 해결할 부분이 있지만 Database I/O 의 병목 현상을 해결하는 쿼리 튜닝을 수행해본다.  

상황에 따라 쿼리 튜닝이 달라지므로 상황을 설정하고 어떤식으로 개선했는지 살펴본다.  

# 🎯 시나리오

공장에는 여러대의 설비가 있고 각 설비마다 발생하는 데이터들을 1분 주기로 24시간 내내 수집하고 있다.  
레코드는 `UPDATE`와 `DELETE`는 발생하지 않는 로그성 데이터이다. 해당 데이터들을 분석하고 가공하여 유의미한 결과를 도출해낸다.  

설비 1대당 수집하는 데이터는 `1,440(24시간 * 60분)`개이다.  
설비 1대당 1년에 525,600(`365 * 1,440 = 525,600`) 레코드면 많지 않은 데이터지만, 수집하는 데이터의 종류가 1개가 아니다.  

즉 `N(설비수) * 1,440(1분주기 24시간) * M(데이터 종류)` 가 하루동안 쌓일 총 레코드 수이다.  
특정 설비에 대한 기간별 데이터들을 조회하는 것에 초점을 맞춘다.  

### ✅ Ready

새로 설계하는 것이 아닌 기존에 구축되어 있는 시스템을 수정하는 것이다.  
특징으로는 시퀀스 키가 없으며 복합키만 존재한다.  
`PRDDATE` 컬럼은 `yyyy.MM.dd` 형식으로 날짜 컬럼처럼 보이지만 실제 `VARCHAR(10)` 타입이다.  
샘플 데이터는 실제 데이터들 중 일부를 가져왔으며 `MsSQL` -> `MySQL` 로 변경해 진행한다.  

- DDL

```sql
CREATE TABLE `TB_TEMP1` (
  `DEVICECODE` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `DEVICESUBCODE` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `PRDDATE` varchar(10) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,
  `PRDTIME` datetime NOT NULL,
  `PV_VALUE` float DEFAULT NULL,
  `UNITCODE` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  `MCCODE` varchar(20) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci DEFAULT NULL,
  PRIMARY KEY (`DEVICECODE`,`DEVICESUBCODE`,`PRDDATE`,`PRDTIME`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci;
```

비교를 위해 2개의 테이블을 언급하며 진행한다.  

- `TB_TEMP1`: 쿼리 튜닝 ⭕️ 
- `TB_TEMP2`: 쿼리 튜닝 ❌

주요 컬럼 소개

- DEVICESUBCODE: 데이터 코드
- PRDDATE: yyyy.MM.dd 날짜형식의 문자열
- PRDTIME: 실제 데이터가 수집된 주기 (1분 단위)
- PV_VALUE: 데이터 값
- UNITCODE: 단위
- MCCODE: 설비 코드

# 🎯 Tuning

`TB_TEMP1` 과 `TB_TEMP2` 는 이름만 다를뿐 같은 테이블이며 레코드 수 또한 같다.  

```sql
mysql> SELECT COUNT(*) FROM TB_TEMP1;
+----------+
| COUNT(*) |
+----------+
|  1971231 |
+----------+
1 row in set (0.56 sec)

mysql> SELECT COUNT(*) FROM TB_TEMP2;
+----------+
| COUNT(*) |
+----------+
|  1971231 |
+----------+
1 row in set (0.45 sec)
```

특정 기간동안 조회하는 쿼리의 실행 계획을 보면, `테이블 풀 스캔`으로 처리됨을 알 수 있다.  

```sql
mysql> EXPLAIN SELECT * FROM TB_TEMP1 WHERE PRDTIME >= '2024.05.01' AND PRDTIME < '2024.06.01';
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | TB_TEMP1 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1949291 |    11.11 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 3 warnings (0.01 sec)

mysql> EXPLAIN SELECT * FROM TB_TEMP2 WHERE PRDTIME >= '2024.05.01' AND PRDTIME < '2024.06.01';
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | TB_TEMP2 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1901865 |    11.11 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)
```

튜닝을 하기 앞서 해당 테이블이 어떤 역할로 많이 사용되는지, 어떤 컬럼들이 기준이 되어야하는지 알아야 한다.  
한 번 적재된 레코드에 대한 갱신(`UPDATE`, `DELETE`)이 거의 없고 시간 범위에 대한 조회가 많은 상황이다.  
 
기간을 기준으로 조회하기 위해 `PRDDATE` 컬럼과 `PRDTIME` 컬럼이 있다.  
`PRDDATE` 는 문자열 타입이며 `yyyy-MM-dd` 형식을 가지고 있다. 문자열의 형식을 날짜와 동일하게 가져가더라도 날짜타입(`date`,`datetime`)은 정수형인 `timestamp` 값으로 저장돼서 옵티마이저가 더 유리하게 가져갈 수 있다.  

기간을 조회하기 위해 `PRDTIME` 컬럼을 인덱스로 추가하고 그외에 설비를 식별할 설비 코드인 `MCCODE` 도 추가해준다.  
`DEVICESUBCODE` 는 해당 레코드가 어떤 데이터를 의미하는지 나타내는데, 지표와 관련된 테이블과 조인할 것이므로 `PRDTIME`, `MCCODE`, `DEVICESUBCODE` 가 중요하다고 볼 수 있다.  

기수성이 높은 `PRDTIME`과 각 설비를 구분하기 위해 `MCCODE` 를 인덱스에 포함시켜야 한다.  
이외에 필요한 컬럼들은 커버링 인덱스를 사용할 수 있도록 추가한다.  

# 🎯 1차 튜닝 

`TB_TEMP1` 테이블에 `PRDTIME`, `MCCODE` 로 인덱스를 설정하고 실행계획을 다시 살펴보자.  

```sql
mysql> CREATE INDEX idx_tb_temp1
    -> ON TB_TEMP1 (PRDTIME, MCCODE);
Query OK, 0 rows affected (4.02 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

```sql
mysql> EXPLAIN SELECT * FROM TB_TEMP1 WHERE PRDTIME >= '2024.05.01' AND PRDTIME < '2024.06.01';
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | TB_TEMP1 | NULL       | ALL  | idx_tb_temp1  | NULL | NULL    | NULL | 1949291 |    50.00 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)

mysql> EXPLAIN SELECT * FROM TB_TEMP2 WHERE PRDTIME >= '2024.05.01' AND PRDTIME < '2024.06.01';
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | TB_TEMP2 | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 1901865 |    11.11 | Using where |
+----+-------------+----------+------------+------+---------------+------+---------+------+---------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)
```

`TB_TEMP1` 에 인덱스를 추가했지만 잘 활용할 수 없었다.  
인덱스를 활용할 수 있도록 수정한다면 다음과 같은 쿼리는 해당될 것이다.  

```sql
mysql> EXPLAIN SELECT PRDTIME, MCCODE FROM TB_TEMP1 WHERE PRDTIME >= '2024.05.01' AND PRDTIME < '2024.06.01';
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+--------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key          | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | TB_TEMP1 | NULL       | range | idx_tb_temp1  | idx_tb_temp1 | 5       | NULL | 974645 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+--------+----------+--------------------------+
1 row in set, 3 warnings (0.01 sec)

mysql> EXPLAIN SELECT PRDTIME FROM TB_TEMP1 WHERE PRDTIME >= '2024.05.01' AND PRDTIME < '2024.06.01';
+----+-------------+----------+------------+-------+----------------------+---------+---------+------+--------+----------+----------------------------------------+
| id | select_type | table    | partitions | type  | possible_keys        | key     | key_len | ref  | rows   | filtered | Extra                                  |
+----+-------------+----------+------------+-------+----------------------+---------+---------+------+--------+----------+----------------------------------------+
|  1 | SIMPLE      | TB_TEMP1 | NULL       | range | PRIMARY,idx_tb_temp1 | PRIMARY | 211     | NULL | 216544 |   100.00 | Using where; Using index for skip scan |
+----+-------------+----------+------------+-------+----------------------+---------+---------+------+--------+----------+----------------------------------------+
1 row in set, 3 warnings (0.00 sec)
```

첫 번째는 커버링 인덱스도 성공적이며  
두 번째 쿼리는 복합 인덱스에서 누락된 컬럼이 있어도 옵티마이저(MySQL 8.0이상)가 인덱스 스킵 스캔을 사용했음을 알 수 있다.  
하지만 `PRIMARY`가 포함되었는데 이는 옵티마이저가 복합키에서 필요한 값만 걸러도 빠르다고 판단했기 때문이다. (인덱스가 이미 클러스터링되어 있으니)  

두 번째 쿼리의 실행계획도 충분히 좋지만 강제로 인덱스를 활용하게 할 수 있다.  

```sql
mysql> EXPLAIN SELECT PRDTIME FROM TB_TEMP1 FORCE INDEX (idx_tb_temp1) WHERE PRDTIME >= '2024-05-01' AND PRDTIME < '2024-06-01';
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key          | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | TB_TEMP1 | NULL       | range | idx_tb_temp1  | idx_tb_temp1 | 5       | NULL |    1 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+--------------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.02 sec)
```
