# ✅ Ready

대용량의 데이터가 필요하므로 🔗 [Repository: test_db](https://github.com/datacharmer/test_db) 를 참고하여 🔗 [1. index-data](../1.index-data/) 디렉토리에 통해 필요한 데이터만 모아 두었습니다. 하나의 MySQL 서버에서 2개의 데이터베이스를 구축하고 하나의 데이터베이스에서는 테이블을 튜닝하고 하나는 하지 않는 것이 비교하면서 확인하기 좋습니다.  

# 🎯 목표에 앞서

사용할 테이블은 🔗 [employees](../1.index-data/employees.sql) 이므로 간략하게 테이블 구조를 확인하기 바랍니다.  

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
```

튜닝한 테이블과 튜닝하지 않는 테이블을 두면 직접 비교하기 수월해서 2개의 데이터베이스를 생성하고 똑같은 데이터를 준비합니다.  

```sql
mysql> CREATE DATABASE normal;
mysql> CREATE DATABASE tuning;
# 이후 normal 과 tuning에 데이터 추가.
```

`normal` 데이터베이스의 테이블들은 튜닝하지 않고 `tuning` 데이터베이스의 테이블들은 튜닝합니다.  