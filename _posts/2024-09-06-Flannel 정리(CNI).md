---
layout: post
title: Flannel 정리 (CNI)
date: 2024-09-06 09:00 +0900 
description: Flannel 정리 (CNI)
category: [Kubernetes, Network] 
tags: [KANS, CloudNet, Kubernetes, Network, KANS#2, Flannel, CNI] 
pin: false
math: true
mermaid: true
---
KANS 스터디 2주차 CNI, Flannel 알아보기
<!--more-->


> 💡 **KANS 스터디**  
> CloudNet에서 주관하는 KANS(**K**ubernetes **A**dvanced **N**etworking **S**tudy)으로 쿠버네티스 네트워킹 스터디입니다. 아래의 글은 스터디의 내용을 기반으로 작성했습니다.  
>   
> 스터디에 관심이 있으신 분은 [CloudNet Blog](/c9dfa44a27ff431dafdd2edacc8a1863)를 참고해주세요.


### 들어가며


Kubernetes CNI에 대해 학습하는 시간이다. Flanne을 시작으로 Calico, Cilium에 대해서도 추후 배울 예정이다.


### CNI 명세서 정리


[공식 문서](https://github.com/containernetworking/cni/blob/main/SPEC.md) 위의 문서를 정리해주셨다.

- 명세에는 컨테이너가 리눅스 네트워크 namespace안에 있다고 정의합니다.
도커와 같은 컨테이너 runtime은 매 컨테이너 실행 시, 새로운 namespace를 만들기에 네트워크 namespace에 대해 잘 알고 있어야 합니다.
- CNI의 네트워크 정의서는 JSON 형식으로 정의됩니다.
- 네트워크 정의서는 STDIN을 통해 스트림으로 CNI plugin에 전달되어야 합니다. 네트워크 설정을 위한 파일이 따로 특정 위치에 저장되어 참조되지 않아야 합니다.
- 다른 매개변수들은 환경변수로 plugin에 전달되어야 합니다.
- CNI plugin은 실행파일(executable)로 구현되어야 합니다.
- CNI plugin은 컨테이너 네트워크 연결에 책임을 가지고 있습니다. (컨테이너가 네트워크에 연결되기 위한 모든 작업에 책임을 가집니다.)
도커에서는 컨테이너의 네트워크 namespace를 호스트에 연결 시키는 것까지 포함됩니다.
- CNI plugin은 IPAM(IP 할당관리)에 책임을 가지고 있습니다. 이것은 IP주소 할당 뿐만 아니라 적절한 라우팅 정보를 입력하는 것까지 포함됩니다.

요약하면 CNI는 IP 할당 + 컨테이너 네트워크 연결(파드 내부, 같은 노드 내의 다른 파드, 다른 노드에 위치한 파드 포함)에 대한 책임을 가진다.


파드 생성부터 IP할당과 관련된 자세한 다이어그램은 아래와 같다.


![image.png](/assets/img/post/Flannel%20정리(CNI)/1.png)


출처: [https://ronaknathani.com/blog/2020/08/how-a-kubernetes-pod-gets-an-ip-address/](https://ronaknathani.com/blog/2020/08/how-a-kubernetes-pod-gets-an-ip-address/)


## Flannel


Flannel은 CNI 구현체 중 하나로, 3개의 네트워크 Fabric을 간단하게 구현할 수 있다.


**특징**

- 네트워크 구성, 라우팅 테이블 정보를 etcd를 통해 다른 노드들과 동기화
- 같은 노드 내의 파드 통신을 브릿지 형태로 제공
- VXLAN, host-gw, UDP(비권장) 네트워크 지원

네트워크 지원방식을 먼저 확인해보면

- VXLAN: LAN 계층을 가상화하여, 터널링을 제공한다.
- UDP: UDP 네트워크 오버레이 기법으로, 오래된 커널버전에서 동작한다. (8285 port)
- host-gw: 오버레이없이, 각 노드의 파드 네트워크 대역을 라우팅 테이블에 업데이트한다. 오버레이가 없어 빠르나 노드의 개수가 커질수록 테이블 크기와 성능저하 우려가 있다.

Flannel의 통신 방식을 사용하면 아래와 같다. 같은 노드에 위치한 파드는 cni0이라는 브릿지를 통해 연결하고, 외부와는 flannel 인터페이스를 통해 통신한다.


![image.png](/assets/img/post/Flannel%20정리(CNI)/2.png)


Flannel은 네트워크 구성 정보, Routing Table 정보를 **etcd를 통해서 받아 온다.**

- etcd에 적당한 부하가 생기겠지만 구조가 심플해서 성능은 괜찮다. 관련 [벤치마크 자료](https://itnext.io/benchmark-results-of-kubernetes-network-plugins-cni-over-10gbit-s-network-36475925a560)
- **하지만, networkpolicy를 지원하지 않는 단점이 있다.**

![image.png](/assets/img/post/Flannel%20정리(CNI)/3.png)


출처: [https://docs.openshift.com/container-platform/3.4/architecture/additional_concepts/flannel.html](https://docs.openshift.com/container-platform/3.4/architecture/additional_concepts/flannel.html)


### 실습 환경 구성


kind 배포 파일이다. flannel 설치를 위해 기본 CNI 생성을 disables했다.


```bash
cat <<EOF> kind-cni.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  labels:
    mynode: control-plane
  extraPortMappings:
  - containerPort: 30000
    hostPort: 30000
  - containerPort: 30001
    hostPort: 30001
  - containerPort: 30002
    hostPort: 30002
    
    ## 아래는 추후, 그라파나 및 프로메테우스를 위해 사전 설정(사전으로 안하면, static pod.yaml 값 변경하고 restart해야함)
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    controllerManager:
      extraArgs:
        bind-address: 0.0.0.0
    etcd:
      local:
        extraArgs:
          listen-metrics-urls: http://0.0.0.0:2381
    scheduler:
      extraArgs:
        bind-address: 0.0.0.0
  - |
    kind: KubeProxyConfiguration
    metricsBindAddress: 0.0.0.0
- role: worker
  labels:
    mynode: worker
- role: worker
  labels:
    mynode: worker2
  networking:
  disableDefaultCNI: true
EOF
```

- 클러스터 생성

```bash
kind create cluster --config kind-cni.yaml --name myk8s --image kindest/node:v1.30.4
```

- 필요 도구 설치

```bash
docker exec -it myk8s-control-plane sh -c 'apt update && apt install tree jq psmisc lsof wget bridge-utils tcpdump iputils-ping htop git nano -y'
docker exec -it myk8s-worker  sh -c 'apt update && apt install tree jq psmisc lsof wget bridge-utils tcpdump iputils-ping -y'
docker exec -it myk8s-worker2 sh -c 'apt update && apt install tree jq psmisc lsof wget bridge-utils tcpdump iputils-ping -y'
```

- flannel CNI 생성

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```


설치를 진행해도 호스트 IP를 가지는 파드외에 파드의 IP 할당이 안되는 것을 알 수 있다. 이는 브릿지 플러그인없어서 그렇다. 아래의 명령어를 통해 브릿지 파일을 다운받는다.


```bash
curl -O https://raw.githubusercontent.com/han-0315/han-0315.github.io/main/assets/practice/bridge
```

- 해당 파일을 각 node container에 옮겨주면, flannel이 정상적으로 지원한다.

```bash
docker cp bridge myk8s-control-plane:/opt/cni/bin/bridge
docker cp bridge myk8s-worker:/opt/cni/bin/bridge
docker cp bridge myk8s-worker2:/opt/cni/bin/bridge
_docker exec -it myk8s-control-plane  chmod 755 /opt/cni/bin/bridge
docker exec -it myk8s-worker         chmod 755 /opt/cni/bin/bridge
docker exec -it myk8s-worker2        chmod 755 /opt/cni/bin/bridge
```

- 파드 IP 할당 유무 확인

![image.png](/assets/img/post/Flannel%20정리(CNI)/4.png)

- 각 노드의 Flannel 정보 확인
	- 각 노드마다 파드 네트워크 대역이 `10.244.0.1`, `10.244.1.1`, `10.244.2.1`로 쪼개진 것을 확인할 수 있다.

```bash
for i in myk8s-control-plane myk8s-worker myk8s-worker2; do echo ">> node $i <<"; docker exec -it $i
 cat /run/flannel/subnet.env ; echo; done

>> node myk8s-control-plane <<
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=65485
FLANNEL_IPMASQ=true

>> node myk8s-worker <<
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.2.1/24
FLANNEL_MTU=65485
FLANNEL_IPMASQ=true

>> node myk8s-worker2 <<
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.1.1/24
FLANNEL_MTU=65485
FLANNEL_IPMASQ=true
```


### 파드 배포(라우팅 정보 확인)


```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    app: pod
spec:
  nodeSelector:
    kubernetes.io/hostname: myk8s-worker
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-2
  labels:
    app: pod
spec:
  nodeSelector:
    kubernetes.io/hostname: myk8s-worker2
  containers:
  - name: netshoot-pod
    image: nicolaka/netshoot
    command: ["tail"]
    args: ["-f", "/dev/null"]
  terminationGracePeriodSeconds: 0
EOF
```

- 파드 배포 전 상태

![image.png](/assets/img/post/Flannel%20정리(CNI)/5.png)

- 배포 후

![image.png](/assets/img/post/Flannel%20정리(CNI)/6.png)


해당 노드에 파드가 하나 생겨서, 인터페이스가 생기고, cni0과도 연결된 모습


```bash
k get pods -o wide
NAME    READY   STATUS             RESTARTS      AGE   IP           NODE            NOMINATED NODE   READINESS GATES
pod-1   1/1     Running            0             41m   10.244.2.3   myk8s-worker    <none>           <none>
pod-2   1/1     Running            0             41m   10.244.1.6   myk8s-worker2   <none>           <none>
```


파드의 라우팅 테이블을 확인해보면, 같은 노드는 위치한 파드 대역은 10.244.1.1로 보낸다.


```bash
route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.244.1.1      0.0.0.0         UG    0      0        0 eth0
10.244.0.0      10.244.1.1      255.255.0.0     UG    0      0        0 eth0
10.244.1.0      0.0.0.0         255.255.255.0   U     0      0        0 eth0
```


worker 노드의 ip 라우팅 테이블이다.


```bash
ip route
default via 172.18.0.1 dev eth0
10.244.0.0/24 via 10.244.0.0 dev flannel.1 onlink
10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1
10.244.2.0/24 via 10.244.2.0 dev flannel.1 onlink
172.18.0.0/16 dev eth0 proto kernel scope link src 172.18.0.2
```


2번째 규칙의 의미를 자세하게 살펴보면 아래와 같다.

- `10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1`
	- 10.244.1.0/24 대역으로 향하는 패킷은 cni0으로 전달된다.
- `proto kernel scope link src 10.244.1.1`
	- 커널에 의해 생성된 것이며, cni0과 연결된 link에만 적용된다.
	- 이 인터페이스의 IP는 10.244.1.1이다.

나머지 규칙도 요약하면 아래와 같다.

1. 아래의 경로에 해당되지 않은 트래픽은 eth0(172.18.0.1-gw)으로 전달
2. `10.244.0.0/24`, `10.244.2.0/24` 대역(다른 노드의 파드 대역) flannel.1 인터페이스로 전달
3. `172.18.0.0/16` 노드 대역은 eth0(172.18.0.2-자신) 인터페이스로 전달

### 실습 진행(통신 확인)


*추가로 `k run test --image nginx --port 80` 명령어를 통해 파드를 하나 더 생성한다. (같은 노드 내의 파드 통신 테스트를 위해)


아래의 실습을 통해 통신 테스트를 진행한다.


**[터미널1]** 같은 노드내의 위치한 파드, 다른 노드에 위치한 파드, 구글 DNS에 핑을 보낸다.


![image.png](/assets/img/post/Flannel%20정리(CNI)/7.png)


**[터미널2]** worker node에 접속하여 트래픽을 캡쳐한다.


```bash
docker exec -it myk8s-worker  bash
```


![image.png](/assets/img/post/Flannel%20정리(CNI)/8.png)


> 💡 tcpdump를 통해 아래의 정보를 알 수 있다.  
> - **같은 노드 내의 파드 통신은 cni0 인터페이스를 통해 전달**  
>   
> - **다른 노드 내의 파드 통신은 flannel.1 인터페이스를 통해 전달**  
>   
> - **외부와의 통신은 노드의 eth0 인터페이스를 통해 전달**


#### wireshark


다른 노드 내의 파드 통신의 캡슐화를 확인해보기 위해 이제 캡쳐한 패킷을 한번 wireshark를 통해 까본다.


wireshark UDP 기본 옵션이 4789여서 flannel(8479)와 달라, 캡슐화로 감춰진 내용이 보이지 않는다.  아래의 화면을 통해 VXLAN 프로토콜 포트를 8479로 수정하면 확인할 수 있다.


![vxlan-flannel.gif](/assets/img/post/Flannel%20정리(CNI)/9.gif)


위에서 생성한 vxlan.pcap 패킷 파일을 열어본다. 아래와 같이 10.244.2.3(pod-1)에서 10.244.1.6(pod-2)로 보내는 패킷에 노드의 IP이 추가되어 감싸진 것을 확인할 수 있다.


![image.png](/assets/img/post/Flannel%20정리(CNI)/10.png)


### 정리

1. Flannel은 VXLAN 프로토콜을 통해 터널링을 진행한다.
2. 같은 노드 내의 파드 통신은 cni0 브릿지를 통해 진행한다.
3. 다른 노드 내의 파드 통신은 flannel.1 인터페이스에서 VXLAN 터널링으로 진행한다.
4. 외부와의 통신은 노드의 노드의 eth0을 통해 진행된다. (NAT 변환 진행)
5. Flannel은 라우팅 정보를 etcd를 통해 얻는다.
6. 다만, network policy를 지원하지 않는다는 단점이 있다.
