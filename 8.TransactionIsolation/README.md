# 🎯 Transaction Isolation Level

# ✅ 시작에 앞서

트랜잭션 격리 수준을 테스트하기위한 테이블과 데이터 입니다.

- DDL

```sql
CREATE TABLE TB_MEMBERS (
    id        INT AUTO_INCREMENT PRIMARY KEY,
    name      VARCHAR(50) NOT NULL,
    age       INT NOT NULL,
    nickname  VARCHAR(30) UNIQUE
) ENGINE=InnoDB;
```

- DML

```sql
INSERT INTO TB_MEMBERS (name, age, nickname) VALUES
('Alice', 25, 'alice123'),
('Bob', 30, 'bobby'),
('Charlie', 28, 'charlieX');
```

# 🎯 READ UNCOMMITTED

```mermaid
sequenceDiagram
    participant Session1
    participant MySQLDatabase
    participant Session2
    Note over Session1, Session2: SET autocommit=OFF
    Note over Session1, Session2: SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED

    Session1 ->>MySQLDatabase: BEGIN
    Session1 ->>MySQLDatabase: INSERT INTO TB_INNODB<br/>VALUES(12)

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 1건

    Session1 ->>MySQLDatabase: ROLLBACK

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 0건
```

# 🎯 READ COMMITTED

```mermaid
sequenceDiagram
    participant Session1
    participant MySQLDatabase
    participant Session2
    Note over Session1, Session2: SET autocommit=OFF
    Note over Session1, Session2: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED

    Session1 ->>MySQLDatabase: BEGIN
    Session1 ->>MySQLDatabase: INSERT INTO TB_INNODB<br/>VALUES(12)

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 1건

    Session1 ->>MySQLDatabase: ROLLBACK

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 0건
```

# 🎯 REPEATABLE READ

```mermaid
sequenceDiagram
    participant Session1
    participant MySQLDatabase
    participant Session2
    Note over Session1, Session2: SET autocommit=OFF
    Note over Session1, Session2: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED

    Session1 ->>MySQLDatabase: BEGIN
    Session1 ->>MySQLDatabase: INSERT INTO TB_INNODB<br/>VALUES(12)

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 1건

    Session1 ->>MySQLDatabase: ROLLBACK

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 0건
```

# 🎯 SERIALIZABLE

```mermaid
sequenceDiagram
    participant Session1
    participant MySQLDatabase
    participant Session2
    Note over Session1, Session2: SET autocommit=OFF
    Note over Session1, Session2: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED

    Session1 ->>MySQLDatabase: BEGIN
    Session1 ->>MySQLDatabase: INSERT INTO TB_INNODB<br/>VALUES(12)

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 1건

    Session1 ->>MySQLDatabase: ROLLBACK

    Session2 ->>+MySQLDatabase: SELECT * <br/>FROM TB_INNODB<br/>WHERE INNODB_NUMBER = 12
    MySQLDatabase ->>-Session2: 결과 0건
```