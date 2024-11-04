---
layout: post
title: kube-proxy vs cni
date: 2024-03-21 09:00 +0900 
description: kube-proxy와 CNI 차이 비교
category: [Kubernetes, Network] 
tags: [Kubernetes, kube-proxy, cni, calico] 
pin: false
math: true
mermaid: true
---


kube-proxy와 CNI 차이 비교
<!--more-->


## kube-proxy vs CNI


오랜만에 쿠버네티스 네트워크를 다시 정리하는데, 커피고래님의 좋은 [포스트](https://coffeewhale.com/packet-network3#kube-proxy-iptable-mode)를 발견하여 학습한 내용을 다시 한번 정리한다. 


간단하게 요약하자면, kube-proxy는 서비스 리소스의 네트워크를 구현하며, cni는 파드 네트워크를 구현한다. 


## kube-proxy


여기서는 iptables 모드를 기준으로 설명한다. kube-proxy는 서비스의 가상 IP에 대한 트래픽을 캡처하고 올바른 파드로 리디렉션하도록 iptables 규칙을 구성한다.


iptables 모드로 설정된 경우 netstate 명령어로는 listen port를 확인할 수 없다. iptables 자체는 커널 단에서 특정 트래픽이 들어오면, 변환하는 역할을 진행하는 것이기 때문이다.


### 작동 방식


[서비스 네트워크 이해하기](https://www.handongbee.com/posts/%EC%84%9C%EB%B9%84%EC%8A%A4-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EB%AA%A8%EB%8D%B8-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0/) 포스트에서 Netfilter를 통해 서비스로 향하는 트래픽이 파드에게 라우팅된다는 것을 알 수 있었다. 여기서는 목적지 파드에서 다시 출발지 파드에게 응답을 보낼 때, 어떻게 작동되는 지 알아본다. 


이때 사용되는 것은 conntrack 모듈이다. conntrack은 iptables의 상태 추적 모듈이다. 이제 응답이 들어오는 작동과정을 자세히 알아보자.


> 💡 **예시 상황**  
> 1번 파드에서 서비스를 통해 2번 파드에게 트래픽을 전달했다. 2번 파드에서 1번 파드로 응답을 보낸 상황이다.


트래픽을 전송할 때, 임시포트를 사용한다. 송수신하는 곳에서 임시포트와 IP에 대한 테이블을 통해 내가 보낸 요청에 대한 응답인지 확인할 수 있다. 즉,  `SRC IP:PORT`, `DST IP:PORT` 정보가 맞아야 트래픽을 수신한다. 파드1번은 “SRC: 파드1, DST:서비스 IP”로 트래픽을 전송했기에 응답으로 오는 트래픽의 정보도 “SRC: 서비스 IP, DST: 파드1”이어야 한다. 


DNAT에는 conntrack이 상태 머신을 이용하여 연결 상태를 유지한다고 한다. 그렇기에 2번 파드에서 1번 파드로 응답을 보내면 이를 kube-proxy가 추적하여 SRC의 정보를 파드2에서 서비스IP로 변경한다. 


덕분에 파드1에서는 응답으로 오는 트래픽을 수신할 수 있다.


### NetworkPolicy


네트워크정책도 `iptables` 를 통해 작동한다. 예를 들어 파드1, 파드2, 파드3 리소스가 있을 때 네트워크 정책으로 파드1으로 접속하는 트래픽은 파드2만 가능하도록 설정한다. 아래의 정책과 같은 모습이다.


```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-access
spec:
  podSelector:
    matchLabels:
      name: "pod1"
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: "pod2"
```


그렇다면 **파드3**이 **파드1**에게 트래픽을 보내면, 트래픽은 **파드3**에서 **파드1**로 전송된다. Services의 동작과 마찬가지로 전송과정에서 중간에 커널을 거친다. iptables 관련 설정으로 파드1로 향하는 트래픽의 SRC가 파드2의 IP인 경우에만 허가되며, 나머지는 드랍하는 규칙이 존재한다. 커널에서 규칙에 따라 파드2의 IP가 아니면 트래픽이 드랍된다.


## CNI


Kube-proxy가 서비스 네트워크를 담당했다면, CNI는 파드 네트워크를 담당한다. 아래에서 주로 설명하는 CNI는 Calico이다. 


### CNI 명세서


[GitHub](https://github.com/containernetworking/cni/blob/main/SPEC.md#overview)에서 간단하게 CNI의 필수요구사항을 확인할 수 있다. 


> **Summary**

	1. A format for administrators to define network configuration.
	2. A protocol for container runtimes to make requests to network plugins.
	3. A procedure for executing plugins based on a supplied configuration.
	4. A procedure for plugins to delegate functionality to other plugins.
	5. Data types for plugins to return their results to the runtime.

### 요구사항

1. IPAM(IP address management)의 역할을 수행해야 한다. 파드가 생겨나면 IP를 할당하고, 파드가 종료되면 IP를 회수한다.
2. 다른 노드에게 IP 라우팅 정보 공유
3. 호스트에 라우팅 정보 삽입
4. 네트워크 정책에 맞는 트래픽처리

### 사전지식


동작과정의 그림을 보기전에 아래의 지식을 알면 좋다.


`10.0.2.11 dev cali123 scope link`와 같이 기술되어있다면, 다음과 의미가 같다. 

- 목적지 주소: 10.0.2.11
- 네트워크 인터페이스: dev cali123
- scope(범위): 링크(로컬)

즉, **10.0.2.11**로 향하는 트래픽을 **cali123**로 라우팅한다는 규칙이다. 


다른 하나의 규칙을 확인해보자.


`10.0.1.0/24 via 192.168.1.10 dev eth0/tunl0`

- 목적지: 10.0.1.0/24
- 중간거점: 192.168.1.10
- 네트워크 인터페이스: eth0 혹은 tunl0

즉 10.0.1.0/24의 트래픽을 발견하면 192.168.1.10를 거쳐 eth0 혹은 tunl0으로 라우팅한다는 규칙이다.


### 동작과정


Calico는 여러 개의 모듈이 존재한다. BIRD(BGP), ConfD, Felix 가 핵심역할을 수행한다. 

- BIRD(BGP): 데몬으로, 다른 노드의 BGP와 라우팅 정보를 교환한다.
	- 즉, 각 파드의 대역별로 어떤 노드로 라우팅해야 하는지 알려준다.
- ConfD: 설정관리 도구로, 네트워크와 서브넷에 대한 설정값을 반영하고 이를 BIRD 데몬이 이해할 수 있도록 변환한다. 즉, 설정이 달라지면 BIRD 데몬이 이를 감지하고 다른 노드의 데몬에게 알려줄 수 있다.
- Felix: etcd로 부터 정보를 읽어 라우팅 테이블을 만든다. 라우팅 테이블은 kube-proxy 모드에 맞게 조작한다.
- Proxy-ARP: 해당 네트워크에 존재하지 않는 ARP 요청에 대한 대리 응답이 가능하다. 여기서 cali123이 Proxy-ARP 역할을 수행한다.

![Untitled.png](/assets/img/post/Kubeproxy%20vs%20CNI/1.png)


[그림 출처: [https://docs.tigera.io/calico/latest/reference/architecture/overview](https://docs.tigera.io/calico/latest/reference/architecture/overview)]


이제 위의 그림을 설명하면, 우선 BGP가 서로와 정보를 교환하며 라우팅 정보를 교환한다. 만약 10.0.1.10 파드(노드1)가 10.0.2.11 파드(노드2)으로 트래픽을 전송하는 예시를 따라가보자.

1. **(파드1)**기본 게이트웨이에 맞게 local-link(로컬 네트워크)로 ARP를 요청한다.
2. **(calico)** Proxy-ARP로, 10.0.2.11에 대한 ARP 요청을 자신의 MAC 주소로 응답한다.
3. **(파드1)** 반환받은 MAC 주소(**calico**)으로 트래픽을 전송한다.
4. **(calico)** 자신에게 온 트래픽을 커널 공간에서 netfilter(iptables라면)으로 라우팅한다.
	1. 라우팅 옵션이 IPIP라면, 예를 들어 **SRC 192.168.1.10(노드1) DST 192.168.1.11(노드2)**로 캡슐화된다.
5. **노드2**는 자신에게 온 트래픽을 받는다.
6. 커널에서 **calico**으로 트래픽을 라우팅한다.
7. (**calico**)에서 캡슐화된 헤더를 제거하고 패킷을 파드2(10.0.2.11)로 트래픽을 전송한다.

이렇듯 CNI pod to pod의 네트워크를 구현하며, 터널링기능을 통해 우리는 다른 노드에 존재하는 파드와도 통신할 수 있다. EKS의 경우 노드와 파드의 네트워크 대역이 같으니, 터널링이 필요없다. 다만 네트워크 대역이 같으므로 파드의 개수의 한계가 있다.

