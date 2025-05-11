---
layout: post
title: istio Security(인가)
date: 2025-05-11 09:02 +0900 
description: istio Security 인가부분 살펴보기
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#5, Security, SPIFFE] 
pin: false
math: true
mermaid: true
---
istio Security 인가부분 살펴보기
<!--more-->


### 들어가며


이전 포스팅에서는 istio의 인증에 대해 살펴봤다. 인증은 PeerAuthentication, RequestAuthentication이 존재한다. 여기서는 istio의 보안과 관련된 리소스 중 마지막 **AuthorizationPolicy**에 대해 살펴볼 예정이다. 


![image.png](/assets/img/post/istio%20Security(인가)/1.png)


출처: 스터디원(istio security)


### 인가


인증은 상대방의 신원을 조회했다면, 인가는 요청한 행동에 대한 자격이 있는지 확인하는 과정이다. 인가가 존재한다면, 인증서를 유출했어도 접근 가능한 범위를 해당 인증서의 권한 범위로 줄일 수 있다.


![image.png](/assets/img/post/istio%20Security(인가)/2.png)


출처: 스터디원(istio security)


istio의 인가 정책 리소스는 **AuthorizationPolicy** 하나만 존재한다. 먼저 해당 리소스의 형식에 대해 살펴보자.


### 리소스 형식

- scope
	- selector(`matchLabel`)
- rules(or)
	- 하나 이상의 규칙이라도 일치하면 적용된다.
	- **반대로 특정 워크로드에 대한 allow 규칙이 있다면, 해당 규칙이 없는 트래픽은 모두 거부된다.**
	- `from`
		- `namespace`, `ipBlocks`
		- `principals`:  **PeerAuthentication**으로 설정한 mTLS 워크로드 대상
		- `requestPrincipals`:  **Request Authentication** **JWT** 대상
	- `to`
		- `operation`: `paths`, `methods`, `hosts`, `ports`
	- `when`: 별도 커스텀 조건(key, value)
		- 사용할 수 있는 key는 [docs](https://istio.io/latest/docs/reference/config/security/conditions/)에서 확인할 수 있다.
- action: `ALLOW`, `DENY`, `AUDIT`, `CUSTOM`, `empty(DENY)`

ex) webapp으로 향하는 트래픽 중 `/api/catalog*` 에 대한 트래픽만 허용


```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "allow-catalog-requests-in-web-app"
  namespace: istioinaction
spec:
  selector:
    matchLabels:
      app: webapp # 워크로드용 셀렉터 Selector for workloads
  rules:
  - to:
    - operation:
        paths: ["/api/catalog*"] # 요청을 경로 /api/catalog 와 비교한다 Matches requests with the path /api/catalog
  action: ALLOW # 일치하면 허용한다 If a match, ALLOW
```


### AuthorizationPolicy 적용해보기


기존의 인증 단계에서 사용했던 리소스를 그대로 사용하며, webapp에 대해 위의 예시로 언급한 AuthoricationPolicy을 적용해보자.


아래의 리소스를 적용하면, webapp으로 들어오는 트래픽에 대해 `/api/catalog` prefix로 시작하는 요청만 허용된다. (나머지는 거부된다.)


```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "allow-catalog-requests-in-web-app"
  namespace: istioinaction
spec:
  selector:
    matchLabels:
      app: webapp 
  rules:
  - to:
    - operation:
        paths: ["/api/catalog*"] 
  action: ALLOW 
```


적용 후 다시 sleep 파드에서 werbapp으로 요청을 보내보자.


`/api/catalog` 요청


```bash
kubectl exec deploy/sleep -- curl -sSL webapp.istioinaction/api/catalog
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```


`/hello/world` 요청 → 거부되는 것을 확인할 수 있다.


```bash
kubectl exec deploy/sleep -- curl -sSL webapp.istioinaction/hello/world 
RBAC: access denied
```


kiali에서 확인하면, 아래와 같다.


![image.png](/assets/img/post/istio%20Security(인가)/3.png)


이제 다음 실습을 통해 인가 리소스를 삭제한다.


```bash
kubectl delete -f ch9/allow-catalog-requests-in-web-app.yaml
```


#### 모든 트래픽 거부


위의 형식부분에서 봤듯이 actions을 비워두면 `Deny`로 설정된다. selector 등으로 지정하지 않을 경우 모든 워크로드로 지정되며 아래와 같이 비워두면 모든 워크로드에 `Deny` 설정이 적용된다. 


```yaml

apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: istio-system # 이스티오를 설치한 네임스페이스의 정책은 메시의 모든 워크로드에 적용된다
spec: {} # spec 이 비어있는 정책은 모든 요청을 거부한다
```


설정 적용 후 통신을 확인해보면 아래와 같이 모두 거부된다.


```bash
curl -s http://webapp.istioinaction.io:30000/api/catalog
RBAC: access denied
```


```bash
kubectl exec deploy/sleep -- curl -sSL webapp.istioinaction/api/catalogalog
RBAC: access denied
```


webapp proxy 로그를 확인해보면, 아래와 같이 모든 요청이 무시된다. 


```bash
[2025-05-11T05:15:24.816Z] "GET /api/catalog HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "-" "curl/8.5.0" "1c29b4b4-acdd-9196-b093-322fd083ac38" "webapp.istioinaction" "-" inbound|8080|| - 10.10.0.16:8080 10.10.0.14:38380 - -
[2025-05-11T05:15:26.965Z] "GET /api/catalog HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "-" "curl/8.5.0" "780b5494-90d9-99a1-a3ee-9383b773b8ff" "webapp.istioinaction" "-" inbound|8080|| - 10.10.0.16:8080 10.10.0.14:38382 - -
[2025-05-11T05:15:35.865Z] "GET /hello HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "-" "curl/8.5.0" "a302f316-82c5-9d52-b4de-60e286c3ea8e" "webapp.istioinaction" "-" inbound|8080|| - 10.10.0.16:8080 10.10.0.14:46204 - -
[2025-05-11T05:15:36.004Z] "GET /hello HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "-" "curl/8.5.0" "e62b7ae0-d37b-9642-b7f5-5a34072ae887" "webapp.istioinaction" "-" inbound|8080|| - 10.10.0.16:8080 10.10.0.14:46216 - -
```


ingress gateway의 로그를 확인해보면, 아래와 같이 적용 후 webapp으로 향하는 트래픽이 거부된다.


```bash
kubectl logs -n istio-system -l app=istio-ingressgateway -f-f
2025-05-11T01:37:55.739675Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T02:06:00.045187Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T02:36:51.768059Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T03:09:43.210126Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T03:39:20.555071Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T04:09:15.635581Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T04:39:04.804576Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
2025-05-11T05:11:49.135021Z	info	xdsproxy	connected to upstream XDS server: istiod.istio-system.svc:15012
[2025-05-11T05:14:26.022Z] "GET /api/catalog HTTP/1.1" 403 - rbac_access_denied_matched_policy[none] - "-" 0 19 0 - "172.18.0.1" "curl/8.5.0" "5ae164d2-1cf3-9d91-b384-90dcd052b46a" "webapp.istioinaction.io:30000" "-" outbound|80||webapp.istioinaction.svc.cluster.local - 10.10.0.6:8080 172.18.0.1:53824 - -
```


#### 특정 네임스페이스 허용


sleep pod가 존재하는 namespace에 대해 istioinaction(webapp)으로 향하는 트래픽을 허용해보자.


```bash
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "webapp-allow-view-default-ns"
  namespace: istioinaction # istioinaction의 워크로드
spec:
  rules:
  - from: # default 네임스페이스에서 시작한
    - source:
        namespaces: ["default"]
    to:   # HTTP GET 요청에만 적용 
    - operation:
        methods: ["GET"]
```


적용 후 다음과 같이 요청을 하면


`webapp.istioinaction` 요청


```bash
kubectl exec deploy/sleep -- curl -sSL webapp.istioinaction
<!doctype html>
<html lang="en">
<head>
  ...
```


`webapp.istioinaction/api/catalog` 요청 → 거부되는 것을 확인할 수 있다.


```bash
kubectl exec deploy/sleep -- curl -sSL webapp.istioinaction/api/catalog
error calling Catalog service
```


이는 아래와 같이 webapp에서 catalog로 요청이 필요하기 때문이다. (default → istioninaction)으로 GET요청을 보내는 것만 허용했으니 istioninaction → istioninaction은 거부되어서 그렇다.


![image.png](/assets/img/post/istio%20Security(인가)/4.png)


#### 특정 서비스 어카운트 요청 허용


서비스 어카운트 정보는 SVID에도 존재하며, mTLS 통신 중 SA 정보를 검증하고 메타데이터에 저장한다.


webapp의 서비스 어카운트를 기반으로 오는 요청을 허용하도록 진행한다.


```bash
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "catalog-viewer"
  namespace: istioinaction
spec:
  selector:
    matchLabels:
      app: catalog
  rules:
  - from:
    - source: 
        principals: ["cluster.local/ns/istioinaction/sa/webapp"] # Allows requests with the identity of webapp
    to:
    - operation:
        methods: ["GET"]
```


이제 정책을 적용하고 다시 


`webapp.istioinaction/api/catalog` 요청을 진행해본다.


```bash
kubectl exec deploy/sleep -- curl -sSL webapp.istioinaction/api/catalog
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```


istioninaction → istioninaction 네임스페이스에 대한 정책을 허용하지 않았지만, webapp의 SA를 기반으로 통신이 허용되는 것을 볼 수 있다.


![image.png](/assets/img/post/istio%20Security(인가)/5.png)


#### `WHEN` 정책


위에서 from, to 조건이 일치하고 별도로 WHEN 정책을 추가로 적용할 수 있다. 아래의 예시로 살펴보면 모든 정책을 ALLOW하지만, 실제로는  `auth@istioinaction.io/*` 에 의해 발근된 토큰이거나 JWT에 값이 admin group claim이 포함된 경우에만 통신을 허용한다.


```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "AuthorizationPolicy"
metadata:
  name: "allow-mesh-all-ops-admin"
  namespace: istio-system
spec:
  rules:
  - from:
    - source:
        requestPrincipals: ["auth@istioinaction.io/*"]
    when:
    - key: request.auth.claims[groups] # 이스티오 속성을 지정한다
      values: ["admin"] # 반드시 일치해야 하는 값의 목록을 지정한다
```


#### 결정 순서


Custom > Deny > Allow 순서로 결정된다. 즉, Allow, Deny가 같이 있으면 기본적으론 요청은 거부된다.


![image.png](/assets/img/post/istio%20Security(인가)/6.png)


출처: [https://istio.io/latest/docs/concepts/security/#implicit-enablement](https://istio.io/latest/docs/concepts/security/#implicit-enablement)


### action: Custom 


여기서는 action 필드를 Custom으로 설정하여 외부 인가와 연동해보자.


#### 샘플 인가 서비스 배포


해당 인가 서비스는 아주 간단하게 요청에 `x-ext-authz` 헤더 값이 `allow`라면 요청을 허용하는 서비스이다. 


```bash
docker exec -it myk8s-control-plane bash
---
kubectl apply -f istio-$ISTIOV/samples/extauthz/ext-authz.yaml -n istioinaction
```


#### istio-system config map 수정


아래와 같이 configmap을 수정하여 우리가 이번에 배포한 외부 인가 서비스를 인식할 수 있도록 설정한다. 그 값은 `extensionProviders` 필드에 넣어두면 된다.


```bash
kubectl edit -n istio-system cm istio

    extensionProviders:
    _- name: "sample-ext-authz-http"
      envoyExtAuthzHttp:
        service: "ext-authz.istioinaction.svc.cluster.local"
        port: "8000"
        includeRequestHeadersInCheck: ["x-ext-authz"]_
```


#### Policy 배포


istioinaction 네임스페이스의 webapp 워크로드로 들어오는 트래픽을 검사할 예정이다.


```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ext-authz
  namespace: istioinaction
spec:
  selector:
    matchLabels:
      app: webapp
  action: CUSTOM    # custom action 사용
  provider:
    name: sample-ext-authz-http  # meshconfig 이름과 동일해야 한다
  rules:
  - to:
    - operation:
        paths: ["/*"]  # 인가 정책을 적용할 경로
```


#### 통신 확인


헤더 없이 통신을 진행해보자 아래와 같이 “denied by ext_authz”를 확인할 수 있다. 


“header `x-ext-authz: allow` in the request"를 통해 실제 왜 거부되었는지 이유도 볼 수 있다.


```yaml
kubectl -n default exec -it deploy/sleep -- curl webapp.istioinaction/api/catalog
denied by ext_authz for not found header `x-ext-authz: allow` in the request
```


##### ext-authz log


`[HTTP][denied]: GET webapp.istioinaction/api/catalog`트래픽을 헤더를 검사하여 거부했다는 로그를 볼 수 있다.


```bash
 kubectl logs -n istioinaction -l app=ext-authz -c ext-authz -f
2025/05/11 05:51:53 Starting gRPC server at [::]:9000
2025/05/11 05:51:53 Starting HTTP server at [::]:8000
2025/05/11 05:55:53 [HTTP][denied]: GET webapp.istioinaction/api/catalog, headers: map[Content-Length:[0] X-B3-Parentspanid:[0eb6c1dc45c026ab] X-B3-Sampled:[1] X-B3-Spanid:[339cdadfbf5ec5a6] X-B3-Traceid:[3b3fef6df5c8345973ec314d6ee1dd73] X-Envoy-Expected-Rq-Timeout-Ms:[600000] X-Envoy-Internal:[true] X-Forwarded-Client-Cert:[By=spiffe://cluster.local/ns/istioinaction/sa/default;Hash=23c3b9684b09083846537817c42b25343268218c1fdf8b85b6053e0aa6535646;Subject="";URI=spiffe://cluster.local/ns/istioinaction/sa/webapp] X-Forwarded-For:[10.10.0.16] X-Forwarded-Proto:[https] X-Request-Id:[f3c603d5-be24-9d34-a0ba-413725fdead9]], body: []
2025/05/11 05:56:01 [HTTP][denied]: GET webapp.istioinaction/api/catalog, headers: map[Content-Length:[0] X-B3-Parentspanid:[22b07d2d8d4ee609] X-B3-Sampled:[1] X-B3-Spanid:[151418893647f6a6] X-B3-Traceid:[42c77b1eb5276dbcaecb79f4df6dff5a] X-Envoy-Expected-Rq-Timeout-Ms:[600000] X-Envoy-Internal:[true] X-Forwarded-Client-Cert:[By=spiffe://cluster.local/ns/istioinaction/sa/default;Hash=23c3b9684b09083846537817c42b25343268218c1fdf8b85b6053e0aa6535646;Subject="";URI=spiffe://cluster.local/ns/istioinaction/sa/webapp] X-Forwarded-For:[10.10.0.16] X-Forwarded-Proto:[https] X-Request-Id:[08029a19-bfae-950a-aef5-6bd3351aac45]], body: []
2025/05/11 05:56:15 [HTTP][denied]: GET webapp.istioinaction/api/catalog, headers: map[Content-Length:[0] X-B3-Parentspanid:[df5ef18e3330d307] X-B3-Sampled:[1] X-B3-Spanid:[25a9ee3650bffb2a] X-B3-Traceid:[4166abbf69b60476b72d15de3e8f67de] X-Envoy-Expected-Rq-Timeout-Ms:[600000] X-Envoy-Internal:[true] X-Forwarded-Client-Cert:[By=spiffe://cluster.local/ns/istioinaction/sa/default;Hash=23c3b9684b09083846537817c42b25343268218c1fdf8b85b6053e0aa6535646;Subject="";URI=spiffe://cluster.local/ns/istioinaction/sa/webapp] X-Forwarded-For:[10.10.0.16] X-Forwarded-Proto:[https] X-Request-Id:[5e67826f-5969-9e82-990d-2834c1a7d28c]], body: []
```


#### 헤더 추가하여 통신 진행


헤더를 추가하니, 정상적으로 값을 잘 가져온다.


```bash
kubectl -n default exec -it deploy/sleep -- curl -H "x-ext-authz:allow" webapp.istioinaction/api/catalog
[{"id":1,"color":"amber","department":"Eyewear","name":"Elinor Glasses","price":"282.00"},{"id":2,"color":"cyan","department":"Clothing","name":"Atlas Shirt","price":"127.00"},{"id":3,"color":"teal","department":"Clothing","name":"Small Metal Shoes","price":"232.00"},{"id":4,"color":"red","department":"Watches","name":"Red Dragon Watch","price":"232.00"}]
```


로그 또한 정상적으로 보인다.


```bash
2025-05-11T05:57:22.375660Z	debug	envoy rbac external/envoy/source/extensions/filters/http/rbac/rbac_filter.cc:130	shadow denied, matched policy istio-ext-authz-ns[istioinaction]-policy[ext-authz]-rule[0]	thread=30
2025-05-11T05:57:22.375668Z	debug	envoy rbac external/envoy/source/extensions/filters/http/rbac/rbac_filter.cc:167	no engine, allowed by default	thread=30
[2025-05-11T05:57:22.377Z] "GET /items HTTP/1.1" 200 - via_upstream - "-" 0 502 2 2 "-" "beegoServer" "e638da6f-6f0e-92b1-bc17-970d7ace06cc" "catalog.istioinaction:80" "10.10.0.15:3000" outbound|80||catalog.istioinaction.svc.cluster.local 10.10.0.16:59726 10.200.1.138:80 10.10.0.16:34594 - default
[2025-05-11T05:57:22.375Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 5 3 "-" "curl/8.5.0" "e638da6f-6f0e-92b1-bc17-970d7ace06cc" "webapp.istioinaction" "10.10.0.16:8080" inbound|8080|| 127.0.0.6:50957 10.10.0.16:8080 10.10.0.18:59660 - defaul
```


이렇게 istio에서는 외부 인가 서비스를 연동하여 커스터마이징이 가능하다. 실제 다른 회사에서는 어떻게 쓰는 지 궁금하다.


### 마치며


이번 포스팅을 마지막으로 istio security 부분을 모두 마무리한다. istio는 기본적으로 SVID를 통해 각 서비스별로 ID(인증서)를 가지며 이를 기반으로 mTLS 통신이 가능하다. 또, 모든 트래픽에 인증(mTLS, JWT), 인가에 대한 설정이 가능하다. 인가 정책은 Custom을 사용하여 실제 별도 인가와 관련된 서비스를 배포하고, 이를 이용할 수 있다.

