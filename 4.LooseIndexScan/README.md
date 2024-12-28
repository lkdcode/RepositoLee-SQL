시작하기 전에 🔗[READY_INDEX.md](../0.common/0.readme/READY_INDEX.md) 를 읽어주세요.  

# 🎯 루즈 인덱스 스캔

루즈 인덱스 스캔이란 말 그대로 느슨하게 또는 듬성듬성하게 인덱스를 읽는 것을 의미한다.  
인덱스 레인지 스캔은 상반된 의미에서 '타이트 인덱스 스캔'으로 분류하는데 이는 구간내의 모든 인덱스들을 읽기 때문이다.  
루즈 인덱스 스캔은 레인지 스캔과 비슷하게 작동하지만 중간에 필요하지 않은 인덱스 키 값은 무시하고 넘어간다.  

일반적으로 `GROUP BY` 또는 집합 함수 가운데 `MAX()`, `MIN()` 함수에 대해 최적화를 하는 경우에 사용된다.  
역시 정렬과 관계 있다. 인덱스는 기본적으로 '좌측'을 기준하여 정렬한다 그것이 좌측의 문자열이든 좌측의 컬럼이든 간에.  

# 🎯 실행계획  

루즈 인덱스 스캔은 말 그대로 띄엄띄엄 읽는 것이다. 그룹핑, 집합 함수 등을 위해 사용되고 이를 가능케하는 것은 정렬이다.  
인덱스는 항상 좌측을 기준으로 정렬된다. `employees.first_name` 컬럼과 `employees.hire_date` 컬럼만 놓고 시나리오를 작성해보자.  
두 컬럼은 정렬되어 있다. 어떻게 정렬되어 있는가?  

우선 `first_name` 컬럼부터 보면 왼쪽 문자열을 기준으로 정렬한다.  
첫 번째 글자 기준으로 정렬한 후에 두 번째 글자를 기준으로 정렬한다. 그 다음 세 번째 글자를 기준으로 정렬하고.. 마지막 n번째 글자까지 정렬한다.  
마찬가지로 `hire_date` 도 왼쪽을 기준으로 정렬한다. 먼저 왼쪽 컬럼인 `first_name` 을 기준으로 졍렬한 후에 날짜 타입이므로 년-월-일 기준으로 정렬한다.  

정렬의 기준은 왼쪽이다. 그것이 첫 번째 값이든, 왼쪽의 부모 컬럼이든.  
위의 배경지식을 바탕으로 아래의 데이터를 보면 어떤 기준으로 정렬됐는지 알 수 있다.  

```sql
+------------+------------+
| first_name | hire_date  |
+------------+------------+
| Georgi     | 1985-11-21 |
| Georgi     | 1986-06-26 |
| Georgi     | 1989-08-28 |
| Georgi     | 1999-12-01 |
| Kyoichi    | 1981-09-12 |
| Kyoichi    | 1984-06-02 |
| Kyoichi    | 1994-02-10 |
| Saniya     | 1977-09-15 |
| Saniya     | 1981-02-18 |
| Saniya     | 1993-08-24 |
+------------+------------+
```

여기서 `first_name` 기준으로 `hire_date` 의 `MIN()`, `MAX()` 집함 함수를 사용하면 어떻게 되는가?  
모든 데이터를 읽을 필요가 없다. 맨 위에는 최소값이며 맨 아래는 최댓값일 것이다(혹은 그 반대).  

- `first_name` 으로 그룹핑하고 `hire_date` 의 최솟값을 읽으려면?

```sql
+------------+------------+
| first_name | hire_date  |
+------------+------------+
| Georgi     | 1985-11-21 | << 이 부분만 읽고 스킵
| Georgi     | 1986-06-26 |
| Georgi     | 1989-08-28 |
| Georgi     | 1999-12-01 |
| Kyoichi    | 1981-09-12 | << 이 부분만 읽고 스킵
| Kyoichi    | 1984-06-02 |
| Kyoichi    | 1994-02-10 |
| Saniya     | 1977-09-15 | << 이 부분만 읽고 스킵
| Saniya     | 1981-02-18 |
| Saniya     | 1993-08-24 |
+------------+------------+
```

`first_name` 을 그룹핑 했기 때문에 최솟값인 첫 번째 `hire_date` 만 읽고 그 다음 `first_name` 레코드로 넘어가면 된다.  
이렇게 띄엄띄엄 읽는 것을 루즈 인덱스 스캔이라 한다.  

실제 실행 계획과 인덱스가 있는 쿼리와 없는 쿼리를 비교해보자.  
현재 `normal.employees` 와 `tuning.employees` 는 `PRIMARY KEY` 로 생성된 인덱스만 있을 뿐 다른 인덱스는 없다.  

```sql
# normal, tuning

mysql> SHOW INDEX FROM employees;
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299512 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
1 row in set (0.01 sec)
```

루즈 인덱스 스캔을 적용해보기 위해 만들 쿼리는 `employees.first_name` 을 그룹핑하고 `hire_date` 의 최댓값과 최솟값을 구하는 쿼리이다.  
아래와 같이 쿼리문을 작성하고 실행계획을 살펴보자

```sql
# normal.employees

mysql> USE normal;

mysql> EXPLAIN SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra           |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299069 |   100.00 | Using temporary |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
1 row in set, 1 warning (0.00 sec)
```

`type=ALL` 에서 알 수 있듯이 테이블 풀 스캔으로 실행 계획이 수립되고 `Extra=Using temporary` 를 보면 `내부 임시 테이블`을 생성하는 것을 알 수 있다.  
여기서 말하는 `내부 임시 테이블`은  `CREATE TEMPORARY TABLE` 명령어로 만든 임시 테이블과 다른 테이블로 MySQL 엔진이 사용하는 테이블로  
처음에는 메모리에 생성됐다가 테이블의 크기가 거지면 디스크로 옮겨진다(특정 케이스에 대해서는 바로 디스크로 생성될 수 있음).  
우리가 접근할 수도 활용할 수도 없는 이 테이블은 MySQL 엔진이 내부적인 가공을 위해 생성한다.  
다른 세션, 다른 쿼리에서도 접근할 수 없다. 내부적으로 사용되므로 공간이 필요하며 이 역시 사용하지 않는 것보다는 비용을 치뤄야 한다.  

정리하자면 현재 인덱스가 없기에 테이블 풀 스캔과 내부 임시 테이블을 사용하여 보다 비효율적인 결과를 얻는 것을 뜻한다.  
이제 인덱스를 추가하고 실행계획을 다시 살펴보자.  

- 인덱스 추가

```sql
# tuning.employees

mysql> USE tuning;

mysql> CREATE INDEX ix_loose ON employees(first_name, hire_date);
Query OK, 0 rows affected (0.46 sec)
Records: 0  Duplicates: 0  Warnings: 0
```

- 인덱스를 사용한 실행계획

```sql
mysql> EXPLAIN SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_loose      | ix_loose | 58      | NULL | 1260 |   100.00 | Using index for group-by |
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.01 sec)
```


기존의 테이블 풀 스캔이 아닌 `type=range` 로 인덱스 레인지 스캔처럼 특정 범위에 대한 스캔을 의미하고,  
`Extra=Using index for group-by` 를 통해 실제 루즈 인덱스 스캔이 적용됐음을 확인할 수 있다.  
내부 임시 테이블을 사용하지 않는 것처럼 보이지만 실제로는 사용할 수도 있다. 이는 실행계획에 보여지진 않는다.  


인덱스 유무가 똑같은 쿼리에 대해 실행 계획이 어떻게 달라지는가 확인해보자.  

```sql
# normal.employees

mysql> EXPLAIN SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra           |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299069 |   100.00 | Using temporary |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-----------------+
1 row in set, 1 warning (0.00 sec)
```

```sql
# tuning.employees

mysql> EXPLAIN SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key      | key_len | ref  | rows | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_loose      | ix_loose | 58      | NULL | 1260 |   100.00 | Using index for group-by |
+----+-------------+-----------+------------+-------+---------------+----------+---------+------+------+----------+--------------------------+
1 row in set, 1 warning (0.01 sec)
```

실제 처리 속도는 어떨까?  

```sql
# normal.employees

mysql> SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
...
| Khaled         | 1985-02-10     | 1999-06-27     |
| Ravishankar    | 1985-02-16     | 1999-01-22     |
+----------------+----------------+----------------+
1275 rows in set (0.12 sec)
```

```sql
# tuning.employees

mysql> SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
...
| Zsolt          | 1985-02-10     | 1999-02-26     |
| Zvonko         | 1985-02-25     | 1999-02-24     |
+----------------+----------------+----------------+
1275 rows in set (0.01 sec)
```

위의 결과를 통해 루즈 인덱스 스캔과 테이블 풀 스캔의 시간 차이를 확인할 수 있다. 위의 결과를 얻기 위해 MySQL 엔진이 `내부 임시 테이블` 을 사용했는지도 확인해 보자.  

중요한건 `내부 임시 테이블` 이 디스크에까지 쓰였냐 하는 문제인데, 메모리에 올라가는 건 디스크 I/O 를 줄여 성능에 큰 영향을 주지 않는다.  
물론 가변 길이에 대한 메모리 할당이나 트랜잭션 지원, 관련 환경 변수 설정, 스토리지 엔진 등을 확인해야 한다.  

`내부 임시 테이블` 사용 여부 및 횟수를 확인해 보자.  

`SHOW STATUS LIKE 'Created_tmp%';` 해당 명령어를 통해 `내부 임시 테이블`을 확인할 수 있다.  
`Created_tmp_disk_tables` 컬럼은 디스크에 생성된 `내부 임시 테이블`이며,  
`Created_tmp_tables` 컬럼은 메모리에 생성된 `내부 임시 테이블`이다.  

```sql
# normal.employees

# 현재 임시 테이블을 확인한다.
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 0     |
+-------------------------+-------+
3 rows in set (0.02 sec)

# 쿼리 수행
mysql> SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
...
| Khaled         | 1985-02-10     | 1999-06-27     |
| Ravishankar    | 1985-02-16     | 1999-01-22     |
+----------------+----------------+----------------+
1275 rows in set (0.14 sec)

# 쿼리 수행 후 현재 임시 테이블을 확인한다.
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 1     |  << 1 이 증가함.
+-------------------------+-------+
3 rows in set (0.01 sec)

# ... 여러번의 쿼리를 실행한 후
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 4     |  << 추가 쿼리 실행 횟수만큼 증가함.
+-------------------------+-------+
3 rows in set (0.00 sec)
```

`normal.employees` 는 위의 쿼리 결과를 만족하기 위해선 테이블 풀 스캔을 하므로 `내부 임시 테이블`을 생성하게 된다.  
쿼리를 실행할 때마다 `내부 임시 테이블` 이 증가하는 것을 알 수 있다.  

이제 인덱스가 설정되어 있는 `tuning.employees` 쿼리의 결과를 확인해보자.  

```sql
# tuning.employees

# 현재 임시 테이블을 확인한다.
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 0     |
+-------------------------+-------+
3 rows in set (0.02 sec)

# 쿼리 수행
mysql> SELECT first_name, MIN(hire_date), MAX(hire_date) FROM employees GROUP BY first_name;
...
| Zsolt          | 1985-02-10     | 1999-02-26     |
| Zvonko         | 1985-02-25     | 1999-02-24     |
+----------------+----------------+----------------+
1275 rows in set (0.02 sec)

# 쿼리 수행 후 현재 임시 테이블을 확인한다.
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 0     |  << 증가하지 않음.
+-------------------------+-------+
3 rows in set (0.00 sec)


# ... 여러번의 쿼리를 실행한 후
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 0     |  << 증가하지 않음.
+-------------------------+-------+
3 rows in set (0.00 sec)
```

위의 결과를 통해 `내부 임시 테이블` 은 사용되지 않음을 알 수 있다. 이처럼 루즈 인덱스 스캔은 그룹핑, 집합 함수 등 효율적으로 처리할 수 있다.  

참고로 `내부 임시 테이블` 은 세션 단위로 관리되므로 현재 서버를 종료하거나 세션을 종료하면 된다.  

```sql
mysql > exit;

# 재접속 후
mysql> SHOW STATUS LIKE 'Created_tmp%';
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 0     |
| Created_tmp_files       | 93    |
| Created_tmp_tables      | 0     |  << 초기화 됨.
+-------------------------+-------+
3 rows in set (0.00 sec)
```