# 🎯 Lock

잠금과 트랜잭션은 서로 비슷한 개념 같지만 그렇지 않다.  
잠금은 동시성을 제어하기 위함이고 트랜잭션은 데이터의 정합성을 보장하기 위함이다.  

MySQL의 대표적인 스토리지 엔진인 `InnoDB`와 `MyISAM`이 트랜잭션 지원 여부가 다른데 그 관점에서 어떤 차이가 있는지 살펴보자.  

# 🎯 Transaction

정수타입의 하나의 컬럼만 가진 테이블을 만든다. 테이블은 스토리지 엔진이 다른데, 하나는 `InnoDB`으로 다른 하나는 `MyISAM`이다.

```sql
CREATE TABLE TB_INNODB (
    INNODB_NUMBER INT NOT NULL,
    PRIMARY KEY (INNODB_NUMBER)
) ENGINE=InnoDB;
```

```sql
CREATE TABLE TB_MYISAM (
    MYISAM_NUMBER INT NOT NULL,
    PRIMARY KEY (MYISAM_NUMBER)
) ENGINE=MyISAM;
```

`xxx_NUMBER` 컬럼은 PK임을 염두에 두고 아래와 같이 기본 데이터를 추가한다.

```sql
INSERT INTO TB_INNODB (INNODB_NUMBER) VALUES(3);
INSERT INTO TB_MYISAM (MYISAM_NUMBER) VALUES(3);
```

현재 `TB_INNODB`와 `TB_MYISAM` 테이블에 1개의 레코드만 존재하고 있다.   

```sql
mysql> SELECT * FROM TB_INNODB;
+---------------+
| INNODB_NUMBER |
+---------------+
|             3 |
+---------------+

mysql> SELECT * FROM TB_MYISAM;
+---------------+
| MYISAM_NUMBER |
+---------------+
|             3 |
+---------------+
```

이 상황에서 `xxx_NUMBER` 컬럼에 1,2,3값을 추가하는 쿼리를 실행시켜보자.  

```sql
mysql> INSERT INTO TB_INNODB VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'TB_INNODB.PRIMARY'

mysql> INSERT INTO TB_MYISAM VALUES (1),(2),(3);
ERROR 1062 (23000): Duplicate entry '3' for key 'TB_MYISAM.PRIMARY'
```

두 테이블 모두 3이 이미 존재하므로 쿼리는 실패했는데 실제 테이블에 남아있는 데이터는 차이가 있다.  


```sql
mysql> SELECT * FROM TB_INNODB;
+---------------+
| INNODB_NUMBER |
+---------------+
|             3 |
+---------------+
1 row in set (0.00 sec)

mysql> SELECT * FROM TB_MYISAM;
+---------------+
| MYISAM_NUMBER |
+---------------+
|             1 |
|             2 |
|             3 |
+---------------+
3 rows in set (0.00 sec)
```

`InnoDB`에 경우 트랜잭션을 지원하므로 데이터의 정합성이 보장되었다. 쿼리 중 일부라도 오류가 발생하면 모두 롤백이 되지만  
`MyISAM`에 경우에는 '1'과 '2'는 저장에 성공하고 실패한 데이터 '3'에 대해서만 오류를 발생하고 쿼리가 종료된 것이다.    

이런 경우 필요에 따라 기존에 있는 쓰레기 데이터('1'과 '2')를 제거해주는 쿼리를 작성하거나 애초에 쿼리문을 생성할 때 조건문으로 파악해야 한다.  
쓰레기 데이터를 제거해주는 쿼리도 어렵지만 쿼리를 조건문으로 수행되게 풀어내는 것은 더 까다롭다.    

# 🎯 Caution

트랜잭션은 데이터베이스의 커넥션을 획득한 이후 시작되므로 범위를 최소화하는 것이 좋다. 특히 내부적으로 수행되는 비즈니스 로직과  
외부 API 호출처럼 네트워크 작업이 있는 로직에서 트랜잭션이 같이 묶여버리면 DBMS 서버가 높은 부하 상태로 빠지거나 위험한 상태에 빠지는 경우가 발생한다.  
그러므로 가급적 트랜잭션의 범위를 최소화하고 실제 트랜잭션이 필요한 경우에 적재적소에 사용해야 한다.  