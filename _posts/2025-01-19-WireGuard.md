---
layout: post
title: WireGuard 살펴보기 1편
date: 2025-01-19 09:00 +0900 
description: WireGuard란 무엇이고, 어떻게 동작할까?
category: [Security, VPN] 
tags: [K8s, Security, Zero Trust, VPN, CNI, 대칭키, 암호화, 복호화] 
pin: false
math: true
mermaid: true
---


WireGuard란 무엇이고, 어떻게 동작할까?
<!--more-->


아래의 작성된 글과 그림은 [https://www.wireguard.com/papers/wireguard.pdf](https://www.wireguard.com/papers/wireguard.pdf)에서 확인하실 수 있습니다.


### 들어가며


최근 Flannel 코드 분석 스터디를 진행하며 WireGuard 부분을 맡았다. WireGuard에 대한 정보를 찾던 중 백서를 홈페이지에서 쉽게 찾을 수 있었다. 최근에 **Zero Trust**와 같은 보안 모델도 대두되고 전반적인 보안의 중요성이 강조되는 분위기다. 나는 이번 기회에 보안 백서도 읽어보면 좋지 않을까? 라는 마음으로 WireGuard를 정리하게 되었다. 


돌아가서 왜 CNI에서 WireGuard와 같은 패킷 암호화를 지원할까? 쿠버네티스 환경에서 VPC와 같은 프라이빗 네트워크가 아닌 경우 외부 인터넷을 거치는 통신이 발생한다. 이러한 상황 혹은 필요에 의해 암호화를 지원하는 경우가 생긴다. WireGuard는 높은 보안 수준과 성능 덕에 강력한 보안 솔루션으로 조명받고 있다. Flannel, Cilium, Calico 등 여러 CNI에서 암호화 옵션으로 WireGuard를 지원한다.


![<8> Performance](/assets/img/post/WireGuard/1.png)


### WireGuard란


우리가 크롬 브라우저를 통해 naver.com에 접속할 때, HTTPS 통신을 한다. 덕분에 외부의 누군가가 우리의 패킷을 들여다봐도 내용을 해석할 수 없다. WireGuard도 목적이 같다. 다른 대상(peer)과 통신할 때 패킷을 암호화하여 외부로부터 데이터를 보호하는 솔루션이다. 


조금 더 자세히 살펴보면, WireGuard는 오픈소스 VPN 프로젝트로 Layer3 수준에서 동작하며 Open VPN과 IPSec을 대체한다. `wg0`와 같은 가상의 인터페이스를 통해 처리하기에 기존 네트워크 스택과 통합된다. Linux 5.6 Kernel부터 기본 패키지로 포함되어 접근성이 좋다.


WireGuard는 어떻게 데이터를 암호화할까? 암호화 이전에 사전 작업으로 SSH와 같이 사전에 비대칭키를 설정해줘야한다.


### 사전 설정


SSH와 같이 사전의 키 생성 및 교환이 필요한데, 방식이 agnostic하다. (agnostic은 여기서 특정 방식에 얽매이지 않는다는 뜻이다.) 이런 접근 방식은 OpenSSH에서 영감을 받았다고 백서에 설명되어있다. 키를 교환하는 방식을 정해두지 않았고, 이를 상관쓰지 않는다. 이것은 **사용자의 몫**으로 바라보는 시각이다.
즉 WireGuard를 사용하기 위해서 아래와 같이 자신의 Public key, Private key를 설정하고 상대방의 Public Key를 사용자가 모두 등록하는 작업이 필요하다.


![<2> Cryptokey Routing](/assets/img/post/WireGuard/2.png)


### 키 교환


WireGuard는 사전에 교환된 키를 기반으로 암호화를 진행한다. 하지만, 해당 키가 노출되어도 쉽게 해독할 수 없다. 어떻게 암호화를 진행하기에 이것이 가능할까?


데이터 암호화에 사전에 교환된 키만 사용하는 것이 아니다. **임시 키도 사용**한다. Diffie–Hellman key 교환 방식을 통해 대칭 키를 생성한다. 대칭키는 아래의 핸드쉐이크 과정에서 공유된다.


![<5.4.1> Protocol Overview](/assets/img/post/WireGuard/3.png)


구체적으로 키를 교환하는 과정을 살펴보면 아래와 같은데 간단하게 “핸드쉐이크를 진행하며 생성한 **임시키**와 사전에 확보한 상대방 **Public key**를 통해 여러 번의 해싱 함수를 거치며 최종적으로는 대칭키를 만들어낸다.”라고 생각해도 좋다.


#### 핸드쉐이크


처음 통신을 시작하기 위해 초기자(송신자)가 핸드쉐이크를 시작하는 패킷 구조이다.


![<5.4.2> First Message: Initiator to Responder](/assets/img/post/WireGuard/4.png)


송신자는 자신의 임시키를 생성하고 임시 Public Key를 `msg.ephemeral`에 넣는다.


$
(E_i^{\text{priv}}, E_i^{\text{pub}}) := \text{DH-GENERATE}()$


$
\text{msg.ephemeral} := E_i^{\text{pub}}$


자신의 사전에 설정된 Public Key, Private key, 임시 Private Key 그리고 사전에 공유된 응답자의 Public Key를 기반으로 static과 timestamp를 만든다. 이 항목들로 **신원을 한번 더 검증**할 수 있다.


msg.static 필드와 msg.timestamp를 계산하는 과정을 자세하게 살펴보고 싶다면, 아래의 수식을 참고하면 된다. (백서 11p)


$
Hᵢ := \text{HASH}(Hᵢ \parallel S_r^{\text{pub}})$


$
Cᵢ := \text{KDF}_1(Cᵢ, E_i^{\text{pub}})$


$
Cᵢ, \kappa := \text{KDF}_2(Cᵢ, \text{DH}(E_i^{\text{priv}}, S_r^{\text{pub}}))$


$
\text{msg.static} := \text{AEAD}(\kappa, 0, S_i^{\text{pub}}, Hᵢ)$


$
Hᵢ := \text{HASH}(Hᵢ \parallel \text{msg.static})$


$
Cᵢ, \kappa := \text{KDF}_2(Cᵢ, \text{DH}(S_i^{\text{priv}}, S_r^{\text{pub}}))$


$
\text{msg.timestamp} := \text{AEAD}(\kappa, 0, \text{TIMESTAMP}(), Hᵢ)$


$
Hᵢ := \text{HASH}(Hᵢ \parallel \text{msg.timestamp})$


잠깐 내용을 정리해보자. 사전에 비대칭키를 사용자가 각 서버에 설정한다. 핸드쉐이크 과정에서 사전에 설정한 비대칭키를 확인하며 임시키를 생성하여 교환한다. 이 과정을 통해 만들어낸 세션키를 통해 암호화와 복호화를 진행한다. 암호화와 복호화는 어떤 방식으로 진행되는지 아래에서 알아본다.


### 암호화


핸드쉐이크를 통해 세션키를 만들고 이를 통해 암호화 복호화를 진행한다. 구체적인 방식을 살펴보면**ChaCha20Poly1305**를 통해 암호화 및 복호화가 진행되며, 이는 **XOR 연산**을 기반으로 한다. XOR 연산은 복호화에 용이하다. 
$
A \oplus B \oplus B = A$ 와 같이 XOR 연산을 두 번하면 원래 데이터의 값이 되어 암호화 복호화에 용이하다. 


![<5.4.6> Subsequent Messages: Transport Data Messages](/assets/img/post/WireGuard/5.png)


핸드쉐이크 이후 보내는 패킷 구조는 위와 같다. 


여기서 `msg.packet`은 ChaCha20Poly1305를 통해 암호화된 데이터를 의미한다. $I_m$과 Counter 필드에 대해 살펴보자.

- **Receiver Index(**$I_m$) 수신하는 피어가 해당 필드를 참고하여 적합한 **세션 키**를 찾게 도와주는 정보로, 이는 핸드셰이크에서 교환시 값이 결정된다.
- **Counter (Nonce):** 이 필드는 **패킷의 순서**를 알 수 있게 해준다. 각 메시지마다 증가하며, 암호화할 때도 사용된다. 수식으로는 $
\text{msg.counter} := N_m^{\text{send}}$이다.

**packet** 패킷은 다음과 같은 연산을 통해 암호화된다.


 $
\text{msg.packet} := \text{AEAD}(T_m^{\text{send}}, N_m^{\text{send}}, P, \epsilon)$  _여기서_ $T_m^{\text{send}}$_는 대칭키 중 송신자의 키를 의미한다._


### 암호화 방식과 프로토콜


위의 설명에서 확인된 암호화 방식말고 다른 방식을 쓰고 싶을 수도 있다. WireGuard에서는 불가능하다. WireGuard는 제한된 암호화 알고리즘과 프로토콜만을 사용하여 프로토콜 유연성을 제거했다. 유연성을 잃었지만 잦은 업데이트를 피했다. 만약 하나의 암호화 프로토콜에서 취약점이 발견된다면 모든 업데이트가 필요하기에 유지보수에 많은 비용이 든다.


지금까진 취약점이 없는 것으로 알려졌지만 반대로 해당 프로토콜에 의존하므로 취약점이 발견되면 모든 통신과 사용자가 한번에 노출될 수 있다는 단점이 있다. (이에 대한 대비도 하여, 언제든지 프로토콜을 교체할 수 있다고는 한다.)


### 마무리


이 포스팅에서는 WireGuard을 간단하게 살펴봤다. 간단하게 요약한다면, WireGuard는 오픈소스 VPN 프로젝트로 기존의 유사 모델(IPsec, OpenVPN)을 대체하기 위해 나왔다. 리눅스 커널에 내장되어있어 접근성이 좋으며 쿠버네티스에서 네트워킹 보안을 위해 쓰이기도 한다. 사전에 교환된 비대칭키와 핸드쉐이크 과정에서 생성한 임시키를 통해 세션키를 만들어내고 이를 통해 암호화를 진행한다. 


다음 편에서는 WireGuard에서 여러 공격을 막기 위해 도입한 구조를 알아보고, 실습을 진행하며 이론으로 살펴본 내용을 확인해볼 예정이다.

