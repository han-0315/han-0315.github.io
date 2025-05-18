---
layout: post
title: istio sidecar 자세히 살펴보기
date: 2025-05-18 09:02 +0900 
description: istio sidecar 자세히 살펴보기
category: [Kubernetes, Network] 
tags: [istio, Kubernetes, Network, istio#6, traffic, controle] 
pin: false
math: true
mermaid: true
---
istio sidecar 자세히 살펴보기
<!--more-->


### 들어가며


istio의 Sidecar 리소스는 프록시의 Ingress, Egress 트래픽을 제어하는 리소스다. 게이트웨이에서는 외부↔내부 트래픽 제어를 진행한다고 하면, Sidecar 리소스는 클러스터 내부에서 상세한 제어가 필요할 때 진행하는 것으로 보인다. 여기서는 Sidecar에 대해 자세히 살펴보고 실습해본다.


[https://istio.io/latest/docs/reference/config/networking/sidecar/](https://istio.io/latest/docs/reference/config/networking/sidecar/)


### Resource 형식


Sidecar의 리소스 형식은 아래와 같다. workloadSelector를 통해 특정 워크로드만 지정하거나, 셀렉터를 사용하지 않으면 namespace 단위로 지정할 수 있다. 네임스페이스의 설정이 없는 경우 istio-config(ex. istio-system) 설정을 따라간다.


```yaml
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
  name: connection-pool-settings
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  inboundConnectionPool:
      http:
        http1MaxPendingRequests: 1024
        http2MaxRequests: 1024
        maxRequestsPerConnection: 1024
        maxRetries: 100
  ingress:
  - port:
      number: 80
      protocol: HTTP
      name: somename
    connectionPool:
      http:
        http1MaxPendingRequests: 1024
        http2MaxRequests: 1024
        maxRequestsPerConnection: 1024
        maxRetries: 100
      tcp:
        maxConnections: 100

```



connectionPool에 대한 설정도 가능한데, 이는 이전 복원성과 관련된 [포스팅](https://www.handongbee.com/posts/%EB%B3%B5%EC%9B%90%EC%84%B1/#connection-pool)에서 확인할 수 있다.


### 적용 우선 순위 및 범위

1. **workloadSelector:** 해당 인스턴스에만 적용
2. **namespace:** workloadSelector가 없는 경우에 해당하며, 해당 네임스페이스의 모든 워크로드에 적용
	1. namespace 범위의 sidecar는 네임스페이스 별로 1개만 존재해야한다. 둘 이상이면 오동작
3. **전역 설정**: 아무 설정 없는 네임스페이스는 istio config(ex. `istio-system`)가 있는 네임스페이스 설정을 상속받음

*Gateway에는 사이드카 리소스가 적용되지 않는다.


### 실습 구성


간단한 httpbin 앱을 만들어두고, 트래픽 통신할 클라이언트 파드를 생성한다.

- istioninaction

```bash
kubectl apply -n istioinaction -f https://raw.githubusercontent.com/istio/istio/release-1.26/samples/httpbin/httpbin.yaml

serviceaccount/httpbin created
service/httpbin created
deployment.apps/httpbin created
```


```bash
kubectl run -n istioinaction sleep --image=nginx:alpine -- sleep 1d
pod/sleep created
```


비교를 위해 default 네임스페이스에도 httpbin 서비스를 배포한다.


```bash
kubectl apply -n default -f https://raw.githubusercontent.com/istio/istio/release-1.26/samples/httpbin/httpbin.yaml
serviceaccount/httpbin created
service/httpbin created
deployment.apps/httpbin created
```


### 서비스 레지스트리에 등록된 호스트만 허용하기


이전 4주차에 트래픽 제어 파트에서 ServiceEntry 실습을 할 때 아래와 같이 지정했었다. 


```bash
istioctl install --set profile=default --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
```


이는 아래의 Sidecar 모드와 동일하다.


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: istio-system
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY #or ALLOW_ANY
```


이렇게 지정하면, 허용되지 않는 호스트명은 제한된다. 


테스트를 진행해보자. 클러스터 내부 통신과 외부 통신(google.com)을 진행해본다.


```yaml
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://httpbin.istioinaction:8000/ip
{
  "origin": "127.0.0.6:43905"
}
```


google.com 


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```


둘 다 정상적으로 잘 통신된다. 이제 Sidecar 리소스를 적용한다.


```bash
k apply -f sidecar_REGISTRY_ONLY.yaml
sidecar.networking.istio.io/default created
```


클러스터 내부 서비스 통신은 정상적으로 허용되나


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://httpbin.istioinaction:8000/ip
{
  "origin": "127.0.0.6:37743"
}
```


외부 통신은 막힌다. 


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl http://google.com
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
```


### 적용 범위 확인


위에서 적용 우선순위는 “workloadSelector > namespace > 전역 설정”으로 진행된다고 했다. 여기서는 실제 리소스를 구성해서 우선순위를 확인해본다. 전역설정은 되어있으니, istioninaction 네임스페이스 설정을 진행해본다.


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: istioinaction
spec:
  outboundTrafficPolicy:
    mode: ALLOW_ANY
```


배포 후 명령어로 확인해보자.


```bash
kubectl get sidecar.networking.istio.io -A
NAMESPACE       NAME      AGE
istio-system    default   8m14s
istioinaction   default   10s
```


이제 다시 외부 통신(google.com)을 확인해보면


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```


정상적으로 잘 수행된다!


이로써, 전역설정보다 특정 네임스페이스 설정이 우선순위가 높다는 것을 확인할 수 있었다. 이제 특정 워크로드 설정을 진행해보자. 그렇게 하기 위해선 해당 네임스페이스 설정도 `REGISTRY_ONLY` 로 변경한다.


```bash
k edit -n istioinaction sidecars.networking.istio.io default 
...
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY # ALLOW_ANY -> REGISTRY_ONLY 변경
```


변경 후 다시 통신이 막힌 모습을 볼 수 있다.


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://google.com
```


이제 특정 워크로드(sleep 인스턴스)에만 다시 `ALLOW_ANY`로 적용한다.


```bash
cat outbound_workload.yaml 
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: workload
  namespace: istioinaction
spec:
  workloadSelector:
    labels:
      run: sleep
  outboundTrafficPolicy:
    mode: ALLOW_ANY
```


적용한 후 다시 외부 통신을 진행해보면


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```


정상적으로 통신된다.


이로써 “workloadSelector > namespace > 전역 설정” 적용 우선순위를 모두 확인해봤다.


### 호스트 제한


이제 호스트를 제한해보자.


```bash
cat outbound_workload.yaml 
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: workload
  namespace: istioinaction
spec:
  outboundTrafficPolicy:
    mode: REGISTRY_ONLY  
  workloadSelector:
    labels:
      run: sleep
  egress:
  - hosts:
    - "istioinaction/*"
```


sleep 워크로드에 대해서는 istioinaction 호스트만 허용하는 sidecar이다.


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://httpbin.istioinaction:8000/ip
{
  "origin": "127.0.0.6:55123"
}
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://httpbin.default:8000/ip
{
  "origin": "10.10.0.14:53560"
}
```


적용하기 전에 default 서비스도 잘 접속한다.


이제 적용해보자.


```bash
k apply -f outbound_workload.yaml 
sidecar.networking.istio.io/workload configured
```


적용 후 다시 테스트해보면, 아래와 같이 default는 막힌모습이다. istioninaction 호스트만 유일하게 허용된다.


```bash
kubectl exec -n istioinaction sleep -c sleep -- curl -s http://httpbin.default:8000/ip

kubectl exec -n istioinaction sleep -c sleep -- curl -s http://httpbin.istioinaction:8000/ip
{
  "origin": "127.0.0.6:43043"
}
```


### capture mode


stioIngressListener 또는 IstioEgressListener에서 앱으로 가는 트래픽을 어떻게 가로채서 Envoy로 전달할지결정하는 옵션이다. 기본적으로는 iptables를 사용한다.


```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: partial-ip-tables
  namespace: prod-us1
spec:
  workloadSelector:
    labels:
      app: productpage
  ingress:
  - bind: 172.16.1.32
    port:
      number: 80 # binds to 172.16.1.32:80
      protocol: HTTP
      name: somename
    defaultEndpoint: 127.0.0.1:8080
    captureMode: NONE
  egress:
    # use the system detected defaults
    # sets up configuration to handle outbound traffic to services
    # in 192.168.0.0/16 subnet, based on information provided by the
    # service registry
  - captureMode: IPTABLES
    hosts:
    - "*/*"

```


위의 예시 경우에서는 ingress에 대해서는  `captureMode: NONE` 으로 설정되어 실제 엔보이는 해당 포트(80)에 대해서 직접 ip(172.16.1.32)으로 바인딩한다. 통신이 되기 위해선 해당 `IP:Port`로 요청을 직접 보내야한다.


즉, productpage 서비스에 접속하기 위해선 ClusterIP가 아닌 **172.16.1.32:80로 라우팅되어야 엔보이에서 처리하여 실제 앱으로 전달된다.**

