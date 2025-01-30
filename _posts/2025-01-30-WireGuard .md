---
layout: post
title: WireGuard 살펴보기 2편
date: 2025-01-30 09:00 +0900 
description: WireGuard의 방어시스템과 실제 실습을 통한 확인
category: [Security, VPN] 
tags: [K8s, Security, Zero Trust, VPN, CNI, 대칭키, 암호화, 복호화] 
pin: false
math: true
mermaid: true
---
아래의 작성된 글과 그림은 [https://www.wireguard.com/papers/wireguard.pdf](https://www.wireguard.com/papers/wireguard.pdf)에서 확인하실 수 있습니다.



<!--more-->


### 들어가며


지난 1편에서는 WireGuard가 무엇인지 어떤 방식으로 암호화를 지원하는지 알아봤다. 2편에서는 WireGuard에 존재하는 여러 방어시스템을 알아보고 실제 WireGuard 환경을 구성해 1편에서 봤던 내용을 직접 확인한다.


## 방어 시스템


WireGuard는 다양한 보안 위협으로부터 안전하게 통신을 유지하기 위해 여러 방어 시스템을 도입하고 있다. 주요 방어 대상은 DoS(Denial of Service) 공격, 재전송 공격 등이있다. 우선 Dos 공격을 방어하는 쿠키 시스템부터 알아보자. 


### 쿠키  시스템


핸드셰이크 요청을 반복해서 보내는 공격자가 있다고 가정하자. 서버는 이 요청을 응답하는 과정에서 신뢰성 확인을 위해 Curve25519 계산이 필요하다. 이는 CPU 연산이 많이 필요로 하며 결국 서버가 CPU 리소스를 소진하여 비정상적인 상태를 가질 수 있다. WireGuard에서는 이를 방지하고자 쿠키 시스템을 사용한다. 


![5.4.1 Protocol Overview](/assets/img/post/WireGuard2/1.png)


서버의 **부하 발생시** 핸드쉐이크를 진행하지 않고 **송신자에게 쿠키를 던져준다.** 이 쿠키는 송신자의 신원을 식별할 수 있으며, 공격자는 쿠키없이 다시 요청을 보내도 서버에서 무시하기에 Dos 공격을 방어할 수 있다.  자세한 방식을 살펴보면 아래와 같다.


아래의 MAC은 MAC 주소를 의미하는 것이 아닌 Message Authentication Code를 의미한다.


**(1)** 수신자는 부하 발생시 핸드셰이크를 하지 않고 쿠키 응답 메시지를 보낸다

- 쿠키($L$) = `MAC(2분마다 변경되는 랜덤한 값, 송신자의 IP 주소)`
- 쿠키 자체가 IP와 연관있어, IP 속도 제한 알고리즘도 사용할 수 있다.

**(2)** 송신자는 이 쿠키를 `mac2`에 포함하여 핸드셰이크 시작 패킷을 다시 전송한다.

- $\text{msg.mac1} := \text{Mac} \Big( \text{Hash} (\text{Label-Mac1} \parallel S_{m'}^{\text{pub}}), \text{msg}_{\alpha} \Big)$
- $\text{msg.mac2} := \text{Mac}(L_m, \text{msg}_{\beta})$, 여기서 $L_m$은 `m seconds` 이전에 받은 쿠키 메시지를 의미한다.

![5.4.2 First Message: Initiator to Responder](/assets/img/post/WireGuard2/2.png)


**(3)** 수신자는 올바른 MAC을 가진 송신자의 패킷만 수신한다. mac2가 없거나, 비정상적인 경우 이를 무시한다.


방식을 살펴보면 쿠키 시스템을 통해 Dos 공격으로부터 방어가 가능해보인다. 다음과 같은 특수 상황도 고려 해보자. 만약 최악의 시나리오로 공격자가 특정 서버의 공개키를 탈취하는 것에 성공했다면, `msg.mac1` MAC 코드를 감청하여 지속적으로 공격할 수 있을까?


이는 현실적으로 어렵다. 쿠키는 2분의 시간제한이 걸려있어 지속적으로 사용이 어렵고 쿠키는 송신자의 IP 기반으로 만들어지므로 공격자는 특정 IP로 고정할 수 밖에 없다. 해당 IP로 공격이 들어오는 것을 식별하면 IP 속도 제한 알고리즘을 사용할 수 있다.


### 재전송 공격


재전송 공격이란, 공격자가 정상적인 송신자의 메시지를 캡처하고 이를 재전송하여 수신자를 속이는 공격방식을 의미한다. 이러한 공격은 인증된 통신을 방해하고, 데이터의 무결성을 해칠 수 있다.


WireGuard에서는 counter 혹은 타임스탬프 방식으로 패킷의 순서를 표시하여, 재전송 공격을 방지한다.


#### 핸드셰이크 메시지의 재전송 공격 방지


TAI64N 타임스탬프를 포함하는데, 이는 나노초 기준으로 타임스탬프를 표시하는 것이다. 


```go
func stamp(t time.Time) Timestamp {
	var tai64n Timestamp
	secs := base + uint64(t.Unix())
	nano := uint32(t.Nanosecond()) &^ whitenerMask
	binary.BigEndian.PutUint64(tai64n[:], secs)
	binary.BigEndian.PutUint32(tai64n[8:], nano)
	return tai64n
}
```


응답자는 타임스탬프를 통해 **최근에 수신한 타임스탬프보다 최신**인지 확인한다. 만약 최신 패킷이 아니라면 그대로 무시한다. 한번 연결이 끊어졌다가 재연결된 경우에는 과거보다 더 큰 타임스탬프를 두는 것으로 해결한다.


#### 메시지 카운터를 통한 재전송 공격 방지


실제 암호화가 이뤄진 후에는 counter를 통해 메시지의 순서를 알린다. 덕분에 WireGuard는 UDP 기반임에도 데이터의 순서를 알 수 있다. 또한 이 순서는 재전송 공격 방지에도 사용된다.


![5.4.6 Subsequent Messages: Transport Data Messages](/assets/img/post/WireGuard2/3.png)


WireGuard에서 사용하는 2가지 방어시스템에 대해 알아봤다. 이제 실습을 들어가기 앞서 실제 통신이 어떤식으로 이뤄지는 지 구체적인 예시로 살펴보자.


## Packet Flow


이제 WireGuard에서 패킷이 송신되고 수신되는 과정을 단계별로 살펴보자, 구성은 아래의 그림과 같다고 가정한다.


![출처: https://www.wireguard.com/papers/wireguard.pdf](/assets/img/post/WireGuard2/4.png)


### 송신 과정

- 로컬에서 생성된 패킷이 wg0 interface로 전달된다.
- 패킷의 목적지 주소 IP를 확인하고 Table에서 매칭되는 Peer를 확인한다.
	- 만약, 매칭되는 게 없으면 버려지고 발신자에게 표준  ICMP “no route to host” 패킷을 받는다.
- counter 헤더를 추가한다. (재전송 방지)
- Peer와 연관된 대칭키를 기반으로 ChaCha20Poly1305 알고리즘을 사용하여 패킷을 암호화한다.
- 이제 테이블에서 확인한 **엔드포인트로 전송**한다.
	- 엔드포인트는 “미리 구성되어 있거나”, 혹은 “가장 최근에 올바르게 인증된 수신 패킷의 외부 소스 IP 헤더 필드에서 학습된다”
	- 만약, 엔드포인트가 없는 경우 패킷은 버려지고 ICMP 메시지가 전송되며, `-EHOSTUNREACH`가 사용자 공간으로 반환된다.

### 수신 과정

- 암호화된 패킷을 UDP로 수신한다.
- Receiver Index를 통해 어떤 Peer와 연관되어있는지 파악하고, 메시지 카운터의 유효성을 확인하며, 보안 세션키를 통해 해독을 시도한다. 만약 피어를 결정할 수 없거나 인증이 실패되면 패킷을 버린다.
- 인증된 패킷이므로 외부 헤더에 있는 IP를 통해 **엔드포인트 업데이트**한다.
- 패킷 페이로드(데이터 영역)을 해독하여 일반 텍스트 패킷을 갖는다. 이것이 IP 패킷이 아니라면 버리고, IP 패킷이 맞으면 Source IP주소를 확인해 Public key와 일치하는 지 확인한다.
- 암호화된 패킷을 wg0 인터페이스 수신 큐에 추가한다.

## 실습


이제 직접 실습을 통해 통신되는 과정을 살펴보자. 실제 서버역할을 하는 컨테이너2개를 docker 기반으로 띄어서 테스트를 진행한다. 이미지는 linuxserver에서 지원해주는 [wireguard 이미지](https://github.com/linuxserver/docker-wireguard)를 사용했다.


### 환경 구성


아래는 `docker-compose.yml` 파일이다. 간단하게 핵심 내용을 정리하면 다음과 같다.

- **Name**: wg_peer1, wg_peer2
- **Private IP**: `10.13.13.1`, `10.13.14.1`
- **port**: peer1(51820), peer2(51821)

```yaml
version: "3"
services:
  peer1:
    image: linuxserver/wireguard
    container_name: wg_peer1
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Seoul
      - SERVERURL=peer1
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0
    volumes:
      - ./config/peer1:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    networks:
      testing_net:
        ipv4_address: 172.20.0.2

  peer2:
    image: linuxserver/wireguard
    container_name: wg_peer2
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Seoul
      - SERVERURL=peer2
      - SERVERPORT=51821
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.14.0
    volumes:
      - ./config/peer2:/config
      - /lib/modules:/lib/modules
    ports:
      - 51821:51821/udp
    networks:
      testing_net:
        ipv4_address: 172.20.0.3

networks:
  testing_net:
    ipam:
      driver: default
      config:
        - subnet: 172.20.0.0/16
```


아래의 명령어를 통해 컨테이너를 생성하고, tcpdump 패키지를 설치한다.


```bash
docker compose up -d
docker exec -it wg_peer1 sh -c "apk update && apk add tcpdump"
docker exec -it wg_peer2 sh -c "apk update && apk add tcpdump"
```


#### wg0 설정

- **[peer1]** `/wireguard/config/peer1/wg_confs/wg0.conf` 파일 수정

```yaml
[Interface]
Address = 10.13.13.1/24
ListenPort = 51820
PrivateKey = {peer1의 config/server/privatekey-server 값}

[Peer]
PublicKey = {peer2의 config/server/publickey-server 값}
PresharedKey = 5FlR09bc0DX5DUicKQ4DA2J+5PPtYpPXg7RDOcWxskU=
AllowedIPs = 10.13.14.1/24
Endpoint = 172.20.0.3:51821 # peer2 컨테이너의 IP 및 포트
```

- **[peer2]** `/wireguard/config/peer2/wg_confs/wg0.conf` 파일 수정

```yaml
[Interface]
Address = 10.13.14.1/24
ListenPort = 51821
PrivateKey = {peer2의 config/server/publickey-server 값}

[Peer]
PublicKey = {peer1의 config/server/privatekey-server 값}
PresharedKey = 5FlR09bc0DX5DUicKQ4DA2J+5PPtYpPXg7RDOcWxskU=
AllowedIPs = 10.13.13.1/24
Endpoint = 172.20.0.2:51820 # peer1 컨테이너의 IP 및 포트
```



설정을 진행하고, 각 컨테이너에서 WireGuard 설정 적용을 wg0 인터페이스를 다시 로딩한다.


```bash
wg-quick down wg0
wg-quick up wg0
```


#### 통신 테스트


wireguard 연결을 테스트해보기 위해 wg_peer1에 접속한다. 이후 peer2의 WireGaurd private ip, `10.13.14.1`으로 ping을 보낸다. 설정이 잘된 경우 아래와 같이 정상적으로 통신이 이뤄진다.


```bash
root@06a0ad82e632:/# ping 10.13.14.1
PING 10.13.14.1 (10.13.14.1) 56(84) bytes of data.
64 bytes from 10.13.14.1: icmp_seq=1 ttl=64 time=3.52 ms
64 bytes from 10.13.14.1: icmp_seq=2 ttl=64 time=0.419 ms
64 bytes from 10.13.14.1: icmp_seq=3 ttl=64 time=3.89 ms
64 bytes from 10.13.14.1: icmp_seq=4 ttl=64 time=0.893 ms
```


### Wireshake


#### 상황


wg_peer1에서는 아래와 같이 eth0과 wg0 인터페이스 덤프를 진행한다.


```bash
tcpdump -i eth0 udp port 51820 -w /config/peer1_eth0.pcap
tcpdump -i wg0 -w /config/peer1_wg0.pcap
```


wg_peer2에서는 아래와 같이 wg_peer1으로 ping 통신을 진행한다.


```bash
ping 10.13.13.1
```


#### 핸드셰이크


![image.png](/assets/img/post/WireGuard2/5.png)


**Handshake Initiation**을 살펴보자 아래와 같이 임시키와 자신의 Public 키를 기반으로 만들어진 Static 키와 timestamp를 함께 확인할 수 있다. 


![image.png](/assets/img/post/WireGuard2/6.png)


**Handshake Response**를 살펴보면, 아래와 같이 자신의 임시키를 보내고 현재는 부하가 존재하는 상태가 아니기에 쿠키 reply를 하지 않는다.


![image.png](/assets/img/post/WireGuard2/7.png)


위 두 과정을 통해 신원 확인 및 세션키 생성이 완료되고 세션키를 통해 데이터를 암호화한다. 그렇기에 우리는 내부 데이터를 들여다볼 수 없다. 


![image.png](/assets/img/post/WireGuard2/8.png)


#### wg0 인터페이스 dump 확인


wg0 인터페이스로 확인하면, 복호화가 완료된 데이터이므로 내부 데이터를 확인할 수 있다. 우리는 ping 통신을 진행했는데, 그 내용을 확인할 수 있으며 어떤 wg private IP로부터 왔는지도 알 수 있다.


![image.png](/assets/img/post/WireGuard2/9.png)


### 마치며


이번 포스팅에서는 WireGuard의 다양한 방어 시스템에 대해 살펴보고, Docker를 기반으로 진행한 실습을 통해 실제 통신 과정을 확인했다. 특히, 쿠키 매커니즘과 재전송 공격 방지 방법을 통해 WireGuard가 어떻게 높은 보안성을 유지하는지 이해할 수 있었다. 또, 실습 환경을 구성하여 실제 WireGuard의 패킷을 볼 수 있었다. WireGuard 백서를 들여다보면서 WireGuard의 보안 메커니즘을 이해할 수 있었고 부족했던 보안 지식을 채울 수 있었던 것 같다.

