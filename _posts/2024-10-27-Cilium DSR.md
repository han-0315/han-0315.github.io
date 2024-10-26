---
layout: post
title: Cilium DSR 알아보기
date: 2024-10-27 08:00 +0900 
description: Cilium DSR 알아보기
category: [Kubernetes, Network] 
tags: [KANS, CloudNet, Kubernetes, Network, KANS#8, DSR,  Cilium] 
pin: false
math: true
mermaid: true
---
Cilium DSR 알아보기
<!--more-->


> 💡 **KANS 스터디**  
> CloudNet에서 주관하는 KANS(**K**ubernetes **A**dvanced **N**etworking **S**tudy)으로 쿠버네티스 네트워킹 스터디입니다. 아래의 글은 스터디의 내용을 기반으로 작성했습니다.  
>   
> 스터디에 관심이 있으신 분은 [CloudNet Blog](/c9dfa44a27ff431dafdd2edacc8a1863)를 참고해주세요.


	


### DSR이란


DSR은 Direct Server Return의 약자로 LB를 통해 들어온 트래픽을 바로 서버로 리턴하는 구조이다. 


엔터프라이즈 급의 환경이라면 내부 서버를 L4와 연결하고 VIP를 통해 바인딩을 진행한다. 이때 inbound 트래픽과 outbound 트래픽을 모두 L4에서 연결 정보를 유지하며 처리해야 한다. 하지만 라우터에서 outbound 트래픽에 대해서 부하가 크다고 한다. DSR을 통해 라우터의 부하를 줄일 수 있다.(OutStream 통신을 위해 구성을 유지할 필요도 없으며, 서버에서 클라이언트로 나가는 트래픽이 모두 LB를 안거치니 좋다.) 


최근의 DSR은 L3DSR을 사용한다. L3DSR이란 Layer 3계층(IP)를 활용하여 DSR을 진행한다. IPIP 헤더를 쌓아서 터널링(tunl0)을 하거나 아래의 그림처럼 헤더의 내용을 변경한다. 아래 그림에선 VIP에 대한 목적지 IP만 변경한다. 


![image.png](/assets/img/post/Cilium%20DSR/1.png)


출처: [https://tech.kakao.com/posts/306](https://tech.kakao.com/posts/306)


클라이언트는 서버와 TCP 세션을 맺는데, DSR을 하면 IP 혹은 포트가 변경되지 않는지 의심이 들었다. 만약 세션을 유지하고 있는 스위치를 안거치고 나간다고 가정하면 TCP 세션에서 IP와 Port 정보가 달라지니 클라이언트가 해당 응답을 무시할 수 있다. 하지만, 내용을 잘 살펴보면 서버에서 클라이언트에게 응답을 보낼때도 외부와 연결된 스위치를 거쳐나가니 결국 클라이언트는 DSR의 과정을 모르며 정상 통신된다. 


### Cilium DSR


기본적으로 쿠버네티스에서 NodePort 혹은 LoadBalancer Service나 ExternalIP를 통해 외부에서 들어오는 트래픽이라면 다른 노드로 리다이렉션될 수 있다. 만약 kube-proxy를 사용한다면 기본적으로 SNAT이 한번되기에 백엔드 서버에서는 클라이언트의 IP를 알기 힘들다. 이런 점 때문에 `externalTrafficPolicy=Local` 옵션을 사용하기도 하나, 이는 모든 노드에 해당 백엔드 서버가 존재해야 하며 로드밸런싱이 고르지 않게 될 수 있다.


![image.png](/assets/img/post/Cilium%20DSR/2.png)


출처: [https://cilium.io/static/ae8ca98fe1a89b33ebd09f7dfc2d6eff/f9c4a/sock-1.png](https://cilium.io/static/ae8ca98fe1a89b33ebd09f7dfc2d6eff/f9c4a/sock-1.png)


Cilium 또한 외부 트래픽에 대해서는 기본적으로 SNAT 모드로 제공한다. 하지만 **로드밸런싱 모드를 DSR로 설정하여 Network 홉은 최소화**할 수 있다. 


아래의 그림과 같이 Cilium에서 DSR로 설정하면 백엔드 파드에서 클라이언트로 서비스의 IP와 PORT를 가지고 Return한다. (SRC가 서비스로 변경된다.)


![image.png](/assets/img/post/Cilium%20DSR/3.png)


출처: [https://cilium.io/static/b4488d749f6e74376e90dcff34c1ab6b/0aaa4/dsr-with.png](https://cilium.io/static/b4488d749f6e74376e90dcff34c1ab6b/0aaa4/dsr-with.png)


#### 주의사항 및 고려사항

- DSR 모드를 사용하려면 **Native-Routing 모드**를 사용해야 한다. 즉, tunnel 인터페이스를 `disabled` 해야한다.
- externalTrafficPolicy을 local로 설정을 완벽히 지원한다. 엔드포인트가 없는 노드로 전달하지 않고 해당 패킷을 drop 시킴으로 추가적인 네트워크 홉을 피할 수 있다.
- healthCheckNodePort 필드도 지원하여 Load Balancer 타입 서비스의 경우 외부 LB에서 Healcheck가 가능하다.
