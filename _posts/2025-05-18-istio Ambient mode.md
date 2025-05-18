---
layout: post
title: istio Ambient mode
date: 2025-05-18 09:01 +0900 
description: istio Ambient mode 살펴보기
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#6, Ambient, proxy, sidecar] 
pin: false
math: true
mermaid: true
---

<!--more-->


### 들어가며


이번 istio 스터디 주차의 주제 중 하나는 튜닝이다. istio는 sidecar 모드로 실행시 모든 파드에 하나의 컨테이너를 더 추가하는 구조이기에, 상당히 성능에 영향이 크다. 특히 파드 네트워크에서 iptables도 사용하니 트래픽이 거치는 경로가 더 많아진다. 여기서는 이런 성능문제를 개선한 **Ambient mode에 대해 살펴본다.**


### **Ambient란** 


Ambient는 기존의 사이드 카 모드를 보완하고자 나온 모드로, **각 파드마다 전용 Proxy 사이드카**를 붙이는 방식이 아닌, **노드 별로 하나의 ztunnel**을 두어 서비스 매시를 구현하는 방식이다. 


*v1.24이 릴리즈되면서 Ambient가 stable 버전으로 올라왔다.


![image.png](/assets/img/post/istio%20Ambient%20mode/1.png)


출처: [https://istio.io/latest/blog/2022/introducing-ambient-mesh/](https://istio.io/latest/blog/2022/introducing-ambient-mesh/)


### 구성요소


Ambient 방식은 ztunnel과 waypoint 2가지 구성요소가 존재한다. 

- ztunnel(**L4 Proxy + mTLS**): hbone을 통해 각 노드간의 mTLS 인증 connection을 제공하며 L4 계층 정책 및 라우팅을 담당한다.
	- 노드 별로 존재하며 daemonsets으로 배포한다.
- waypoint: L7과 관련된 정책 및 라우팅 기능을 추가하는 옵션(L7 Proxy)으로 각 네임스페이스 혹은 서비스 어카운트에 대한 게이트 웨이 역할을 수행한다.
	- 네임스페이스 별로 존재하며 Deployment로 배포한다.
	- 내부 트래픽을 통제하며 East/West Gateway로 보기도 한다.

#### ztunnel


ztunnel로 서비스의 트래픽을 통제하기 위해서는 우선 서비스 트래픽이 ztunnel로 향하도록 해야한다. 이를 위해 CNI Network Plugin과 연동하여 노드로 오는 트래픽이 ztunnel로 향하도록 라우팅한다.


![image.png](/assets/img/post/istio%20Ambient%20mode/2.png)


출처: [https://www.solo.io/blog/traffic-ambient-mesh-istio-cni-node-configuration/](https://www.solo.io/blog/traffic-ambient-mesh-istio-cni-node-configuration/)


ztunnel은 위에서 살펴봤듯이 L4 계층의 라우팅과 각 노드 간의 mTLS 연결을 담당한다. 사이드 카 패턴과 달리 노드 별로 ztunnel은 하나씩 구성하므로 daemonsets으로 배포한다. 또한, 보안에 필요한 아래의 그림과 같이 인증서와 구성정보를 istiod에게 받는다.


![image.png](/assets/img/post/istio%20Ambient%20mode/3.png)


출처: [https://istio.io/latest/docs/ambient/architecture/control-plane/](https://istio.io/latest/docs/ambient/architecture/control-plane/)


받아둔 인증정보를 기반으로 해당 파드끼리는 mTLS(HTTP) Connection을 맺어둔다. 다른 노드 간의 통신을 할 땐 해당 커넥션을 이용한다. 


![image.png](/assets/img/post/istio%20Ambient%20mode/4.png)


출처: [https://istio.io/latest/docs/ambient/architecture/data-plane/](https://istio.io/latest/docs/ambient/architecture/data-plane/)


HBONE(HTTP-Based Overlay Network)은 Istio 구성 요소 간에 사용되는 보안 터널링 프로토콜이다. 아래와 같이 TLS위에서 동작하는 프로토콜로 Connection을 맺어 데이터를 송수신할 수 있다.


![image.png](/assets/img/post/istio%20Ambient%20mode/5.png)


출처: [https://www.solo.io/blog/understanding-istio-ambient-ztunnel-and-secure-overlay/](https://www.solo.io/blog/understanding-istio-ambient-ztunnel-and-secure-overlay/)


#### waypoint


waypoint 프록시는 sidecar와 유사하게 Envoy 기반이며 이것은 기본적으로 네임스페이스(필요에따라 service account)별로 작동한다. 


sidecar 패턴에서는 **모든 서비스(Deployment)에 대한 트래픽 구성** 정보를 알았어야 했지만, waypoint는 네임스페이스 별로 구분하여 **각 네임스페이스에 존재하는 구성**에 대한 정보만 알고있으면 된다. 덕분에 상당한 양의 메모리 리소스를 절약할 수 있다.


![image.png](/assets/img/post/istio%20Ambient%20mode/6.png)


출처: [https://istio.io/latest/blog/2022/ambient-security/](https://istio.io/latest/blog/2022/ambient-security/)


구체적으로 그림을 통해 구성정보 차이를 비교해보자.


아래에서 동그라미는 envoy를 의미한다. 왼쪽처럼 사이드카 패턴에서는 모든 애플리케이션에 대한 구성정보를 담아야하지만, waypoint는 ns별로 구별하므로 각 네임스페이스에 대한 구성만 담아도 된다. 


![image.png](/assets/img/post/istio%20Ambient%20mode/7.png)


위의 그림예시의 4개의 파드로만 비교를 해도 필요한 구성의 양은 sidecar:waypoint = 16:4이다. 파드가 n개라면 사이드카에게 필요한 구성의 양은 $n^2$이지만, waypoint는 n개이다. 이는 파드 모두 다른 애플리케이션이라는 가정이고, 만약 deployment처럼 같은 역할을 하는 파드를 다수 배포한다면 둘의 구성차이는 더 커진다.


만약, n개의 deployment에 대해 replicaset이 m이라면 의 구성은 아래와 같다.


$\text{sidecar}:\text{waypoint} =n *(n * m) : n * k$  (waypoint replicas = k로 둔다)


출처: [https://istio.io/latest/blog/2023/waypoint-proxy-made-simple/](https://istio.io/latest/blog/2023/waypoint-proxy-made-simple/)


### 외부 트래픽


외부에서 들어오는 트래픽을 생각해보자. 그렇다면, 아래와 같은 경로를 거친다.


**Client > External LB > istio-Ingress-gateway pod > ztunnel** 


**> waypoint(option) > ztunnel > application pod**


Virtual Service, Destination Rule에 L7 라우팅이 존재하면 waypoint를 위와 같이 거친다.


![image.png](/assets/img/post/istio%20Ambient%20mode/8.png)


출처: [https://picluster.ricsanfre.com/docs/istio/](https://picluster.ricsanfre.com/docs/istio/)


### 성능(vs sidecar)


#### Latency


mTLS enable한 2개의 프록시를 두고 `http/1.1` protocol, 1kB 데이터로 1000 RPS를 진행했을 때  각 connection 수 (1,2,4,8,16,32,64)에 따른 Latency이다.


![image.png](/assets/img/post/istio%20Ambient%20mode/9.png)


#### Resource


리소스도 절약할 수 있다. 이는 당연한게 per pod에서 per node로 변했으니, 그만큼 리소스가 감소한다. 관련하여 밴치마킹 테스트를 한 결과이다. 1000 RPS 기준으로 sidecar 모드가 0.5 vCPU를 더 사용한다고 한다.


**“In Istio 1.22, a sidecar proxy consumes about 0.5 vCPU per 1000 requests per second.”**


출처: [https://istio.io/latest/docs/ops/deployment/performance-and-scalability/](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)


### Network Policy 변경사항


NetworkPolicy는 L4에 대한 통제를 진행한다. 우리는 HBONE을 통해 트래픽을 전송하기에 이에 대한 설정이 필요한데, Istio는 CNI가 아니며 Network Policy를 관리하지 않고 우회하지도 않는다. “**does not enforce or manage** `NetworkPolicy`**, and in all cases respects it”** 


그렇기에 HBONE에 대한 정책도 열어줘야 한다. (CNI의 Network Policy 구현 방식에 따라 다르겠지만, 차단될 가능성이 있으니 열어두는 것이다.)


우리는 이제 파드 간의 통신에서 HBONE을 사용한다. 만약, 아래와 같은 네트워크 정책이 있었더라면 Ambient 모드 적용시 두번째 네트워크 정책으로 변경되어야 한다.


**(1) Before**


```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  ingress:
  - ports:
    - port: 9090 # 기존에는 9090만 열어줌
      protocol: TCP
  podSelector:
    matchLabels:
      app.kubernetes.io/name: my-app
```


**(2) After**


```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
spec:
  ingress:
  - ports:
    - port: 8080 # HTTP 기반이기에 8080
      protocol: TCP
    - port: 15008 # HBONE 트래픽을 열어주기 위한 포트
      protocol: TCP
  podSelector:
    matchLabels:
      app.kubernetes.io/name: my-app
```


### 기타 자료


eBPF로 사용하자 > kmesh > [https://jimmysong.io/en/blog/introducing-kmesh-kernel-native-service-mesh/](https://jimmysong.io/en/blog/introducing-kmesh-kernel-native-service-mesh/)


deep dive: [https://istio.io/latest/blog/2022/ambient-security/](https://istio.io/latest/blog/2022/ambient-security/)

