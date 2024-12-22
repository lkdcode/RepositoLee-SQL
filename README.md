# 🎯 MySQL

🐬 My-SQL을 알아보자.  

## 0. Common

- `docker-compose.yml` 파일을 제공해 My-SQL 서버를 띄울 수 있습니다. (port:3333, id:root, pwd:lkdcode)
- `DDL.sql` 과 `xxx.dump` 파일을 제공해 필요한 데이터를 제공합니다.  

### ✅ Ready

1. `docker-compose.yml` 실행하여 MySQL 서버를 띄웁니다.

```bash
$ docker-compose up
```

2. MySQL 서버에 접속합니다.

```bash
$ docker exec -it lkdcode-my-sql bash
$ mysql -u root -p
Enter password: lkdcode
mysql > Welcome to the MySQL ...
```

3. `DDL.sql` 을 실행해 테이블을 생성해줍니다.

```bash
mysql > CREATE TABLE ... ()
```

4. `xxx.dump` 파일을 MySQL 서버로 복사합니다.

```bash
$ docker cp [복사할_덤프_파일] real-my-sql:[복사할_위치]
$ docker cp lkdcode.dump real-my-sql:/tmp/lkdcode.dump
```

5. 복사한 `xxx.dump` 파일을 실행하여 MySQL 서버에 데이터를 추가합니다.

```bash
mysql > SOURCE [복사한_덤프_파일]
mysql > SOURCE /tmp/lkdcode.dump 
```

## 🔗 [1. 기수성(Cardinality)](./1.Cardinality)

기수성(Cardinality)을 기준으로 어떤 쿼리가 효율적인지 옵티마이저가 어떻게 판단하고 우리는 어떤 쿼리를 제공해야할지 간단하게 알아본다.

## 🔗 [2. 인덱스 레인지 스캔](./2.IndexRangeScan)

검색해야할 인덱스의 벙뮈가 결정됐을 때 사용하는 방식이다. 읽기 손익분기점과 커버링 인덱스도 같이 알아본다.  