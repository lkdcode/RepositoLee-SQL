시작하기 전에 🔗[READY_INDEX.md](../0.common/0.readme/READY_INDEX.md) 를 읽어주세요.  

# 🎯 인덱스 풀 스캔

인덱스 레인지 스캔과 마찬가지로 인덱스를 사용하지만 인덱스의 처음부터 끝까지 모두 읽는 방식을 뜻한다.  
일반적으로 인덱스의 크기는 테이블의 크기보다 작으므로 직접 테이블을 처음부터 끝까지 읽는 것보다 인덱스만 읽는 것이 효율적이다.  
인덱스 풀 스캔은 인덱스 레인지 스캔보다 효율은 떨어지지만 커버링 인덱스로 처리가 된다면 테이블 풀 스캔보다 더 효율적인 디스크 I/O 로 쿼리를 처리할 수 있다.  

# 🎯 실행계획  

`tuning.employees` 에 인덱스를 생성해준다. 총 3개의 컬럼을 가지고 있으며, 순서에 주의한다.  

```sql
mysql> USE tuning;
mysql> CREATE INDEX ix_full ON employees(first_name, last_name, gender);
```

```sql
# 현재 생성된 인덱스 
mysql> SHOW INDEX FROM employees;
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299512 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_full  |            1 | first_name  | A         |        1314 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_full  |            2 | last_name   | A         |      276618 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_full  |            3 | gender      | A         |      289357 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
4 rows in set (0.01 sec)
```

`ix_full` 인덱스로 처리될 수 있는 쿼리는 `first_name, last_name, gender` 순서를 지킨 컬럼이어야 한다.  
만약 조건절에 `last_name` 컬럼이나 `gender` 컬럼으로 조회하는 경우 인덱스 풀 스캔으로 처리된다.  

```sql
# normal.employees (인덱스 없음)
# tuning.employees (인덱스 있음)

# 인덱스 유무와 상관없이 해당 쿼리는 테이블 풀 스캔으로 동작한다.

mysql> EXPLAIN SELECT * FROM employees;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299069 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```

위에서 알 수 있듯이 인덱스와 상관없이 테이블 풀 스캔으로 실행되는 쿼리이다. 하지만 `ix_full` 인덱스에 있는 컬럼만 조회하게 된다면 인덱스를 사용할 수 있다.  

```sql
# normal.employees (인덱스 없음)
mysql> EXPLAIN SELECT first_name, last_name, gender FROM employees;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299069 |   100.00 | NULL  |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------+
1 row in set, 1 warning (0.01 sec)
```

```sql
# tuning.employees (인덱스 있음)
mysql> EXPLAIN SELECT first_name, last_name, gender FROM employees;
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | index | NULL          | ix_full | 125     | NULL | 299512 |   100.00 | Using index |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```

`tpye=all` 은 테이블 풀 스캔을 뜻하고 `type=inex` 는 인덱스 풀 스캔을 의미한다.
`ix_full` 인덱스는 `first_name`, `last_name`, `gender` 로 이루어져 있으므로 `tuning.employees` 에서는 사용이 가능한 것이다.  

하지만 실제 쿼리 처리 소요시간은 크게 다르지 않다.  

```sql
# normal
mysql> SELECT first_name, last_name, gender FROM employees;
...
| Berhard        | Lenart           | M      |
| Patricia       | Breugel          | M      |
| Sachin         | Tsukuda          | M      |
+----------------+------------------+--------+
300024 rows in set (0.12 sec)
```

```sql
# tuning
mysql> SELECT first_name, last_name, gender FROM employees;
...
| Zvonko         | Zobel            | M      |
| Zvonko         | Zuberek          | F      |
+----------------+------------------+--------+
300024 rows in set (0.13 sec)
```

테이블 풀 스캔은 순차 I/O 로 읽는데 반면 인덱스는 랜덤 I/O 로 읽게 된다.  
하지만 커버링 인덱스가 가능하므로 인덱스 풀 스캔에서 랜덤 I/O 가 발생하지 않고 리프 노드를 순차적으로 탐색하고 쿼리가 종료되므로 성능이 비슷하게 나오는 것이다.  
현재 레코드 수는 약 30만개로 이보다 더 많은 레코드 수를 가지고 있는 테이블이라면 인덱스 풀 스캔의 성능이 더 효율적이라고 볼 수 있다.  
일반적으로 인덱스의 크기는 더 작으므로 I/O 비용을 절감할 수 있다.  
버퍼 풀 메모리는 자주 사용하는 인덱스 및 테이블 페이지를 캐싱하여 디스크 I/O 를 최소화할 수 있는데 인덱스 풀 스캔은 메모리 내에서 처리되는 경우가 많아 성능이 더 우수하다.  

읽기 손익분기점을 잘 고려해야겠지만 인덱스 풀 스캔은 정렬에도 의미가 있다.  
이미 정렬되어 있는 이점을 활용한다면 `ORDER BY` 구문에서 확실한 성능 차이를 볼 수 있다.  

```sql
# normal
# 실행 계획

mysql> EXPLAIN SELECT first_name, last_name, gender FROM employees ORDER BY first_name, last_name;
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra          |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299069 |   100.00 | Using filesort |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+----------------+
1 row in set, 1 warning (0.02 sec)

# 실제 결과

mysql> SELECT first_name, last_name, gender FROM employees ORDER BY first_name, last_name;
...
| Zvonko         | Zobel            | M      |
| Zvonko         | Zuberek          | F      |
+----------------+------------------+--------+
300024 rows in set (0.30 sec)
```

```sql
# tuning
# 실행 계획

mysql> EXPLAIN SELECT first_name, last_name, gender FROM employees ORDER BY first_name, last_name;
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | index | NULL          | ix_full | 125     | NULL | 299512 |   100.00 | Using index |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

# 실제 결과

mysql> SELECT first_name, last_name, gender FROM employees ORDER BY first_name, last_name;
...
| Zvonko         | Zobel            | M      |
| Zvonko         | Zuberek          | F      |
+----------------+------------------+--------+
300024 rows in set (0.08 sec)
```

인덱스 풀 스캔은 테이블 풀 스캔과 다르게 항상 정렬되어 있으므로 위의 쿼리에서 확실하게 차이를 확인할 수 있다.  