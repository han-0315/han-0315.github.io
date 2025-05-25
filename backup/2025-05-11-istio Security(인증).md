---
layout: post
title: istio Security(인증)
date: 2025-05-10 09:01 +0900 
description: istio Security(인증) 살펴보기
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#5, Security, 인증] 
pin: false
math: true
mermaid: true
---
istio Security(인증) 살펴보기
<!--more-->


### 들어가며


kubernetes에서는 자체의 인증/인가/admisstion controller 기능이 있지만, 해당 부분은 kube API에 대한 보안 절차이다. istio에서는 서비스 간의 네트워크 통신에서 인증 기능을 설정할 수 있다. 만약 클라우드 환경이고 간혈적으로 public network를 이용한다면 이런 기능은 필수라고 생각된다. 또, 온프렘이더라도 공용(멀티 테넌시)클러스터를 운영한다면 istio에 이런 트래픽 보안 기능에 대한 니즈가 있지 않을까 싶다.


### 실습 환경


istio 실습 환경은 이전과 동일하며, 여기서 추가로 배포되는 리소스는 아래와 같다.


자주보던 webapp, catalog이외에 기존의 netshoot이 아닌 sleep앱을 별도로 생성하여 클라이언트로 사용한다. **sleep앱은 default namespace에 생성한다. (istio 시스템을 적용받지 않음)**


```bash
kubectl apply -f services/catalog/kubernetes/catalog.yaml -n istioinaction
kubectl apply -f services/webapp/kubernetes/webapp.yaml -n istioinaction 
kubectl apply -f services/webapp/istio/webapp-catalog-gw-vs.yaml -n istioinaction
kubectl apply -f ch9/sleep.yaml -n default
```


# Authentication


istio의 인증 기능을 명세한 리소스로는 PeerAuthentication , RequestAuthentication가 존재한다.


## PeerAuthentication


PeerAuthentication 리소스를 사용하면 mTLS를 요구가 가능하며 평문 트래픽을 허용여부를 설정할 수 있다.


우선 mTLS에 대해 먼저 살펴보자.


#### mTLS


TLS는 HTTPS 등으로 인해 한번쯤 들어봤을 인증과 관련된 프로토콜이다. mTLS는 mutual TLS로 기존에는 서버 측만 인증을 진행했다면, mTLS는 클라이언트 측도 동일하게 자신의 신분을 증명하는 프로토콜이다.


아래는 HTTPS 핸드쉐이크 과정 예시이다.


```bash
curl -v 8 https://google.com
*   Trying 0.0.0.8:80...
...
* ALPN: curl offers h2,http/1.1
* (304) (OUT), TLS handshake, Client hello (1):
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
```


### 형식


리소스의 형식은 아래와 같다. 

- scope
	- namespace
		- 전체 범위: istio가 설치된 namespace로 설정하고, name = default로 설정
		- 특정 범위: 특정 namespace 명시
	- workload
		- selector를 통해 특정 서비스(파드) 지정 가능
- mtls.mode
	- STRICT: mTLS 강제화, mTLS가 아닌 트래픽은 허용하지 않음
	- PERMISSIVE: 평문, mTLS 모두 허용
	- UNSET: 상위 정책 상속
	- DISABLE: 비활성화

```bash
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default" # Mesh-wide policies must be named "default"
  namespace: "istio-system" # Istio installation namespace
spec:
  mtls:
    mode: STRICT # mutual TLS mode
```


### 모든 트래픽 거부


모든 트래픽 거부를 위해선 scope는 전체 네임스페이스로 적용해야 한다. 이를 위해 istio가 설치된 네임스페이스로 설정하고, name을 `default`로 지정한다.


```bash
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default" # Mesh-wide policies must be named "default"
  namespace: "istio-system" # Istio installation namespace
spec:
  mtls:
    mode: STRICT # mutual TLS mode
```


이제 webapp에 접속해보면, 다음과 같이 오류가 발생한다.


```bash
kubectl exec deploy/sleep -c sleep -- curl -s http://webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"
000
command terminated with exit code 56
```


webapp 로그를 한번 확인해보자.


아래와 같이 10.10.0.14에서 10.10.0.13으로 들어오는 트래픽이 막혔다는 것을 볼 수 있다. 


```bash
kubectl logs -n istioinaction -l app=webapp -c istio-proxy -f
[2025-05-10T16:42:25.024Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:51048 - -
[2025-05-10T16:42:27.154Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:51060 - -
[2025-05-10T16:42:29.294Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:51072 - -
[2025-05-10T16:42:31.474Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:51076 - -
[2025-05-10T16:42:33.624Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:50066 - -
[2025-05-10T16:42:35.765Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:50068 - -
[2025-05-10T16:42:37.915Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:50080 - -
[2025-05-10T16:42:40.074Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:50090 - -
[2025-05-10T16:42:42.234Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:50096 - -
[2025-05-10T16:42:44.374Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:49320 - -
[2025-05-10T16:42:46.537Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:49326 - -
[2025-05-10T16:42:48.714Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:49340 - -
```


관련 IP에 대한 파드를 조회해보면, sleep과 webapp이라는 것을 알 수 있다. (우리가 조회한 것과 당연히 동일)


```bash
kubectl get pods -A -o wide | grep -E '10.10.0.13|10.10.0.14'
default              sleep-6f8cfb8c8f-v2smw                        1/1     Running   0          12m     10.10.0.14   myk8s-control-plane   <none>           <none>
istioinaction        webapp-7685bcb84-l5c4m                        2/2     Running   0          13m     10.10.0.13   myk8s-control-plane   <none>           <none>
```


에러 로그는 `0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:51048 - -` 와 같은데, 여기서  NR은 “Non-Route”를 의미한다. Envoy에서 라우팅까지 가지 못하고 실패했다는 것을 의미하며 “**filter_chain_not_found**”은 해당 Listener에서 제공된 SNI(Server Name Indication), IP, 포트, ALPN 등의 조건에 맞는 filter_chain이 설정에 없다는 뜻이라고 한다.


즉, 평문에 대한 요청이 거부되었다는 의미이다. 우리는 별도 istio proxy가 없는 sleep에서 통신을 http로 진행했기에 `curl -s webapp.istioinaction/api/catalog`로 당연한 결과이다.


### 특정 엔드포인트만 허용


webapp으로 들어오는 트래픽은 mTLS가 아닌 경우, 즉 평문도 허용하도록 아래와 같이 설정한다.


```bash
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "webapp"
  namespace: "istioinaction"
spec:
  selector:
    matchLabels:
      app: "webapp"  # 레이블이 일치하는 워크로드만 PERMISSIVE로 동작
  mtls:
    mode: PERMISSIVE
```


이제 통신 테스트를 진행하면, 예상했듯 성공한다.


```bash
kubectl exec deploy/sleep -c sleep -- curl -s webapp.istioinaction/api/catalog -o /dev/null -w "%{http_code}\n"
200
```


webapp istio-proxy의 로그를 확인해보자. 통신이 진행되지 않다가, 위의 `PeerAuthentication` 설정 후 정상적으로 통신이 되는 모습이다.


```bash
[2025-05-10T16:48:08.954Z] "- - -" 0 NR filter_chain_not_found - "-" 0 0 0 - "-" "-" "-" "-" "-" - - 10.10.0.13:8080 10.10.0.14:45156 - -
[2025-05-10T16:55:01.036Z] "GET /items HTTP/1.1" 200 - via_upstream - "-" 0 502 13 13 "-" "beegoServer" "7a15fa50-b200-9b26-936d-0e7f90c4d4ef" "catalog.istioinaction:80" "10.10.0.12:3000" outbound|80||catalog.istioinaction.svc.cluster.local 10.10.0.13:34572 10.200.1.74:80 10.10.0.13:48952 - default
[2025-05-10T16:55:01.035Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 15 15 "-" "curl/8.5.0" "7a15fa50-b200-9b26-936d-0e7f90c4d4ef" "webapp.istioinaction" "10.10.0.13:8080" inbound|8080|| 127.0.0.6:58493 10.10.0.13:8080 10.10.0.14:55804 - default
[2025-05-10T16:55:01.245Z] "GET /items HTTP/1.1" 200 - via_upstream - "-" 0 502 2 2 "-" "beegoServer" "75888a59-4f76-9e3e-a19b-a724cdc1a791" "catalog.istioinaction:80" "10.10.0.12:3000" outbound|80||catalog.istioinaction.svc.cluster.local 10.10.0.13:34572 10.200.1.74:80 10.10.0.13:48964 - default
[2025-05-10T16:55:01.244Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 3 3 "-" "curl/8.5.0" "75888a59-4f76-9e3e-a19b-a724cdc1a791" "webapp.istioinaction" "10.10.0.13:8080" inbound|8080|| 127.0.0.6:40727 10.10.0.13:8080 10.10.0.14:55820 - default
[2025-05-10T16:55:03.386Z] "GET /items HTTP/1.1" 200 - via_upstream - "-" 0 502 2 1 "-" "beegoServer" "58bb5eaa-ec9d-9d5c-8ba4-73e734cb358e" "catalog.istioinaction:80" "10.10.0.12:3000" outbound|80||catalog.istioinaction.svc.cluster.local 10.10.0.13:34572 10.200.1.74:80 10.10.0.13:41666 - default
[2025-05-10T16:55:03.384Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 3 3 "-" "curl/8.5.0" "58bb5eaa-ec9d-9d5c-8ba4-73e734cb358e" "webapp.istioinaction" "10.10.0.13:8080" inbound|8080|| 127.0.0.6:40727 10.10.0.13:8080 10.10.0.14:36910 - default
```


한번 catalog 쪽으로 요청을 보내보자. 


```bash
kubectl exec deploy/sleep -c sleep -- curl -s http://catalog.istioinaction/api/items -o /dev/null -w "%{http_code}\n"n"
000
command terminated with exit code 56
```


당연하게 webapp을 제외하고는 mTLS 통신이 아니라면 막히는 모습이다.


### 인증서 확인


각 워크로드의 인증서를 실제로 확인해보자. 인증서는 위에서 언급했듯 istio-proxy가 istiod에 요청하여 발급받으며, istio-proxy 내부에 존재한다.


```bash
kubectl -n istioinaction exec deploy/webapp -c istio-proxy -- ls -l /var/run/secrets/istio/root-cert.pem
lrwxrwxrwx. 1 root root 20 May 10 16:30 /var/run/secrets/istio/root-cert.pem -> ..data/root-cert.pem
```


아래의 명령어로 인증서에 대한 세부 정보를 확인할 수 있다.


```bash
kubectl -n istioinaction exec deploy/webapp -c istio-proxy \
  -- openssl s_client -showcerts \
  -connect catalog.istioinaction.svc.cluster.local:80 \
  -CAfile /var/run/secrets/istio/root-cert.pem | \
  openssl x509 -in /dev/stdin -text -noout
  
...
depth=1 O = cluster.local
verify return:1
depth=0 
verify return:1
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            f9:f5:b3:20:24:fe:d7:5d:63:81:48:44:88:0f:39:20
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: O=cluster.local
        Validity
            Not Before: May 10 16:28:54 2025 GMT
80FB52A3AC7F0000:error:0A00045C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required:../ssl/record/rec_layer_s3.c:1584:SSL alert number 116
            Not After : May 11 16:30:54 2025 GMT
        Subject: 
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:e6:fd:74:af:16:ee:1e:21:74:ea:34:d7:2b:c7:
                    1d:8a:b8:7a:50:22:5a:fb:67:95:0e:71:14:f4:18:
                    20:57:50:03:95:8d:5b:f4:05:5f:a5:c5:63:e7:8f:
                    39:c8:b3:94:c9:68:09:47:f7:ab:3e:9e:78:1c:a8:
                    03:bf:06:41:03:9e:bd:72:b3:4f:e4:05:b6:76:f3:
                    44:6e:e7:5c:e8:2e:34:6a:75:b2:c5:7e:6d:df:a8:
                    21:95:fa:da:f0:b2:8e:1c:89:a1:6a:f4:f1:24:5b:
                    d4:33:1e:a8:79:7a:0a:b4:15:cf:fe:8d:39:e2:42:
                    1b:d8:d1:9a:1a:c5:37:3b:a0:fa:c6:d4:e6:c9:30:
                    95:c6:9a:f3:00:87:47:eb:52:8c:47:f4:c5:4e:f7:
                    8a:2d:ff:ff:12:35:66:e0:96:14:f6:74:75:be:d9:
                    e4:45:45:09:35:2b:fc:3e:19:35:19:2e:46:2d:a0:
                    b8:52:a7:02:d9:2f:1f:ee:bd:03:f6:21:ef:08:47:
                    8c:a8:38:b0:9b:cd:29:e5:bc:e0:d7:0a:45:13:db:
                    72:c5:f9:b4:34:8f:25:e2:82:de:dd:fa:ec:ae:4b:
                    10:95:9f:6e:87:93:1b:b4:f4:5b:d6:af:ae:a2:fa:
                    fe:07:a0:ba:ad:e9:b9:97:8f:85:74:13:29:85:01:
                    65:2d
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: 
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Authority Key Identifier: 
                52:B0:C5:7A:2F:A8:52:70:5E:15:B2:23:1D:19:32:89:B4:FD:50:6B
            X509v3 Subject Alternative Name: critical
                URI:spiffe://cluster.local/ns/istioinaction/sa/catalog
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        63:6f:bf:77:3c:3c:41:90:fd:15:27:88:d2:f4:50:e4:e4:64:
        52:e3:2b:99:8e:39:1e:7c:a2:46:cb:ae:69:87:fb:69:f4:d4:
        2b:86:63:7c:c4:ef:56:1e:15:10:89:08:41:d8:1c:10:e1:11:
        26:ad:d9:e7:c4:7c:aa:a2:9e:e8:4f:c3:4a:af:a7:5c:09:86:
        b5:0d:72:c4:fb:2b:be:5b:c9:ac:c6:2a:c2:11:bc:6b:06:ee:
        0d:92:8a:81:0b:7e:54:1d:cd:20:f0:97:c3:c3:28:8f:8a:5c:
        ba:d8:fa:9c:e8:1a:18:c8:c0:eb:53:41:66:db:22:bb:d1:b9:
        00:cb:15:ea:c5:97:22:eb:40:ba:17:13:78:ce:3c:eb:1b:c2:
        b5:e7:9a:e2:b7:a1:23:6f:65:a4:45:90:33:8a:e6:24:1e:4a:
        94:eb:bb:12:3a:04:3b:01:7e:9d:28:61:f6:cb:90:2f:fb:0c:
        00:34:aa:a4:61:63:c9:12:a0:59:b2:6d:ae:05:98:09:9c:9f:
        28:a8:20:c1:f3:08:56:f3:37:2a:96:ab:a5:88:18:9f:f4:88:
        fa:1d:09:cd:3b:55:40:fa:1a:45:60:de:44:b2:79:b0:78:94:
        8c:92:c0:ec:98:84:61:8a:e5:16:f4:ec:f6:ec:d5:15:1c:2a:
        ef:c8:4d:cc
command terminated with exit code 1
```


인증서 유효기간이 1 day 정도로 설정되어있는 것을 볼 수 있다.


```bash
            Not Before: May 10 16:28:54 2025 GMT
80FB52A3AC7F0000:error:0A00045C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required:../ssl/record/rec_layer_s3.c:1584:SSL alert number 116
            Not After : May 11 16:30:54 2025 GMT
```


## **RequestAuthentication**


RequestAuthentication는 JWT를 기반으로 유효한 대상인지 검증하는 것이 가능하다. 자세한 확인 Flow는 아래와 같다.


![image.png](/assets/img/post/istio%20Security(인증)/1.png)


### 실습 환경


이전에 사용했던 리소스를 정리한다.


```bash
kubectl delete virtualservice,deployment,service,\
destinationrule,gateway,peerauthentication,authorizationpolicy --all -n istioinaction

kubectl delete peerauthentication,authorizationpolicy -n istio-system --all
```


여기서 사용할 webapp, catalog 리소스만 배포한다.


```bash
kubectl apply -f services/catalog/kubernetes/catalog.yaml -n istioinaction
kubectl apply -f services/webapp/kubernetes/webapp.yaml -n istioinaction
kubectl apply -f ch9/enduser/ingress-gw-for-webapp.yaml -n istioinaction

```


### 리소스 형식

- scope
	- namespace
	- selector: 적용할 워크로드
- jwt: jwt 인증에 필요한 관련 정보를 넣어둔다.

```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-token-request-authn"
  namespace: istio-system # 적용할 네임스페이스
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  jwtRules:
  - issuer: "auth@istioinaction.io" # 발급자 Expected issuer
    jwks: | # 특정 JWKS로 검증
      { "keys":[ {"e":"AQAB","kid":"CU-ADJJEbH9bXl0tpsQWYuo4EwlkxFUHbeJ4ckkakCM","kty":"RSA","n":"zl9VRDbmVvyXNdyoGJ5uhuTSRA2653KHEi3XqITfJISvedYHVNGoZZxUCoiSEumxqrPY_Du7IMKzmT4bAuPnEalbY8rafuJNXnxVmqjTrQovPIerkGW5h59iUXIz6vCznO7F61RvJsUEyw5X291-3Z3r-9RcQD9sYy7-8fTNmcXcdG_nNgYCnduZUJ3vFVhmQCwHFG1idwni8PJo9NH6aTZ3mN730S6Y1g_lJfObju7lwYWT8j2Sjrwt6EES55oGimkZHzktKjDYjRx1rN4dJ5PR5zhlQ4kORWg1PtllWy1s5TSpOUv84OPjEohEoOWH0-g238zIOYA83gozgbJfmQ"}]}
      
```


### JWT 인증 


위에서 예시로 확인한 `RequestAuthentication`를 배포한다.


```bash
apiVersion: "security.istio.io/v1beta1"
kind: "RequestAuthentication"
metadata:
  name: "jwt-token-request-authn"
  namespace: istio-system # 적용할 네임스페이스
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  jwtRules:
  - issuer: "auth@istioinaction.io" # 발급자 Expected issuer
    jwks: | # 특정 JWKS로 검증
      { "keys":[ {"e":"AQAB","kid":"CU-ADJJEbH9bXl0tpsQWYuo4EwlkxFUHbeJ4ckkakCM","kty":"RSA","n":"zl9VRDbmVvyXNdyoGJ5uhuTSRA2653KHEi3XqITfJISvedYHVNGoZZxUCoiSEumxqrPY_Du7IMKzmT4bAuPnEalbY8rafuJNXnxVmqjTrQovPIerkGW5h59iUXIz6vCznO7F61RvJsUEyw5X291-3Z3r-9RcQD9sYy7-8fTNmcXcdG_nNgYCnduZUJ3vFVhmQCwHFG1idwni8PJo9NH6aTZ3mN730S6Y1g_lJfObju7lwYWT8j2Sjrwt6EES55oGimkZHzktKjDYjRx1rN4dJ5PR5zhlQ4kORWg1PtllWy1s5TSpOUv84OPjEohEoOWH0-g238zIOYA83gozgbJfmQ"}]}
      
```


실습에서 사용할 JWT 파일기반으로 배포를 진행했고, 우리의 JWT는 아래의 경로에서 확인할 수 있다.


```bash
cat ch9/enduser/user.jwt
eyJhbGciOiJSUzI1NiIsImtpZCI6IkNVLUFESkpFYkg5YlhsMHRwc1FXWXVvNEV3bGt4RlVIYmVKNGNra2FrQ00iLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ3NDUxNDUwMzgsImdyb3VwIjoidXNlciIsImlhdCI6MTU5MTU0NTAzOCwiaXNzIjoiYXV0aEBpc3Rpb2luYWN0aW9uLmlvIiwic3ViIjoiOWI3OTJiNTYtN2RmYS00ZTRiLWE4M2YtZTIwNjc5MTE1ZDc5In0.jNDoRx7SNm8b1xMmPaOEMVgwdnTmXJwD5jjCH9wcGsLisbZGcR6chkirWy1BVzYEQDTf8pDJpY2C3H-aXN3IlAcQ1UqVe5lShIjCMIFTthat3OuNgu-a91csGz6qtQITxsOpMcBinlTYRsUOICcD7UZcLugxK4bpOECohHoEhuASHzlH-FYESDB-JYrxmwXj4xoZ_jIsdpuqz_VYhWp8e0phDNJbB6AHOI3m7OHCsGNcw9Z0cks1cJrgB8JNjRApr9XTNBoEC564PX2ZdzciI9BHoOFAKx4mWWEqW08LDMSZIN5Ui9ppwReSV2ncQOazdStS65T43bZJwgJiIocSCg
```


이제 이 내용을 기반으로 요청을 보내자.도메인 설정(`/etc/hosts`)도 필요하다.


```bash
curl -H "Authorization: Bearer $USER_TOKEN"      -sSl -o /dev/null -w "%{http_code}" webapp.istioinaction.io:30000/api/catalog
200
```


#### 유효하지 않은 JWT 시도


일부로 유효하지 않은 JWT로 다시 트래픽 요청을 시도해보자.


```bash
cat ch9/enduser/not-configured-issuer.jwt
eyJhbGciOiJSUzI1NiIsImtpZCI6IkNVLUFESkpFYkg5YlhsMHRwc1FXWXVvNEV3bGt4RlVIYmVKNGNra2FrQ00iLCJ0eXAiOiJKV1QifQ.eyJleHAiOjQ3NDUxNTE1NDgsImdyb3VwIjoidXNlciIsImlhdCI6MTU5MTU1MTU0OCwiaXNzIjoib2xkLWF1dGhAaXN0aW9pbmFjdGlvbi5pbyIsInN1YiI6Ijc5ZDc1MDZjLWI2MTctNDZkMS1iYzFmLWY1MTFiNWQzMGFiMCJ9.eUEbrJ3Gr4F5eViMlLsIGcD6UIId6tH6u5vLN_IzPnwpSSp6vy6knVgC1GHsWPWwnEhcPHz1TlQz8E3O6F7oVyNhMTJyniaXtVyByvgAVCbeaOYVRnm1aSWwjFt5IfJJcbk21BWbPfE12Hfbo03sRq1hI1iEcn4nbtoh8tjj_G4r8gwiKVlkA3g5bFkwiSEmZQe2cumzgdtNu4XzU5ghl6cdFyzYD5x3750uy_bfduaQokCVymQq3P-dUPnz7_5-ZOj-3SRb3yHbmvlAnyQgTgIlQc3J-anGnsqec33lhVh5RdNuxKj9J14a-ub9ysjzUvcXh1expDqNxR33BaQnpQ
```


아래와 같은 요청을 날리면 `401` 응답이온다.


```bash
WRONG_ISSUER=$(< ch9/enduser/not-configured-issuer.jwt)
curl -H "Authorization: Bearer $WRONG_ISSUER" \
     -sSl -o /dev/null -w "%{http_code}" webapp.istioinaction.io:30000/api/catalog
401
```


다시 로그를 확인해보면, 아래와 같이 유효하지 않은 사용자 “jwt_authn_access_denied” 접근으로 차단되는 것을 볼 수 있다.


```bash
[2025-05-10T17:30:19.723Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 15 14 "172.18.0.1" "curl/8.5.0" "827128d8-a320-9f74-8b92-51fad30ebc4b" "webapp.istioinaction.io:30000" "10.10.0.16:8080" outbound|80||webapp.istioinaction.svc.cluster.local 10.10.0.6:38450 10.10.0.6:8080 172.18.0.1:41850 - -
[2025-05-10T17:35:04.324Z] "GET /api/catalog HTTP/1.1" 401 - jwt_authn_access_denied{Jwt_issuer_is_not_configured} - "-" 0 28 0 - "172.18.0.1" "curl/8.5.0" "b6d14c77-3798-9fe6-bb34-fefc7d800df2" "webapp.istioinaction.io:30000" "-" outbound|80||webapp.istioinaction.svc.cluster.local - 10.10.0.6:8080 172.18.0.1:35358 - -
```


#### 토큰 없이 요청


이럴 경우 istio에서는 클러스터 내부 요청으로 인식하여, 해당 요청을 Allow 한다.


```bash
curl -sSl -o /dev/null -w "%{http_code}" webapp.istioinaction.io:30000/api/catalog
200
```


로그를 확인해봐도, 문제없이 받아준다.


```bash
[2025-05-10T17:36:57.305Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 5 5 "172.18.0.1" "curl/8.5.0" "569c0218-e9a1-9638-96de-74824802590a" "webapp.istioinaction.io:30000" "10.10.0.16:8080" outbound|80||webapp.istioinaction.svc.cluster.local 10.10.0.6:38434 10.10.0.6:8080 172.18.0.1:57702 - -
[2025-05-10T17:36:59.060Z] "GET /api/catalog HTTP/1.1" 200 - via_upstream - "-" 0 357 5 5 "172.18.0.1" "curl/8.5.0" "c694346d-6720-958f-a2dc-2a06b4969976" "webapp.istioinaction.io:30000" "10.10.0.16:8080" outbound|80||webapp.istioinaction.svc.cluster.local 10.10.0.6:54906 10.10.0.6:8080 172.18.0.1:57716 - 
```


결국 토큰이 없는 요청을 거부하려면 추가 작업이 필요하다. 아직 다루지 않은 `AuthorizationPolicy` 리소스를 통해 ingress gateway로 향하는 트래픽 중에 별도 토큰 값이 없는 주체는 거부하도록 설정하면 된다.


아래와 같이 `AuthorizationPolicy`를 설정하면 된다.


```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: app-gw-requires-jwt
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"] # 요청 주체에 값이 없는 source는 모두 해당된다
    to:
    - operation:
        hosts: ["webapp.istioinaction.io:30000"] 
```


### 마치며


이번 포스팅에서는 istio 트래픽 인증부분을 살펴봤다. 다음 포스팅에서는 인가부분에 대해 자세히 살펴볼 예정이다.

