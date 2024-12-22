# 🎯 인덱스 레인지 스캔

검색해야할 인덱스의 범위가 결정됐을 때 해당 범위의 시작 지점부터 마지막 지점까지 읽는 방식을 말한다.  
인덱스는 항상 정렬되어 있는데 B-Tree 구조에서도 마찬가지다. 이렇게 정렬되어 있는 구조를 통해서 인덱스 레인지 스캔을 할 수 있다.  

### ✅ Ready

대용량의 데이터가 필요하므로 🔗 [Repository: test_db](https://github.com/datacharmer/test_db) 를 참고하여 🔗 [0. data](./0.data/) 디렉토리에 통해 필요한 데이터만 모아 두었습니다. 하나의 MySQL 서버에서 2개의 데이터베이스를 구축하고 하나의 데이터베이스에서는 테이블을 튜닝하고 하나는 하지 않는 것이 비교하면서 확인하기 좋습니다.  

# 🎯 목표에 앞서

사용할 테이블은 🔗 [employees](./0.data/employees.sql) 이므로 간략하게 테이블 구조를 확인하기 바랍니다.  

```sql
mysql> DESCRIBE employees;
+------------+---------------+------+-----+---------+-------+
| Field      | Type          | Null | Key | Default | Extra |
+------------+---------------+------+-----+---------+-------+
| emp_no     | int           | NO   | PRI | NULL    |       |
| birth_date | date          | NO   |     | NULL    |       |
| first_name | varchar(14)   | NO   |     | NULL    |       |
| last_name  | varchar(16)   | NO   |     | NULL    |       |
| gender     | enum('M','F') | NO   |     | NULL    |       |
| hire_date  | date          | NO   |     | NULL    |       |
+------------+---------------+------+-----+---------+-------+
6 rows in set (0.04 sec)
```

튜닝한 테이블과 튜닝하지 않는 테이블을 두면 직접 비교하기 수월해서 2개의 데이터베이스를 생성하고 똑같은 데이터를 준비합니다.  

```sql
mysql> CREATE DATABASE normal;
mysql> CREATE DATABASE tuning;
```

`normal` 데이터베이스의 테이블들은 튜닝하지 않고 `tuning` 데이터베이스의 테이블들은 튜닝합니다.  

# 🎯 실행계획

`employees` 테이블은 `emp_no` 컬럼이 `PRIMARY KEY` 로 설정되어 있으며 현재 다른 인덱스는 없는 상태이다. 총 레코드는 30만개이며 세컨더리 인덱스는 없는 상황이다.  
`emp_no` 컬럼을 기준으로 범위 조건 쿼리의 실행계획을 보면 `type` 이 `range` 인 것을 확인할 수 있다. 이는 `normal`, `tuning` 데이터베이스와 상관없이 `PRIMARY KEY` 를 기준으로 InnoDB 는 인덱스를 생성하기 때문이다.  

- 현재 인덱스를 살펴보자.

```sql
mysql> SHOW INDEX FROM employees;
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299069 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
1 row in set (0.02 sec)
```

- `emp_no` 를 범위 조건으로 두었을 때 실행계획을 살펴보자

```sql
mysql> EXPLAIN SELECT * FROM employees WHERE emp_no > 10300 AND emp_no < 10500;
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | range | PRIMARY       | PRIMARY | 4       | NULL |  199 |   100.00 | Using where |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

`type` 컬럼의 `range` 를 통해 인덱스 레인지 스캔을 사용하는 것을 알 수 있다.  
인덱스가 없는 컬럼을 조건 구문에 넣으면 어떻게 될까?  

- `first_name` 을 범위 조건으로 두었을 때 실행계획을 살펴보자

```sql
mysql> EXPLAIN SELECT first_name FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 299069 |    11.11 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```

`tyep` 컬럼의 `all` 은 테이블 풀 스캔을 뜻하며 최악의 성능을 자랑(?)한다.  
만약 해당 컬럼에 인덱스를 추가하게 되면 `range` 로 바뀌게 되고 테이블 풀 스캔보다 효율적이다.  

실행계획을 수립할 때 샘플링 데이터로 통계 정보를 기준하기 때문에 다소 부정확할 수 있다.  
실제 쿼리를 날렸을 때 얼마나 걸리는지 확인해 보자.  

```sql
mysql> SELECT * FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
...

| 499998 | 1956-09-05 | Patricia       | Breugel          | M      | 1993-10-13 |
| 499999 | 1958-05-01 | Sachin         | Tsukuda          | M      | 1997-11-30 |
+--------+------------+----------------+------------------+--------+------------+
208545 rows in set (0.12 sec)
```

현재 `normal.employees` 와 `tuning.employees` 둘 다 인덱스는 `PRIMARY KEY` 만 있기 때문에 같은 시간이 걸릴텐데 `0.12` 초로 굉장히 빠르게 조회되는 것을 볼 수 있다. 하지만 실행계획에서 알 수 있듯이 테이블 풀 스캔을 하기 때문에 이를 더 단축시킬 수 있다.  

`tuning` 데이터 베이스에 인덱스를 추가해보자.  

```sql
mysql> USE tuning;
mysql> CREATE INDEX ix_first_name ON employees(first_name);
```

새로운 인덱스가 추가되었는지 확인해보고 실행계획과 시간이 얼마나 단축됐는지 확인해보자.  

- `tuning.employees` 테이블에 `first_name` 컬럼으로 인덱스가 생성되었다.  

```sql
mysql> SHOW INDEX FROM employees;
+-----------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name      | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          0 | PRIMARY       |            1 | emp_no      | A         |      299512 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_first_name |            1 | first_name  | A         |        1266 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+---------------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
2 rows in set (0.01 sec)
```

- 아까와 같은 쿼리의 실행 계획을 확인해 보자.  

```sql
mysql> EXPLAIN SELECT * FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows   | filtered | Extra       |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
|  1 | SIMPLE      | employees | NULL       | ALL  | ix_first_name | NULL | NULL    | NULL | 299512 |    50.00 | Using where |
+----+-------------+-----------+------------+------+---------------+------+---------+------+--------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```

인덱스를 추가했음에도 불구하고 여전히 `type=all` 테이블 풀 스캔을 하는데 이는 인덱스만을 통해선 최종 결과를 얻지 못하는 것을 의미한다.  
`SELECT *` 해당 구문은 모든 컬럼을 얻는 구문으로 `first_name` 인덱스만으로는 해당 컬럼의 데이터들을 얻을 수 없기 때문인데,  
`first_name` 으로 인덱스를 생성하면 `emp_no` 와 `first_name` 값만 저장한다. 때문에 이를 제외한 컬럼이 필요하다면 해당 인덱스만으로는 불가하기 때문에 여전히 테이블 풀 스캔을 하는 것이다.  
만약 인덱스에 저장된 값만으로 쿼리를 만족하면 이를 `커버링 인덱스`라 한다. (인덱스만으로 커버가 되기 때문)

1. 인덱스에서 조건을 만족하는 값이 저장된 위치를 찾는다.
2. 1번에서 탐색된 위치부터 필요한 만큼 인덱스를 쭉 읽는다.
3. 2번에서 읽어 들인 키와 레코드 주소를 이용해 레코드가 저장된 페이지를 가져오고, 최종 레코드를 읽어온다.

커버링 인덱스는 3번 과정이 생략된다.  

`first_name` 을 통해 만들어진 인덱스를 통해 커버링 인덱스로 처리되게 바꾸려면 `emp_no` 와 `first_name` 을 조회하면 된다.  

- 아래의 두 실행계획은 인덱스 레인지 스캔을 사용하고 커버링 인덱스로 처리될 수 있다.

```bash
mysql> EXPLAIN SELECT emp_no, first_name FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key           | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_first_name | ix_first_name | 58      | NULL | 149756 |   100.00 | Using where; Using index |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.01 sec)
```

```bash
mysql> EXPLAIN SELECT first_name FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key           | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_first_name | ix_first_name | 58      | NULL | 149756 |   100.00 | Using where; Using index |
+----+-------------+-----------+------------+-------+---------------+---------------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.01 sec)
```

인덱스를 추가하고 커버링 인덱스로 처리될 수 있도록 쿼리를 수정한 뒤 테이플 풀 스캔과 인덱스 레인지 스캔의 시간 차이를 다시 살펴보자.  

```sql
-- DATABASE normal (first_name 인덱스 없음)

mysql> SELECT first_name FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
...
| Patricia       |
| Sachin         |
+----------------+
208545 rows in set (0.08 sec)
```

```sql
-- DATABASE tuning (first_name 인덱스 있음)

mysql> SELECT first_name, emp_no FROM employees WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';
...
| Yinghua        | 497599 |
| Yinghua        | 499003 |
+----------------+--------+
208545 rows in set (0.07 sec)
```

시간을 보면 `0.08` -> `0.07` 로 줄어들었다고 볼 수도 있지만 이것은 오차 허용 범위내이므로 의미가 없다.  
이는 읽기 손익분기점과 관계가 있는데 일반적인 DMBS의 옵티마이저에서 인덱스를 통한 읽기는 4-5배 더 비용이 들기 때문에 인덱스를 통한 읽기가 전체 레코드의 20-25% 를 넘어선다면 비효율적이게 되는 것이다.  
우리가 만든 `ix_first_name` 인덱스는 오름차순으로 정렬되는데 `WHERE first_name BETWEEN 'Georgi' AND 'Yinghua';` 구문에서 읽어야할 레코드가 `G ~ Y` 로 `first_name` 에서 제외되는 값은 a,b,c,d,e,f 그리고 z 밖에 없다. `first_name` 컬럼이 모두 알파벳만으로 이루어져있는 컬럼이라면 인덱스 레인지 스캔은 잘 사용했지만 읽기 손익분기점은 손해를 보는 것이다. 만약 범위를 줄인다면 유의미한 성능 향상을 볼 수 있다.  

```sql
-- DATABASE normal (first_name 인덱스 없음)

mysql> SELECT first_name FROM employees WHERE first_name BETWEEN 'Alejandro' AND 'Basil';
...
| Barton      |
| Aron        |
| Bangqing    |
| Bangqing    |
+-------------+
17886 rows in set (0.07 sec)
```

```sql
-- DATABASE tuning (first_name 인덱스 있음)

mysql> SELECT first_name FROM employees WHERE first_name BETWEEN 'Alejandro' AND 'Basil';
...
| Basil       |
| Basil       |
+-------------+
17886 rows in set (0.02 sec)
```