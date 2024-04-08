---
layout: post
title: 쿠버네티스 Ingress 이해하기
date: 2024-01-12 15:00 +0900 
description: Nginx Ingress Controller를 통해 쿠버네티스 Ingress 이해하기
category: [Kubernetes, Network] 
tags: [Kubernetes, Services, Network, Ingress, Nginx] 
pin: false
math: true
mermaid: true
---
Nginx Ingress Controller를 통해 쿠버네티스 Ingress 이해하기
<!--more-->


## Ingress


### 등장 배경


서비스 타입으로는 provide load balancing, SSL termination and name-based virtual hosting이 불가능하다. Load Balancer 타입의 서비스로 운영하면, **외부 로드밸런서의 여러 개의 서비스를 붙일 수 없다.** 이런 제약 사항 때문에 L7계층 Application 수준 라우팅을 위한 별도의 리소스가 필요했고, [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)가 등장하게 된다.


### Ingress란


Ingress란 주로 L7 계층 라우팅을 담당하는 reverse Proxy server로, `Load Balancer Type` 서비스의 상위 리소스라고 보면 된다. L4 계층을 넘어 L7 계층의 라우팅을 담당한다. Ingress는 기본으로 내장된 컨트롤러가 없는 리소스로, 우리가 별도로 컨트롤러를 쿠버네티스에 설치해야 한다. **Ingress 리소스는 목표 상태를 의미**하며, **컨트롤러는 목표 상태에 맞게 L7(HTTP) 계층 라우팅 작업을 진행**한다.


### 구성


앞서, “`Load Balancer Type` 서비스의 상위 리소스”라고 설명했듯이 Ingress API는 ‘컨트롤러 구현체(주로 Deployment)’와 ‘Service(Load Balancer)’로 이뤄진다. 클라이언트는 **Ingress 전용 로드밸런서**로 접속하여, 인그레스 **컨트롤러 구현체**에게 전달되고, 컨트롤러는 규칙에 맞게 트래픽을 **엔드포인트(파드)로 전달**한다. 아래의 그림을 보면 이해하기 쉽다.


![ingress.svg](/assets/img/post/Nginx%20Ingress%20Controller/1.svg)


출처: [https://kubernetes.io/docs/concepts/services-networking/ingress/](https://kubernetes.io/docs/concepts/services-networking/ingress/)


### Class


**주로 어노테이션을 통해 자신의 상태 값을 지정하며, 컨트롤러는 어노테이션을 보고 작업을 진행한다.**


#### Class


컨트롤러는 `ingress.class`를 확인하여 자신이 작업할 Ingress가 맞는지 확인한다. `ingress.class`를 통해 하나의 클러스터에 여러 Ingress 컨트롤러가 공존할 수 있다.


```yaml
metadata:
  annotations:
    kubernetes.io/ingress.class: "example"
```


`ingress.class`를 제외한 대부분의 설정값들은 Ingress 컨트롤러마다 다르다. 


Nginx Ingress Controller: `nginx.ingress.kubernetes.io/rewrite-target`


Contour Ingress Controller: `projectcontour.io/response-timeout`


## Nginx Ingress Controller


여기서는 주로 사용하는 Nginx Ingress Controller에 대해 살펴본다. Nginx는 `nginx.conf` 파일을 통해 설정(라우팅 규칙)을 관리한다. 파드가 재성성되면, 파드의 IP가 변경된다. 이럴 때마다 설정파일(nginx.conf) reload가 필요하나, nginx는 lua-nginx-module을 통해 reload 없이 변경된 주소를 알 수 있다고 한다. [[커피고래님 블로그](https://coffeewhale.com/packet-network4) 참조]


### 설치(Helm)


실습을 진행하는 환경은 AWS EKS v1.29이며, Helm을 통해 Nginx Ingress Controller를 설치한다.


다른 방법으로 설치하고 싶다면, [공식문서](https://kubernetes.github.io/ingress-nginx/deploy/)를 참고하면 된다.

- **Helm repo**

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

- Chart 다운로드

```bash
helm install [RELEASE_NAME] ingress-nginx/ingress-nginx
```

- values.yaml(values.yaml의 전문은 [GitHub](https://github.com/kubernetes/ingress-nginx/blob/main/charts/ingress-nginx/values.yaml)에서 확인할 수 있다.)

`<CIDR>`: AWS EKS에서 VPC의 `CIDR`을 확인하여 같은 값을 넣어준다.


```yaml
controller:
  name: controller
  kind: Deployment
  dnsPolicy: ClusterFirst
  replicaCount: 1
  ingressClassByName: false
  ingressClassResource:
    name: nginx
    enabled: true
  scope:
    enabled: false
  config:
    proxy-real-ip-cidr: <CIDR>
    real-ip-header: "proxy_protocol"
    use-proxy-protocol: "false"
  service:
    enabled: true
    type: LoadBalancer
    externalTrafficPolicy: Cluster
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      service.beta.kubernetes.io/aws-load-balancer-type: nlb
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
      service.beta.kubernetes.io/aws-load-balancer-ssl-ports: "443"
      service.beta.kubernetes.io/aws-load-balancer-connection-idle-timeout: "60"
    external:
      enabled: true
    internal:
      enabled: false
  configMapNamespace: ""
  tcp:
    configMapNamespace: ""
    annotations: {}
  udp:
    configMapNamespace: ""
    annotations: {}
  affinity: {}
rbac:
  create: true
  scope: false
serviceAccount:
  create: true
  name: ""
```

- Nginx Ingress Controller 배포

```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  -f nginx.yaml
```

- 배포 확인

```bash
k -n ingress get all
NAME                                            READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-55474d95c5-7cd6k   1/1     Running   0          31s

NAME                                         TYPE           CLUSTER-IP     EXTERNAL-IP                                                                          PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.100.7.242   a22390a4d1e244dc2bfeb13e9965adc6-9a8b42b9ac85ec37.elb.ap-northeast-2.amazonaws.com   80:31615/TCP,443:30893/TCP   31s
service/ingress-nginx-controller-admission   ClusterIP      10.100.13.28   <none>                                                                               443/TCP                      31s

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           31s

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-55474d95c5   1         1         1       31s
```


### 예시 리소스 배포하여 확인하기


파드, 서비스, Ingress를 각각 배포한다. 파드는 자신의 클러스터 내부 IP를 호출하는 웹서버이며 서비스는 그런 파드를 노출하고, 인그레스는 `/run`으로 접근하는 요청을 해당 서비스로 라우팅하는 규칙을 작성한다.


#### 파드


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two
  labels:
    id: two
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared-volume
      mountPath: /usr/share/nginx/html
    ports:
    - containerPort: 80
  initContainers:
  - name: init-myservice
    image: busybox
    command: ['sh', '-c', "echo $(hostname -i) > /usr/share/nginx/html/index.html"]
    volumeMounts:
    - name: shared-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: shared-volume
    emptyDir: {}
```


#### 서비스


```bash
k expose pod two --port 80 --target-port 80
```


#### Ingress


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: two-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /run
        pathType: Prefix
        backend:
          service:
            name: two
            port:
              number: 80
```


#### 테스트


서비스에 명시된 EXTERNAL-IP를 확인한다.


```bash
$ kubectl -n ingress get svc
NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.100.251.97   a671d582b8668437ea7d3cc0abe24c44-79da897ea32a3e41.elb.ap-northeast-2.amazonaws.com   80:31831/TCP,443:30476/TCP   17m
ingress-nginx-controller-admission   ClusterIP      10.100.72.8     <none>
```


브라우저 혹은 curl 명령어로 `<EXTERNAL-IP>/run` 으로 접근한다. 그러면 아래와 같이 Pod의 IP가 출력된다.


![Untitled.png](/assets/img/post/Nginx%20Ingress%20Controller/2.png)


Nginx Controller 파드에 접속하여, 로그도 확인할 수 있다.


```bash
❯ k -n ingress logs ingress-nginx-controller-55474d95c5-dgk85 
...
[08/Apr/2024:12:07:54 +0000] "GET /run HTTP/1.1" 200 13 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.0.0 Safari/537.36" 515 0.001 [default-two-80] [] 192.168.6.58:80 13 0.000 200 e612b05a47983cba52e8890028c76972
```


### 작동 방식


작동 방식은 공식문서에서 제공된 아래 그림을 통해 쉽게 이해할 수 있다. 컨트롤러가 지속해서 리소스의 변화를 확인하고 변화가 감지하면 TLS와 config 파일을 업데이트한다. 그리고 reload 하는 방식으로 진행된다.


![ic-process.png](/assets/img/post/Nginx%20Ingress%20Controller/3.png)


[출처: [https://docs.nginx.com/nginx-ingress-controller/overview/design/](https://docs.nginx.com/nginx-ingress-controller/overview/design/)]


아래의 명령어를 통해 로그를 확인할 수 있다. 로그를 통해 새로운 Ingress가 생성되면 컨트롤러 내부에서 어떤 프로세스가 진행되는지 알아볼 수 있다.


```bash
k -n ingress logs ingress-nginx-controller-55474d95c5-pnrd4
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v1.10.0
  Build:         71f78d49f0a496c31d4c19f095469f3f23900f8a
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.25.3

-------------------------------------------------------------------------------
...
store.go:440] "Found valid IngressClass" ingress="default/two-ingress" ingressclass="nginx"
event.go:364] Event(v1.ObjectReference{Kind:"Ingress", Namespace:"default", Name:"two-ingress", ... type: 'Normal' reason: 'Sync' Scheduled for sync
controller.go:190] "Configuration changes detected, backend reload required"
controller.go:210] "Backend successfully reloaded"
event.go:364] Event(v1.ObjectReference{Kind:"Pod", Namespace:"ingress", Name:"ingress-nginx-controller-55474d95c5-dgk85", ... reason: 'RELOAD' NGINX reload triggered due to a change in configuration
```


이제 시간 순서대로 Ingress 관련 로그를 살펴본다.


**(1)** nginx ingress.class를 가진 ingress를 발견했다는 로그이다.


```bash
"Found valid IngressClass" ingress="default/two-ingress" ingressclass="nginx"
```


**(2)** 발견한 Ingress를 가져와 Sync, 동기화를 진행한다.


```bash
Event(v1.ObjectReference{Kind:"Ingress", Namespace:"default", Name:"two-ingress", ... type: 'Normal' reason: 'Sync' Scheduled for sync
```


**(3)** 설정의 변화가 감지되어, backend 객체가 reload 된다.


```bash
"Configuration changes detected, backend reload required"
"Backend successfully reloaded"
```


로그를 통해 작동 방식을 자세하게 살펴볼 수 있었다. 이제 nginx ingress controller 파드에 접속하여, 설정파일(nginx.conf)을 확인해보자.


#### 설정 파일 확인


아래의 명령어를 통해 nginx.conf 파일에서 ingress 설정을 확인할 수 있다. 


`location ~* "^/run"`  이후를 확인하면, 우리가 배포한 Ingress와 서비스에 대한 정보가 나와있다.


```text
$ k -n ingress exec ingress-nginx-controller-55474d95c5-pnrd4 -it -- cat nginx.conf
...
	## start server _
	server {
		server_name _ ;

		http2 on;
		
		listen 80 default_server reuseport backlog=4096 ;
		listen [::]:80 default_server reuseport backlog=4096 ;
		listen 443 default_server reuseport backlog=4096 ssl;
		listen [::]:443 default_server reuseport backlog=4096 ssl;

		set $proxy_upstream_name "-";

		ssl_reject_handshake off;

		ssl_certificate_by_lua_block {
			certificate.call()
		}

		location ~* "^/run" {

			set $namespace      "default";
			set $ingress_name   "two-ingress";
			set $service_name   "two";
			set $service_port   "80";
			set $location_path  "/run";
			set $global_rate_limit_exceeding n;
```


### Annotation


#### Canary(Weight)


공식 문서를 참고하면, `nginx.ingress.kubernetes.io/canary: \"true\"` 어노테이션을 통해 카나리 배포에 도움을 줄 수 있다. 해당 어노테이션으로 기본 라우팅 서비스에서 카나리 서비스로 weight만큼 트래픽을 흘릴 수 있다.

- Prod

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: production
  annotations:
spec:
  ingressClassName: nginx
  rules:
  - host: echo.prod.mydomain.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: production
            port:
              number: 80
```

- Canary(Staging, …)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary
  annotations:
    nginx.ingress.kubernetes.io/canary: \"true\"
    nginx.ingress.kubernetes.io/canary-weight: \"30\"
spec:
  ingressClassName: nginx
  rules:
  - host: echo.prod.mydomain.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: canary
            port:
              number: 80
```


#### rewrite


URI를 변경하는 어노테이션이다. `example.com/a/b` 경로이지만, `example/b`로 URI를 변경할 수 있다. 


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  name: rewrite
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: rewrite.bar.com
    http:
      paths:
      - path: /something(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: http-svc
            port: 
              number: 80
```


어노테이션의 결과로 아래의 URI가 다음과 같이 변경된다.

- `rewrite.bar.com/something` **>** `rewrite.bar.com/`
- `rewrite.bar.com/something/` **>** `rewrite.bar.com/`
- `rewrite.bar.com/something/new` **>** `rewrite.bar.com/new`

#### SSL redirect


들어온 트래픽을 HTTPS로 리다이렉트한다. 


```yaml
nginx.ingress.kubernetes.io/ssl-redirect: "true" or "false"
```


## 참고자료


[https://kubernetes.io/ko/docs/concepts/services-networking/ingress/](https://kubernetes.io/ko/docs/concepts/services-networking/ingress/)


[https://docs.nginx.com/nginx-ingress-controller/overview/design/](https://docs.nginx.com/nginx-ingress-controller/overview/design/)


[https://catalog.us-east-1.prod.workshops.aws/workshops/9c0aa9ab-90a9-44a6-abe1-8dff360ae428/ko-KR/60-ingress-controller/100-launch-alb](https://catalog.us-east-1.prod.workshops.aws/workshops/9c0aa9ab-90a9-44a6-abe1-8dff360ae428/ko-KR/60-ingress-controller/100-launch-alb)


[https://medium.com/@danielepolencic/learning-how-an-ingress-controller-works-by-building-one-in-bash-ac3929f7699](https://medium.com/@danielepolencic/learning-how-an-ingress-controller-works-by-building-one-in-bash-ac3929f7699)ㄴ


[https://coffeewhale.com/packet-network4](https://coffeewhale.com/packet-network4)


[https://medium.com/google-cloud/kubernetes-ingress-vs-gateway-api-647ee233693d](https://medium.com/google-cloud/kubernetes-ingress-vs-gateway-api-647ee233693d)


[https://dramasamy.medium.com/life-of-a-packet-in-kubernetes-part-4-4dbc5256050a](https://dramasamy.medium.com/life-of-a-packet-in-kubernetes-part-4-4dbc5256050a)


[https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html)

