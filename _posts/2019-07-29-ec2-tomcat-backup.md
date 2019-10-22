---
title: EC2 인스턴스에 openJDK 1.8 & tomcat8 설치하기
layout: post
background: '/img/posts/06.jpg'
navigation: true
date: '2019-07-29 19:18:00 +0900'
subclass: post tag-test tag-content
author: p4rksh
categories: p4rksh
---

필자는 EC2 인스턴스에 tomcat8을 설치하여 Spring Boot 애플리케이션을 running 중이다. 
현재까지는 Tomcat과 같은 미들웨어가 있어야 JVM 튜닝 등이 수월하다는 생각으로 Tomcat을 사용중이며, 추후에는 Spring Boot 애플리케이션을 Spring Boot 자체로 running해볼 생각이다. (Spring Boot Embedded Tomcat은 로컬에서 테스트할 때 사용중이다.)

이번 글에서는 새로운 Spring Boot 애플리케이션을 running할 때마다 작업하는 EC2 인스턴스에 openJDK 1.8과 Tomcat8을 설치하는 과정을 정리하고자 한다.

### EC2에 openJDK1.8 설치하기

새로운 EC2(v1)을 생성하였을 때 기본으로 설치되어있는 openJDK는 1.7 버전이다. running할 애플리케이션에서 요구하는 JDK버전이 1.7이라면 상관없겠지만 필자가 개발한 애플리케이션은 1.8을 기반으로 개발하였기 때문에 1.8을 설치해야 한다.

``` html
sudo yum list *openjdk*
```

위 명령어를 실행하면 yum으로 설치할 수 있는 openJDK의 리스트를 조회한다. 그 중 **java-1.8.0-openjdk-devel.x86_64** 를 복사하여

```shell
sudo yum install -y java-1.8.0-openjdk-devel.x86_64
```

위 명령어를 실행하면 openJDK 1.8 버전을 설치한다. (devel은 develop을 의미하며 devel이 붙어있으면 JDK, **java-1.8.0-openjdk.x86_64** 은 JRE라고 보면 된다.)

설치 이후, 해당 리눅스 환경에서 설치한 openJDK 1.8을 바라보게끔 설정해줘야 한다.

```shell
sudo /usr/sbin/alternatives --config java

java -version
```

위 명령어를 실행 후 openJDK 1.8 버전을 선택하도록 한다. 그 후 java -version 명령어를 통해 해당 환경의 JDK 버전을 확인한다.

```shell
sudo yum remove java-1.7.0-openjdk
```

JDK 1.7을 사용하는 환경이 없다면 지워주도록 하자.

### EC2에 Tomcat8 설치하기

JDK 1.8을 설치 완료하였다면 Tomcat8을 설치하도록 하자. 
yum으로 Tomcat을 손쉽게 설치할 수 있지만 필자는 wget으로 Tomcat 공식 홈페이지에 업로드 되어있는 gz파일을 받아와 압축을 푸는것을 선호한다.
(개인적인 생각이지만 압축파일로 받는게 Tomcat의 디렉터리 구조와 Shell 파일들을 튜닝하기 쉽다.)

https://tomcat.apache.org/download-80.cgi

위 링크로 접속하여 가장 최신버전의 core zip 파일의 다운로드 링크를 복사한 후 zip파일을 다운로드 받을 디렉토리로 이동한다.

```shell
sudo wget http://apache.tt.co.kr/tomcat/tomcat-8/v8.5.43/bin/apache-tomcat-8.5.43.tar.gz

tar -xvf apache-tomcat-8.5.43.tar.gz
```

위 명령어를 실행하여 압축파일을 다운로드 후 압축을 풀도록 하자.

```shell
ln -s apache-tomcat-8.5.43 tomcat
```

더 손쉬운 Tomcat 디렉토리 접근을 원한다면 심볼릭 링크를 걸어두도록 한다.

### 글을 마치며
항상 Spring Boot 애플리케이션을 새로운 EC2 환경에서 running할 때 반복적으로 수행하던 작업을 정리해보았다. Spring Boot 애플리케이션을 WAR파일로 압축하여 webapp 디렉토리에 배포 후 startup.sh을 수행하면 Tomcat이 기동될 것이며, 다음엔 Spring Boot 프로젝트를 JAR 파일로 압축하여 배포해보아야겠다.
