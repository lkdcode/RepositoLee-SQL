# 🎯 MySQL

🐬 My-SQL을 알아보자.  

### ✅ 시작하기 앞서. 
`docker-compose.yml`, `xxx-ddl.sql`, `xxx.dump` 파일을 통해 MySQL 서버를 띄우고,  
필요한 데이터를 셋업할 수 있습니다. 🔗 [READY_SERVER](0.common/0.readme/READY_SERVER.md)

## 🔗 [1. 기수성(Cardinality)](./1.Cardinality/README.md)

기수성(Cardinality)을 기준으로 어떤 쿼리가 효율적인지 옵티마이저가 어떻게 판단하고 우리는 어떤 쿼리를 제공해야할지 간단하게 알아본다.

## 🔗 [2. 인덱스 레인지 스캔](./2.IndexRangeScan/README.md)

검색해야할 인덱스의 범위가 결정됐을 때 사용하는 방식이다. 읽기 손익분기점과 커버링 인덱스도 같이 알아본다.  

## 🔗 [3. 인덱스 풀 스캔](./3.IndexFullSacn/README.md)

인덱스 레인지 스캔과 마찬가지로 인덱스를 사용하지만 인덱스의 처음부터 끝까지 모두 읽는 방식을 뜻한다.  

## 🔗 [4. 루즈 인덱스 스캔](./4.LooseIndexScan/README.md)

루즈 인덱스 스캔이란 말 그대로 느슨하게 또는 듬성듬성하게 인덱스를 읽는 것을 의미한다.  

## 🔗 [5. 인덱스 스킵 스캔](./5.IndexSkipScan/README.md)

선행 컬럼의 조건없이도 효율적으로 인덱스를 사용할 수 있다.(feat. 기수성)  