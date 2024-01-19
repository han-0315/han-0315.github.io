---
layout: post
title: LXC란?
date: 2024-01-19 21:10 +0900 
description: Linux Container에 대해서
category: [Docker, LXC] 
tags: [LXC, Linux Container, 가상화, Docker] 
pin: false
math: true
mermaid: true
---
LXC란?
<!--more-->


### LXC란?


Linux Container의 약자로, 리눅스 인스턴스 내에서 컨테이너를 운용할 수 있게한다. [링크](https://linuxcontainers.org/lxc/introduction/)를 통해 공식문서를 확인할 수 있다. 특징을 살펴보면 Namespace, cgroup, chroots 등 Docker와 상당히 유사하다. VM과 같이 가상환경을 제공하고 싶지만, 리소스 낭비를 최소화하고 싶었던 개발자들은 LXC를 만들었다. 


### Docker와 LXC


LXC가 나온 후 몇년 뒤  Docker가 나왔다. 둘은 약간의 차이가 있는데, LXC는 OS 수준의 가상화를 제공하고, Docker는 응용 수준의 가상화를 제공한다. 리소스 활용은 LXC가 더 효율적이나, 프로젝트를 배포하는 애플리케이션 수준에서는 Docker가 훨씬 편하다. Docker는 리눅스 컨테이너 기술을 이용하여 개발자에게 더 편리한 기능을 제공한다. 

- 이미지: Docker의 이미지는 응용 수준, 예를 들면 웹 애플리케이션과 같은 컨테이너 이미지이다. 하지만 LXC는 OS 수준 이미지다, 웹 애플리케이션을 동작시키려면 컨테이너를 띄운 뒤 안에 접속하여 추가적으로 작업해야 한다.
- Dockfile: Dockerfile 혹은 Docker-compose를 통해 IaC 방식으로 쉽게 컨테이너를 설정할 수 있지만, LXC는 그렇지 않다.
	- 네트워크 설정, 의존성 등 환경설정에 어려움이 많다. **즉, 일관된 환경을 제공하지 못한다.**

### LXC 실행해보기

- LXC 설치

```bash
sudo apt update
sudo apt install lxc
```


LXC도 DockerHub와 같이 원격으로 이미지를 가져올 수 있다. 

- 명령어를 통해 이미지 조회한다.

정말 수많은 이미지들이 조회된다.


```bash
$ lxc image list images:
...
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8373b6a1d8e1 | yes    | Devuan beowulf arm64 (20240116_11:50)      | aarch64      | CONTAINER       | 85.83MB   | Jan 16, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8385f154c5c5 | yes    | Oracle 9 amd64 (20240118_07:46)            | x86_64       | CONTAINER       | 87.60MB   | Jan 18, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8512d064d7dc | yes    | Ubuntu focal amd64 (20240117_07:42)        | x86_64       | VIRTUAL-MACHINE | 261.71MB  | Jan 17, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8522cf094c08 | yes    | Funtoo 1.4 amd64 (20240116_16:45)          | x86_64       | CONTAINER       | 610.49MB  | Jan 16, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8601aa404598 | yes    | Ubuntu jammy arm64 (20240117_07:42)        | aarch64      | VIRTUAL-MACHINE | 290.43MB  | Jan 17, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8668a997ffce | yes    | Almalinux 8 arm64 (20240118_23:08)         | aarch64      | CONTAINER       | 122.22MB  | Jan 18, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
|                                          | 8826d63394af | yes    | Fedora 39 amd64 (20240116_20:33)           | x86_64       | CONTAINER       | 113.67MB  | Jan 16, 2024 at 12:00am (UTC) |
+------------------------------------------+--------------+--------+--------------------------------------------+--------------+-----------------+-----------+-------------------------------+
```

- 바로 컨테이너를 띄우려고 했지만, 아래와 같은 에러가 발생한다.

```bash
root@ip-172-31-12-234:~# lxc launch images:ubuntu/20.04 mycontainer
Creating mycontainer
Error: Failed instance creation: No storage pool found. Please create a new storage pool
```

- 아래의 명령어를 통해 저장소를 붙인다.

```bash
$ lxc storage create default dir
Storage pool default created
$ lxc profile device add default root disk path=/ pool=default
Device root added to default
```


이후 다시 컨테이너를 띄우면 정상적으로 생성되지만, 네트워크를 연결하라는 내용이 나온다.


```bash
$ lxc launch images:ubuntu/20.04 mycontainer
Creating mycontainer
  The instance you are starting doesn't have any network attached to it.
  To create a new network, use: lxc network create
  To attach a network to an instance, use: lxc network attach
```


**[네트워크 연결하기]**

- 브릿지를 하나 생성한다.

	```bash
	$ lxc network create bridge0
	Network bridge0 created
	```

- 생성확인

	```bash
	$ lxc network list
	+---------+----------+---------+-------------+---------+
	|  NAME   |   TYPE   | MANAGED | DESCRIPTION | USED BY |
	+---------+----------+---------+-------------+---------+
	| bridge0 | bridge   | YES     |             | 0       |
	+---------+----------+---------+-------------+---------+
	| eth0    | physical | NO      |             | 0       |
	+---------+----------+---------+-------------+---------+
	```

- 방금 만든 브릿지0을 컨테이너와 붙인다.

	```bash
	$ lxc network attach bridge0 mycontainer eth0
	```

- 컨테이너를 재시작하고, 컨테이너에 간단한 nginx 웹서버를 띄운다.

	```bash
	$ lxc exec mycontainer -- /bin/bash
	root@mycontainer:~# apt-get install -y nginx
	...
	root@mycontainer:~# service nginx start
	```

- 이후 exit 명령어를 통해 컨테이너에서 나오고, 로컬에서 컨테이너 IP에 접속한다.

	```bash
	$ lxc list
	+-------------+---------+----------------------+----------------------------------------------+-----------+-----------+
	|    NAME     |  STATE  |         IPV4         |                     IPV6                     |   TYPE    | SNAPSHOTS |
	+-------------+---------+----------------------+----------------------------------------------+-----------+-----------+
	| mycontainer | RUNNING | 10.165.182.30 (eth0) | fd42:69a:34b8:3727:216:3eff:fef3:34f2 (eth0) | CONTAINER | 0         |
	+-------------+---------+----------------------+----------------------------------------------+-----------+-----------+
	```


	위에서 확인한 컨테이너 IP에 접속한다.


	```bash
	$ curl 10.165.182.30:80
	<!DOCTYPE html>
	<html>
	<head>
	<title>Welcome to nginx!</title>
	<style>
	    body {
	        width: 35em;
	        margin: 0 auto;
	        font-family: Tahoma, Verdana, Arial, sans-serif;
	    }
	</style>
	</head>
	<body>
	<h1>Welcome to nginx!</h1>
	<p>If you see this page, the nginx web server is successfully installed and
	working. Further configuration is required.</p>
	...
	```


### 마무리하며


컨테이너에 네트워크를 정상적으로 설정했다. 초기 컨테이너를 띄우면 아무런 네트워크 설정이 없다. Docker는 자동적으로 브릿지에 모든 컨테이너를 연결하나, LXC는 수동으로 설정해줘야한다. Docker와 유사하지만, 이것저것 설정해야 하는 것이 많아 불편했다. 추가적으로 Docker는 이미지 Layer를 공유하여 캐시로 사용할 수 있찌만 LXC는 지원하지 않는다고 한다. 인턴때만 해도 Docker 컨테이너 이미지 용량이 쌓여 거의 10~30GB로 쌓였다. Layer 방식은 Docker의 큰 장점같다.

