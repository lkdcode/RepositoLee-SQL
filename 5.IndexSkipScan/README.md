시작하기 전에 🔗[READY_INDEX.md](../0.common/0.readme/READY_INDEX.md) 를 읽어주세요.  

# 🎯 인덱스 스킵 스캔

인덱스를 생성하면 해당 컬럼들이 쿼리의 조건으로 추가가 돼야 해당 인덱스를 사용하여 결과를 반환할 수 있다.  
만약 인덱스를 다중 컬럼으로 구성한다면 쿼리의 조건에는 해당 컬럼들이 모두 포함되어야 함을 의미하며 선행 컬럼들이 조건 구문에 누락됐다면 인덱스를 사용할 수 없게 된다.  
하지만 인덱스 스킵 스캔은 선행 컬럼 조건 없이도 인덱스를 사용하여 결과를 찾게 해주는 기능이며 선행 컬럼의 기수성이 낮을 때 더 효과적이다.  

어떻게 선행 컬럼 없이도 인덱스 기능을 제공하는지 MySQL 8.0 기준으로 알아보자.  

# 🎯 실행계획  

`employees` 테이블에 `gender`, `hire_date` 컬럼을 기준으로 인덱스를 생성한다.  
```sql
# tuning.employees

mysql> USE tuning;

mysql> CREATE INDEX ix_skip ON employees (gender, hire_date);
Query OK, 0 rows affected (0.35 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> SHOW INDEX FROM employees;
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table     | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| employees |          0 | PRIMARY  |            1 | emp_no      | A         |      299512 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_skip  |            1 | gender      | A         |           1 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| employees |          1 | ix_skip  |            2 | hire_date   | A         |       11987 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
+-----------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
3 rows in set (0.00 sec)
```

`normal.employees` 는 `PRIMARY KEY` 인덱스 하나만 존재하고 `tuning.employees` 는 `PRIMARY KEY`, `ix_skip` 인덱스가 존재한다.  
만약 `ix_skip` 인덱스를 사용하려면 쿼리 WHERE 조건절에 `gender` 컬럼에 대한 비교 조건은 필수이다.  

```sql
# INDEX 를 사용할 수 있는 쿼리
1. mysql > SELECT * FROM employees WHERE gender = 'M';
2. mysql > SELECT * FROM employees WHERE gender = 'M' AND hire_date >= '1985-01-01';

# INDEX 를 사용할 수 없는 쿼리
3. mysql > SELECT * FROM employees WHERE hire_date >= '1985-01-01';
```

다중 컬럼으로 인덱스를 구성하는 경우에는 선행 컬럼의 조건이 필수이며 `3. mysql>..` 쿼리를 수행하기 위해선 `hire_date` 로 구성된 인덱스를 추가해야 한다.  
하지만 MySQL 8.0 부터는 옵티마이저가 `gender` 컬럼을 건너뛰어서 `hire_date` 컬럼만으로도 인덱스를 사용할 수 있게 인덱스 스킵 스캔 최적화 기능이 도입됐다.  
(사실 MySQL8.0 이라면 1,2,3번 모두 실행계획에선 `ix_skip` 을 사용한다.)  

아래의 명령어로 인덱스 스킵 스캔을 비활성화 할 수 있다.  

```sql
# tuning

mysql> SET optimizer_switch='skip_scan=off';
Query OK, 0 rows affected (0.00 sec)
```

이후에 실행 계획을 살펴보자  

```sql
# tuning

mysql> EXPLAIN SELECT gender, hire_date FROM employees WHERE hire_date >= '1985-01-01';
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows   | filtered | Extra                    |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
|  1 | SIMPLE      | employees | NULL       | index | NULL          | ix_skip | 4       | NULL | 299512 |    33.33 | Using where; Using index |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+--------+----------+--------------------------+
1 row in set, 1 warning (0.00 sec)
```

`type=index` 는 인덱스 풀 스캔을 의미하며 우리가 원하는 '효율적으로 인덱스를 사용했다.' 는 아니며, 커버링 인덱스가 가능하므로 위와 같은 실행계획이 수립된 것이다.  
(`EXPLAIN SELECT * FROM employees WHERE hire_date >= '1985-01-01';` 쿼리는 테이블 풀 스캔으로 실행계획이 수립될 것이다.)  

이제 인덱스 스킵 스캔 기능을 활성화 시키고 실행계획을 살펴보자.  

```sql
# tuning

mysql> SET optimizer_switch='skip_scan=on';
Query OK, 0 rows affected (0.00 sec)
```

```sql
mysql> EXPLAIN SELECT gender, hire_date FROM employees WHERE hire_date >= '1985-01-01';
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+-------+----------+----------------------------------------+
| id | select_type | table     | partitions | type  | possible_keys | key     | key_len | ref  | rows  | filtered | Extra                                  |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+-------+----------+----------------------------------------+
|  1 | SIMPLE      | employees | NULL       | range | ix_skip       | ix_skip | 4       | NULL | 99827 |   100.00 | Using where; Using index for skip scan |
+----+-------------+-----------+------------+-------+---------------+---------+---------+------+-------+----------+----------------------------------------+
1 row in set, 1 warning (0.00 sec)
```

실행계획에서 `type=range` 는 인덱스에서 꼭 필요한 부분만 읽었다는 것을 의미하며 `Extra=Using index for skip scan` 은 `ix_skip` 인덱스에 대하여  
인덱스 스킵 스캔을 활용했다는 것을 의미한다. (여전히 `EXPLAIN SELECT * FROM employees WHERE hire_date >= '1985-01-01';` 쿼리는 테이블 풀 스캔일 것이다.)  

인덱스에서 중요한 것은 누누이 말하지만 정렬이며 인덱스 스킵 스캔은 기수성과도 연관이 있다. 옵티마이저는 어떻게 효과적으로 결과를 찾을 수 있을까?  

`ix_skip` 인덱스가 적용된 페이지(리프 노드)는 `gender` 를 먼저 정렬한다. `hire_date` 는 선행컬럼인 `gender` 를 기준으로 정렬될 것이며,  
선행 컬럼인 `gender` 는 'M' 혹은 'F' 값만 가지고 있어 기수성이 낮다. (기수성 2)  
`WHERE hire_date` 을 만족하기 위해 2개의 쿼리를 나눠서 실행하는 것과 비슷한 형태로 최적화를 할 수 있다.  

좀 더 시각화해서 알아보자.  
`ix_skip` 페이지는 아래와 같이 정렬되어 있을 것이다.  

```sql
# tunig.employees ix_skip 리프 노드(페이지)

+--------+------------+
| gender | hire_date  |
+--------+------------+
| M      | 1981-01-01 |
| M      | 1985-01-01 |
| M      | 1989-01-01 |
| M      | 1990-01-01 |
...
| F      | 1985-01-01 |
| F      | 1986-01-01 |
| F      | 1987-01-01 |
| F      | 1998-01-01 |
...
+--------+------------+
```

`hire_date` 도 정렬이 되어있으므로 기수성이 낮은 선행 컬럼을 이용해 2개의 쿼리를 실행한다고 생각해보자.  
```sql
WHERE gender = 'M' AND hire_date >= '1985-01-01';
WHERE gender = 'F' AND hire_date >= '1985-01-01';
```

기수성이 낮으므로 'M' 컬럼에 인덱스 레인지 스캔을 하고 'F' 컬럼에 인덱스 레인지 스캔을 하는 최적화를 떠올릴 수 있다.  

```sql
# tunig.employees ix_skip 리프 노드(페이지)

+--------+------------+
| gender | hire_date  |
+--------+------------+
| M      | 1981-01-01 |
| M      | 1983-01-01 |
| M      | 1989-01-01 | ┌── << 읽기 시작
| M      | 1990-01-01 | │
...                     └── >> gender = 'M' 에 대한 결과
| F      | 1979-01-01 |
| F      | 1982-01-01 |
| F      | 1987-01-01 | ┌── << 읽기 시작
| F      | 1998-01-01 | │
...                     └── >> gender = 'F' 에 대한 결과
+--------+------------+
```

인덱스 스킵 스캔은 선행 컬럼의 조건이 없어도 인덱스를 잘 활용할 수 있는 MySQL 8.0 에 새로 도입된 기능이다. 잘 사용하기 위해선 아래의 조건이 필요하다.  

1. 조건이 누락된 선행 컬럼의 기수성이 낮아야함.
2. 커버링 인덱스.

기수성이 높거나 커버링 인덱스가 불가하다면 오히려 인덱스 풀 스캔으로 실행계획이 수립될 것이다.  