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


	CloudNet에서 주관하는 KANS(**K**ubernetes **A**dvanced **N**etworking **S**tudy)으로 쿠버네티스 네트워킹 스터디입니다. 아래의 글은 스터디의 내용을 기반으로 작성했습니다.


	스터디에 관심이 있으신 분은 [CloudNet Blog](/c9dfa44a27ff431dafdd2edacc8a1863)를 참고해주세요.


### DSR이란


DSR은 Direct Server Return의 약자로 LB를 통해 들어온 트래픽을 바로 서버로 리턴하는 구조이다. 


엔터프라이즈 급의 환경이라면 내부 서버를 L4와 연결하고 VIP를 통해 바인딩을 진행한다. 이때 inbound 트래픽과 outbound 트래픽을 모두 L4에서 연결 정보를 유지하며 처리해야 한다. 하지만 라우터에서 outbound 트래픽에 대해서 부하가 크다고 한다. DSR을 통해 라우터의 부하를 줄일 수 있다.(OutStream 통신을 위해 구성을 유지할 필요도 없으며, 서버에서 클라이언트로 나가는 트래픽이 모두 LB를 안거치니 좋다.) 


최근의 DSR은 L3DSR을 사용한다. L3DSR이란 Layer 3계층(IP)를 활용하여 DSR을 진행한다. IPIP 헤더를 쌓아서 터널링(tunl0)을 하거나 아래의 그림처럼 헤더의 내용을 변경한다. 아래 그림에선 VIP에 대한 목적지 IP만 변경한다. 


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/1.png)


출처: [https://tech.kakao.com/posts/306](https://tech.kakao.com/posts/306)


클라이언트는 서버와 TCP 세션을 맺는데, DSR을 하면 IP 혹은 포트가 변경되지 않는지 의심이 들었다. 만약 세션을 유지하고 있는 스위치를 안거치고 나간다고 가정하면 TCP 세션에서 IP와 Port 정보가 달라지니 클라이언트가 해당 응답을 무시할 수 있다. 하지만, 내용을 잘 살펴보면 서버에서 클라이언트에게 응답을 보낼때도 외부와 연결된 스위치를 거쳐나가니 결국 클라이언트는 DSR의 과정을 모르며 정상 통신된다. 


### Cilium DSR


기본적으로 쿠버네티스에서 NodePort 혹은 LoadBalancer Service나 ExternalIP를 통해 외부에서 들어오는 트래픽이라면 다른 노드로 리다이렉션될 수 있다. 만약 kube-proxy를 사용한다면 기본적으로 SNAT이 한번되기에 백엔드 서버에서는 클라이언트의 IP를 알기 힘들다. 이런 점 때문에 `externalTrafficPolicy=Local` 옵션을 사용하기도 하나, 이는 모든 노드에 해당 백엔드 서버가 존재해야 하며 로드밸런싱이 고르지 않게 될 수 있다.


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/2.png)


출처: [https://cilium.io/static/ae8ca98fe1a89b33ebd09f7dfc2d6eff/f9c4a/sock-1.png](https://cilium.io/static/ae8ca98fe1a89b33ebd09f7dfc2d6eff/f9c4a/sock-1.png)


Cilium 또한 외부 트래픽에 대해서는 기본적으로 SNAT 모드로 제공한다. 하지만 **로드밸런싱 모드를 DSR로 설정하여 Network 홉은 최소화**할 수 있다. 


아래의 그림과 같이 Cilium에서 DSR로 설정하면 백엔드 파드에서 클라이언트로 서비스의 IP와 PORT를 가지고 Return한다. (SRC가 서비스로 변경된다.)


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/3.png)


출처: [https://cilium.io/static/b4488d749f6e74376e90dcff34c1ab6b/0aaa4/dsr-with.png](https://cilium.io/static/b4488d749f6e74376e90dcff34c1ab6b/0aaa4/dsr-with.png)


#### 주의사항 및 고려사항

- DSR 모드를 사용하려면 **Native-Routing 모드**를 사용해야 한다. 즉, tunnel 인터페이스를 `disabled` 해야한다.
- externalTrafficPolicy을 local로 설정을 완벽히 지원한다. 엔드포인트가 없는 노드로 전달하지 않고 해당 패킷을 drop 시킴으로 추가적인 네트워크 홉을 피할 수 있다.
- healthCheckNodePort 필드도 지원하여 Load Balancer 타입 서비스의 경우 외부 LB에서 Healcheck가 가능하다.

## 실습 진행


### cilium 설정 진행


DSR을 지원하도록 아래의 명령어를 통해 설정을 변경한다.


```bash
helm upgrade cilium cilium/cilium --namespace kube-system --reuse-values --set loadBalancer.mode=dsr
Release "cilium" has been upgraded. Happy Helming!
NAME: cilium
LAST DEPLOYED: Sat Nov  2 08:36:45 2024
NAMESPACE: kube-system
STATUS: deployed
REVISION: 3
TEST SUITE: None
NOTES:
You have successfully installed Cilium with Hubble Relay and Hubble UI.

Your release version is 1.16.3.

For any further help, visit https://docs.cilium.io/en/v1.16/gettinghelp
```


config 값이 변경된 것을 확인한다. 


```bash
cilium config view | grep bpf-lb-mode
bpf-lb-mode                                       dsr
```


```bash
c0 status --verbose | grep 'KubeProxyReplacement Details:' -A7
KubeProxyReplacement Details:
  Status:                 True
  Socket LB:              Enabled
  Socket LB Tracing:      Enabled
  Socket LB Coverage:     Full
  Devices:                ens5   192.168.10.10 fe80::ea:6bff:fe11:930f (Direct Routing)
  Mode:                   DSR
    DSR Dispatch Mode:    IP Option/Extension
```


### 실습 리소스 배포


```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: netpod
  labels:
    app: netpod
spec:
  nodeName: k8s-m
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
  name: webpod1
  labels:
    app: webpod
spec:
  nodeName: k8s-w1
  containers:
  - name: container
    image: traefik/whoami
  terminationGracePeriodSeconds: 0
---
apiVersion: v1
kind: Service
metadata:
  name: svc1
spec:
  ports:
    - name: svc1-webport
      port: 80
      targetPort: 80
  selector:
    app: webpod
  type: NodePort
EOF
```


배포 후 아래의 명령어를 통해 확인할 수 있다.


```bash
kubectl get po,svc -o wide
NAME          READY   STATUS    RESTARTS   AGE    IP             NODE     NOMINATED NODE   READINESS GATES
pod/webpod1   1/1     Running   0          4m5s   172.16.2.214   k8s-w1   <none>           <none>

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE    SELECTOR
service/kubernetes   ClusterIP   10.10.0.1       <none>        443/TCP        26m    <none>
service/svc1         NodePort    10.10.147.251   <none>        80:31615/TCP   4m5s   app=webpod
```


서비스 포트 확인


```bash
kubectl get svc svc1 -o jsonpath='{.spec.ports[0].nodePort}';echo
31615
```


### 테스트 진행


#### 시나리오


테스트 시나리오는 **test-pc**서버에서 **k8s-s**의 NodePort로 접근하여 **k8s-w1**에 존재하는 **webpod**와 통신한다.


실습을 진행하기전에 노드의 IP를 확인한다. test-pc의 ip는 192.168.10.200이다.


```bash
kubectl get nodes -o wide
NAME     STATUS   ROLES           AGE   VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
k8s-s    Ready    control-plane   37m   v1.30.6   192.168.10.10    <none>        Ubuntu 22.04.5 LTS   6.8.0-1015-aws   containerd://1.7.22
k8s-w1   Ready    <none>          37m   v1.30.6   192.168.10.101   <none>        Ubuntu 22.04.5 LTS   6.8.0-1015-aws   containerd://1.7.22
k8s-w2   Ready    <none>          37m   v1.30.6   192.168.10.102   <none>        Ubuntu 22.04.5 LTS   6.8.0-1015-aws   containerd://1.7.22
```


테스트 PC에서 k8s-s으로 전송해본다.


```bash
curl k8s-s:31615
Hostname: webpod1
IP: 127.0.0.1
IP: ::1
IP: 172.16.2.214
IP: fe80::1c35:ffff:feae:b43c
RemoteAddr: 192.168.10.200:33774 # 테스트 PC addr 확인
GET / HTTP/1.1
Host: k8s-s:31615
User-Agent: curl/7.81.0
Accept: */*
```


test pc에서 지속적으로 패킷을 전송하고 분석을 시작한다.


```bash
while true; do curl -s k8s-s:31615 | grep Hostname;echo "-----";sleep 1;done
```


#### k8s-s 에서 `80` 혹은 `$NODEPORT` port로 필터링 진행


아래와 같이 기본 구조라면 client ↔ k8s-s ↔ k8s-w1로 통신되어야 하기에 k8s-w1 > client로 향하는 트래픽도 보여야한다. 하지만, 여기서는 **client > k8s-w1로 향하는 트래픽만 보인다.**


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/4.png)


##### k8s-s에서 80 혹은 NodePort에 대한 패킷 dump


`tcpdump -eni any tcp port 80 or tcp port $NODEPORT -q`


여기서는 k8s-s에서 들어오는 패킷과 **파드에서 다시 클라이언트로 나가는 패킷을 확인**할 수 있다.


다시 처음에 들어온 서버로 향하지 않고, 해당 **파드에서 바로 클라이언트로 전송하는 DSR**을 볼 수 있다.


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/5.png)


또한, k8s-w1에서 testpc ip로 필터링하면, **k8s-s:nodeport**의 값으로 나가는 것을 확인할 수 있다.


```bash
tcpdump dst 192.168.10.200 -q
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens5, link-type EN10MB (Ethernet), snapshot length 262144 bytes
09:19:16.558066 IP k8s-s.31615 > testpc.36496: tcp 0
09:19:16.558661 IP k8s-s.31615 > testpc.36496: tcp 0
09:19:16.559356 IP k8s-s.31615 > testpc.36496: tcp 313
09:19:16.559658 IP k8s-s.31615 > testpc.36496: tcp 0
09:19:17.573835 IP k8s-s.31615 > testpc.36510: tcp 0
09:19:17.574125 IP k8s-s.31615 > testpc.36510: tcp 0
09:19:17.574489 IP k8s-s.31615 > testpc.36510: tcp 313
```


##### client에서 dump


마지막으로 Client에서 dump를 진행하면 아래와 같이 처음에 접근한 IP로 패킷이 들어오는 것을 확인할 수 있다. 위의 결과값과 동일하다. k8s-w1에서 패킷이 나갈때 src 정보로 자신의 노드 정보가 아닌 `k8s-s:nodeport`로 나간다.


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/6.png)


*해당 사진은 실습을 한번 종료하고 다시 진행한 것으로, NodePort 정보가 다른 실습 자료와 다릅니다.


#### 패킷 분석(k8s-s > k8s-w1)


이제 자세하게 패킷을 분석해보자. k8s-s 에서 목적지 파드로 보낼 때, 아래와 같이 IP 옵션을 추가한다.


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/7.png)


옵션의 값은 아래와 같다. 위의 값은 16진수이며 2개의 값이 하나의 바이트를 의미한다. 


```bash
	0x0000:  4700 0044 e984 4000 3f06 098a c0a8 0ac8  G..D..@.?.......
	0x0010:  ac10 02d6 9a08 7f7b 0a0a a8c0 c51a 0050  .......{.......P
	0x0020:  0344 df3b 0000 0000 a002 f507 80f9 0000  .D.;............
	0x0030:  0204 2301 0402 080a ba01 d86e 0000 0000  ..#........n....
	0x0040:  0103 0307
```


이제 자세하게 확인해본다.


앞의 47은 0100 0111로  4, 7의 의미를 갖는다. IP 패킷 구조는 **버전**과 **헤더의 길이**를 명시한다. 즉 4, 7의 의미는 ipv4이라는 뜻과 헤더의 길이는 7*4(padding) 28 byte라는 의미이다. 아래의 그림과 같이 20byte까지는 필수헤더 요소이다. 아래의 Option으로 28-20=8byte의 내용이 추가된 것을 알 수 있다.


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/8.png)


출처: [https://en.wikipedia.org/wiki/IPv4](https://en.wikipedia.org/wiki/IPv4)


옵션의 값을 확인해보자. 옵션은 20byte이후에 존재하니, 10개의 값을 건너뛰면 `9a08 7f7b 0a0a a8c0` 이다. 옵션은 순서대로 **유형**, **길이**에 대한 정보를 1byte씩 넣고, 이후 데이터를 넣는다. 즉 옵션 유형은 9a = 154이며, 길이는 08 = 8byte이다. 


나와있는 옵션 유형에 대한 [정보](https://www.iana.org/assignments/ip-parameters/ip-parameters.xhtml)에서 154 값에 대한 없다. 표준 옵션이 아니며, Cilium에서 사용하는 커스텀 옵션으로 보인다.


![image.png](/assets/img/post/Cilium%20DSR%20알아보기/9.png)


출처: [http://www.ktword.co.kr/test/view/view.php?no=1900](http://www.ktword.co.kr/test/view/view.php?no=1900)


값은 **0a0a a8c0**= 10, 10, 168, 192이다. 즉 k8s-s ip가 나오며, `7f7b`를 뒤집어서 `7b7f`으로 계산하면 31615가 나온다. 이는 **NodePort**이다. 클라이언트에서 **처음 방문한 노드의 ip**이며, 접근할 때 사용한 포트이다. `192.168.10.10:31615` 해당 정보를 통해 Cilium에서는 엔드포인트 파드에서 다시 클라이언트로 보낼 때, 자신이 위치한 노드의 IP가 아닌 클라이언트가 바라보는 `SRC IP:PORT`로 변경하여 DSR을 진행한다.


### 마치며


L3DSR은 MAC 주소를 기반으로 통신하는 L2DSR의 방식을 보완하는 방식이다. IP 방식이기에, 같은 네트워크 대역에 존재하지 않아도 DSR이 가능해 더 유연하다.


Cilium에서는 L3DSR과 같은 방식보다는 IP 프레임에 존재하는 Option을 통해 DSR을 구현한다. 내부 커널 단에서 옵션에 대한 정보가 들어있으면, 해당 정보와 일치하는 규칙을 부여하여 DSR을 처리하는 방식인것 같다. 



