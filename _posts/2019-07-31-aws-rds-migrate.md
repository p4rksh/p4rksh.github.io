---
title: Local Maria DB를 AWS RDS로 마이그레이션하기
layout: post
background: '/img/posts/05.jpg'
date: '2019-07-31 22:18:00 +0900'
author: p4rksh
---

회사에서 운영하고 있는 레거시 시스템 중 하나를 버전업하게 되었는데, 해당 애플리케이션은 EC2 인스턴스 내에 MariaDB 엔진을 설치하여 running 중이었다. 버전업과 동시에 DB를 AWS RDS로 마이그레이션 하기로 결정하였는데 그 이유는 아래와 같다.

- paramete group을 통한 간편한 DB 관리
- CloudWatch를 통한 DB 모니터링
- replication 기능을 통한 부하 분산 및 백업
- 스냅샷 기능을 통한 원활한 백업 및 복원

_`'확실히 AWS를 사용하면 사람이 편해진다.'`_  

### 어떤 DB 인스턴스를 사용할까?
마이그레이션을 하면서 어떤 DB인스턴스를 사용해야할지 고민해보았다.  
기본적으로 오토스케일링을 지원하는 서버리스 AuroraDB 사용을 검토해보았으나 해당 서비스는 유저수가 일정하여 부하가 많지 않을것이라고 판단하여 AWS RDS MariaDB로 채택하였다.  
(현재 사내 시스템에선 AuroraDB와 MariaDB엔진을 상황에 맞게 사용중이다.)  

### 어떤 마이그레이션 방법을 사용할까?
마이그레이션 하는 방법은 여러 개가 존재한다. 
AWS의 DMS, Xtra Backup을 사용한 마이그레이션, mysqldump를 사용한 마이그레이션 등등.. 

- [AWS DMS](https://aws.amazon.com/ko/blogs/korea/aws-database-migration-service/)
- [XtraBackup](http://woowabros.github.io/experience/2018/05/28/billingjul.html)

무중단 마이그레이션이 필요하였다면 DMS나 X-tra Backup을 사용하였겠지만 이번 시스템은 무조건 중단을 해야하는 환경이였으므로 mysqldump 명령어를 사용하기로 하였다. 그리고 10G 정도 되는 데이터 양이였기에 mysqldump 명령어로도 짧은 시간안에 마이그레이션이 가능하였다.  

### mysqldump 명령어로 마이그레이션하기
[AWS Docs](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.SmallExisting.html)에도 설명이 되어있지만 아래 명령어로 덤프를 생성하여 바로 AWS RDS로 마이그레이션 할 수 있다. DB 인스턴스가 running 되고있는 서버 환경에 접속하여서 명령어를 사용하면 된다.

```Shell
mysqldump -u <local_user> \
    --databases <database_name> \
    --single-transaction \
    --compress \
    --order-by-primary  \
    --routines=0 \
    --triggers=0 \
    --events=0 \
    -p<local_password> | mysql -u <RDS_user> \
        --port=<port_number> \
        --host=<host_name> \
        -p<RDS_password>
```
**_Parameters_**
- local_user : local DB의 user명.
- database_name : local DB명.
- local_password : local DB의 비밀번호. _`(p다음에 공백없이 비밀번호를 입력해야 한다!)`_
- RDS_user : AWS RDS DB 인스턴스의 DB user.
- port_number : AWS RDS DB 인스턴스의 포트 번호. _`ex) mysql : 3306`_
- host_name : AWS RDS DB 인스턴스의 Host URL(Endpoint).
- RDS_password : AWS RDS DB 인스턴스의 비밀번호. _`(p다음에 공백없이 비밀번호를 입력해야 한다.)`_

**_Options_**
- single-transaction : mysqldump가 local DB의 데이터를 읽어들일 때 local DB가 다른 트랜잭션들을 수용하며 dump의 데이터는 무결성을 유지하게 해주는 옵션. innoDB의 스냅샷을 가져오는 하나의 트랜잭션으로 받아들이기 때문이다.  
_`간단히 말하자면, DB에 Lock을 걸지 않아도 마이그레이션 되는 데이터들의 무결성이 유지된다.`_
- compress : AWS RDS로 마이그레이션 하기 전 local에서 데이터를 압축하여 네트워크 대역폭을 줄임.
- order-by-primary : local DB의 각 테이블마다 primary key를 기준으로 조회하여 dump 시간 단축.
- no-triggers : local DB 테이블들의 trigger를 dump하지 않음. `(밑에 있는 주의할 점 참고)`
- no-routines : local DB 테이블들의 procedure를 dump하지 않음. `(밑에 있는 주의할 점 참고)`

<br>
### 주의할점
1. sys, performance_schema 및 information_schema 스키마는 mysqldump 유틸리티에서 기본적으로 dump에서 제외한다. 즉, DB의 사용자와 권한들을 다시 설정해줘야 한다.
2. Amazon RDS 데이터베이스에서 저장 프로시저, 트리거, 함수 또는 이벤트를 수동으로 만들어야 한다. 필자는 Datagrip을 통해 모든 트리거나 프로시저의 create SQL을 따로 저장 후 실행하였다.

<br>
### 글을 마치며
AWS RDS를 사용해보며 비싸기는 하지만 매우 편리하다고 느끼고 있다. parameter group을 통한 DB 설정값 관리, CloudWatch,  replication 기능들을 통해 devOps에 정말 많은 도움을 받고 있다. 하지만 비용이 무시하지 못할 비용이기에 최적화된 인스턴스 사이즈와 어플리케이션의 튜닝이 필요하다.
