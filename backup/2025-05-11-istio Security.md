---
layout: post
title: istio Security의 필요성과 SPIFFE
date: 2025-05-10 09:00 +0900 
description: istio 보안의 필요성과 SPIFFE에 대해 알아본다.
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#5, Security, SPIFFE] 
pin: false
math: true
mermaid: true
---
istio 보안의 필요성과 SPIFFE 살펴보기
<!--more-->


### 들어가며


이번 주차에서는 보안에 대해 알아본다. 쿠버네티스에서도 기본적으로 제공하는 보안 요소가 있지만 여기서는 istio 기반의 강력한 보안에 대해 살펴볼 예정이다. istio는 각 파드마다 proxy를 두고 있기에, 강력한 제어 및 보안 기능이 가능할 것으로 보인다. 


우선 애플리케이션(서비스)에서 기본적으로 가지고 있어야할 보안 요소에 대해 살펴보자


### istio security


istio [공식문서](https://istio.io/latest/docs/concepts/security/)에 나와있는 마이크로서비스가 기본적으로 갖춰야 할 보안 요소는 아래와 같다.

- **To defend against man-in-the-middle attacks, they need traffic encryption.**
- **To provide flexible service access control, they need mutual TLS and fine-grained access policies.**
- **To determine who did what at what time, they need auditing tools.**

간단히 요약하면, 중간자 공격으로부터 방어가 가능해야하며 mTLS와 적절한 감사도구를 지원할 수 있어야한다.


이는 Istio security 목적과 유사하다. 아래는 공식문서에 나와있는 istio security goal이다.

- **Security by default: no changes needed to application code and infrastructure**
- **Defense in depth: integrate with existing security systems to provide multiple layers of defense**
- **Zero-trust network: build security solutions on distrusted networks**

참고로 istio의 보안 아키텍처는 아래와 같다.


서비스 간 통신에서 **mTLS를 지원하고**, istiod(control plane)에서 인증서 발급 등 보안과 관련된 작업을 담당한다. 


![image.png](/assets/img/post/istio%20Security/1.png)


출처: [https://istio.io/latest/docs/concepts/security/#high-level-architecture](https://istio.io/latest/docs/concepts/security/#high-level-architecture)


## SPIFFE


**SPIFFE**는 **S**ecure **P**rduction **I**dentity **F**ramework **F**or **E**veryone의 약자로 dynamic and heterogeneous environments에서 사용하는 보안 프로토콜 오픈소스라고 한다.


요약하면, 여러 서비스에 ID를 기반으로 데이터를 암호화하고 서로를 식별하는 프로토콜이다. 전통적으로는 IP와 같은 정적인 값을 사용했으나 마이크로서비스나 클라우드 환경으로 들어서면서 IP 또한 동적으로 변할 가능성이 있기에 ID 기반으로 이러한 문제를 해결한다.


### 구성요소

- **Workload**: 특정 목적을 위해 배포된 단일 소프트웨어 단위(특정 서비스 파드)
- **SPIFFE ID**: `spiffe://<trust-domain>/<path>` 형태의 URI. 워크로드를 전역적으로 식별
	- trust-domain은 ID 발급자이며, path는 식별자이다.
	- ID는 X.509 인증서로 인코딩되며 istio에서는 controlplane이 ID 발급을 담당하고 있다.
- **SVID**(SPIFFE Verifiable Identity Document): X.509 또는 JWT 형식으로 SPIFFE ID를 담은 **짧은 수명의** 인증서

### SPIFFE Workload API


SPIFFE 사양에서 컨트롤 플레인 구성 요소를 나타내며, 워크로드가 자신의 ID를 정의하는 SVID 형식 디지털 인증서를 가져갈 수 있도록 엔드포인트를 노출한다.


API를 통해 인증서 발급이 가능하다. 인증서 서명 요청(CSR)에 인증 기관(CA) 개인 키로 서명함으로써 워크로드에 인증서 발급한다.


![image.png](/assets/img/post/istio%20Security/2.png)


그림에 대한 설명을 덧붙이면 아래와 같다.

1. **엔드포인트**는 워크로드의 **무결성을** 확인하고(즉, 워크로드 증명을 수행하고) **SPIFFIE ID가 인코딩된 CSR을 생성**한다.
2. 워크로드 엔드포인트는 **서명을 위해 워크로드 API에 CSR을 제출**한다.
3. **워크로드 API는 CSR을 서명하고 디지털 서명된 인증서로 응답**한다.
	- 이 인증서의 **SAN**의 URI 확장에는 SPIFFE ID가 있다.
	- 이 인증서는 워크로드 ID를 나타내는 **SVID**이다.

### in Istio


istio에서는 **워크로드 엔드포인트** = istio proxy(pilot agent)가 담당하고, **워크로드 API는** istiod(Istio CA)가 수행한다. 


![image.png](/assets/img/post/istio%20Security/3.png)


이렇게 발급받은 ID 인증 체계를 기반으로 mTLS 통신을 진행한다. 실습과 관련된 내용은 다음 편의 PeerAuthentication 실습 페이지에서 확인할 수 있다. 그 외로 istio는 인증/인가에 대한 설정을 모두 지원한다. 인증과정에서는 mTLS 통신이 아니면 거부할 수 있고, 클라이언트의 요청에 대한 인증을 JWT로 강제할 수 있다. 또, 인가에서는 각 서비스마다 요청 권한을 설정할 수 있다.


![image.png](/assets/img/post/istio%20Security/4.png)


출처: 스터디원(istio security)


### 마치며


이번 포스팅에서는 istio 인증과 관련된 주요 이론에 대해 살펴봤다. 기본적인 온프렘 구조에서는 보안이란 어려운 것 같다. 보안을 세밀하게 챙기다보면 성능 및 편의가 많이 무너지는 것 같다. (아닌 부분도 있겠지만), 최근 SKT 유심사태로 보안에 대해 자세히 한번 살펴볼 좋은 기회인 것 같다. 다음 포스팅에는 istio 트래픽 인증에 대해 자세히 살펴본다.

