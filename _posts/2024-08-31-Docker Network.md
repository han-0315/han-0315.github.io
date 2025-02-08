---
layout: post
title: Docker Network
date: 2024-08-31 09:00 +0900 
description: KANS 스터디 1주차 Docker Network
category: [Docker, Network] 
tags: [KANS, CloudNet, Kubernetes, Network, KANS#1] 
pin: false
math: true
mermaid: true
---
KANS 스터디 1주차 Docker Network
<!--more-->


> 💡 **KANS 스터디**  
> CloudNet에서 주관하는 KANS(**K**ubernetes **A**dvanced **N**etworking **S**tudy)으로 쿠버네티스 네트워킹 스터디입니다. 아래의 글은 스터디의 내용을 기반으로 작성했습니다.  
>   
> 스터디에 관심이 있으신 분은 [CloudNet Blog](/c9dfa44a27ff431dafdd2edacc8a1863)를 참고해주세요.




### **네트워크 모델**


Docker 컨테이너에서 사용할 수 있는 다양한 종류의 네트워크 구성을 구현하는 데 사용된다. 주요 네트워크 모델은 아래와 같이 크게 3가지로 나뉜다.


![image.png](/assets/img/post/Docker%20Network/1.png)


출처: [https://towardsdatascience.com/docker-networking-919461b7f498](https://towardsdatascience.com/docker-networking-919461b7f498)


#### 브릿지(Bridge)


컨테이너의 기본 네트워크이다. 브리지 네트워크는 일반적으로 애플리케이션이 독립 실행형 컨테이너에서 실행되고 통신해야 할 때 사용된다. docker-proxy를 통해 호스트와 연결되며 통신이 필요하다면 port 포워딩이 필요하다. 


#### 호스트(Host)


독립 실행형 컨테이너의 경우, 컨테이너와 Docker 호스트 간의 네트워크 격리를 제거하고 호스트의 네트워킹을 직접 사용하기도 한다.


#### 실습


이제 아래에서 실습을 통해 네트워크 모델을 이해한다.

- Docker 명령어를 통해 네트워크 상태를 확인한다.

```bash
$ docker network ls
NETWORK ID     NAME             DRIVER    SCOPE
0e144c808e77   bridge           bridge    local
ecb6e5e600e9   host             host      local
c6576295ec03   none             null      local
```

- 호스트 모드의 컨테이너를 생성한다.

```bash
docker run -d -it --network host --name nginx-host nginx:latest
989b802ec50db95915b896e0d6e2ab4f6a47ab46c215dc482e0e08460827a05b
```

- 네트워크 설정을 확인합니다. 호스트 모드인 것을 확인할 수 있다.

```bash
docker inspect nginx-host | jq '.[].NetworkSettings'
{
  "Bridge": "",
  "SandboxID": "03e7f113e7d9d82f82e8827fdff6707229f293f442628d5c6b25ac160bc3eb30",
  ...
  "Networks": {
    "host": {
      "IPAMConfig": null,
      "Links": null,
      "Aliases": null,
      "NetworkID": "ecb6e5e600e95fc47e2eacccf704eca1b5ec92ba8b4758a9ce630f5548af8a93",
      "EndpointID": "",
      "Gateway": "",
      "IPAddress": "",
      "IPPrefixLen": 0,
      "IPv6Gateway": "",
      "GlobalIPv6Address": "",
      "GlobalIPv6PrefixLen": 0,
      "MacAddress": "",
      "DriverOpts": null
    }
  }
}
```


****호스트 모드이기에, 별도의 포트포워딩 없이 호스트에서 접근할 수 있다.****


```bash
$ curl localhost
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
...
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

- 다음 실습을 위해 컨테이너를 삭제한다.

```bash
$ docker stop nginx-host
nginx-host
$ docker rm nginx-host
nginx-host
```


**[Docker 네트워크는 컨테이너를 브릿지 형태로 연결합니다. 아래의 실습을 통해 이를 직접 확인한다.]**

- 브릿지 모드의 컨테이너를 두개 생성한다.

```bash
$ docker run -d -it --name nginx01 nginx:latest
3c68bb36cf6f3f7065cdc7632851933c25f35d9017dfc02aaff45b2c86b8df7b
$ docker run -d -it --name nginx02 nginx:latest
ef3202dc76a839e010a69a68b77dea140151389da57de9efe585ea9ba6ca4d8b
```

- 아래의 명령어를 통해 브릿지 내용을 확인한다. 두개의 컨테이너가 추가된 것을 볼 수 있다.

```bash
docker network inspect bridge | jq '.[].Containers'
{
  "3c68bb36cf6f3f7065cdc7632851933c25f35d9017dfc02aaff45b2c86b8df7b": {
    "Name": "nginx01",
    "EndpointID": "01c055ac87ac5aac2e2fc962748ba223f486e3e41fccc6cc46df25664e8ba7a1",
    "MacAddress": "02:42:ac:11:00:05",
    "IPv4Address": "172.17.0.5/16",
    "IPv6Address": ""
  },
  "ef3202dc76a839e010a69a68b77dea140151389da57de9efe585ea9ba6ca4d8b": {
    "Name": "nginx02",
    "EndpointID": "2cf03bbf147c855f31c044e251e16f4ed722a934f87c92e2ce9f54ba52babeaf",
    "MacAddress": "02:42:ac:11:00:06",
    "IPv4Address": "172.17.0.6/16",
    "IPv6Address": ""
  },
	...
}
```

- 이제 위의 IP를 바탕으로 두 컨테이너 통신을 시도한다. 통신이 정상적으로 이뤄지는 것을 볼 수 있다.

```bash
$ docker exec -it nginx01 curl 172.17.0.6
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
...
$ docker exec -it nginx02 curl 172.17.0.5
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
...
```

