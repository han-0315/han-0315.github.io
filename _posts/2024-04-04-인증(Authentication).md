---
layout: post
title: 쿠버네티스 인증(Authentication) 정리
date: 2024-04-04 23:00 +0900 
description: 쿠버네티스 인증 방법 정리
category: [Kubernetes, Security] 
tags: [Kubernetes, kube-apiserver, 인증, webhook, openi-connect, tls, X509, service account] 
pin: false
math: true
mermaid: true
---


쿠버네티스 인증(Authentication) 정리
<!--more-->


### 시작하기에 앞서


일반 웹사이트의 경우 인증 정보를 전달받고, 내부 데이터베이스의 정보 중 일치하는 정보가 있다면 인증을 허가한다. 이와 다르게 **쿠버네티스는 유저의 정보를 관리하는 저장소가 별도로 없다.** 


쿠버네티스의 인증 체계는 X509, Tokens, Proxy 등 다양한 외부 시스템에 의존한다. 이 방식은 특별한 방식이 아닌 웹 서버의 인증 방식과 같으며, 쿠버네티스 인증 체계를 학습하며 간단한 웹사이트의 인증 방식을 살펴볼 수 있다.


현재(2024/04/03) 공식문서에 나와 있는 인증 방식은 아래와 같다.

- X.509 Certificate
- Static token file
- Bootstrap tokens
- Service account tokens
- OpenID Connect Tokens
- Webhook Token Authentication
- Authenticating Proxy

## CA 인증 방식


### X.509 Certificate


HTTPS 인증에 사용되는 기술이며, TLS 핸드쉐이크와 같은 방식으로 인증을 진행한다. 일반적인 TLS 핸드쉐이크는 다음과 같은 방식으로 이뤄진다.

1. **사용자:** 웹사이트로 요청을 보냄
2. **서버:** 자신의 Public Key를 CA 인증서로 암호화하여 사용자에게 응답
3. **사용자:** CA 인증서 발행기관에 CA를 검증 후 Public Key 키를 수신
4. **사용자:** 자신의 Pulbic Key(예비 마스터키)를 서버의 Public Key로 암호화하여 전달
5. **서버:** 자신의 Private Key로 메시지를 복호화하고, 건네받은 마스터키를 통해 세션 키 생성
6. 세션 키를 통해 암호화된 메시지를 서로 교환하며, TLS 인증이 정상적으로 작동되었음을 파악함

#### 작동 방식


쿠버네티스에서는 X.509 인증 방식도 위의 과정과 유사하다. 우선 쿠버네티스에서는 RootCA가 필요하다. **Root CA에서 Server CA와 Client CA를 발급한다.** 서버는 api-server에서 사용하고, 클라이언트는 사용자들이 사용하는 인증서이다.

1. 사용자가 k8s-api-server에 인증서를 요청한다.
2. k8s-api-server는 자신의 Public 인증서를 건네준다.
3. 사용자가 건네받은 인증서가 유효한지(정상적인 api-server가 맞는지) Client CA를 통해 확인한다.
4. 확인이 끝나면, 사용자가 자신의 Public 인증서를 api-server에게 건네준다.
5. api-server는 Server CA를 통해 유효한 정보가 맞는지 확인한다.
6. 5번까지 모두 정상적으로 처리되면, k8s-api-server는 사용자가 요청한 작업을 수행한다.

![Untitled.png](/assets/img/post/인증(Authentication)/1.png)


출처: [https://medium.com/swlh/how-we-effectively-managed-access-to-our-kubernetes-cluster-38821cf24d57](https://medium.com/swlh/how-we-effectively-managed-access-to-our-kubernetes-cluster-38821cf24d57)


#### 단점


X509를 이용한 방식은 한번 Private Key와 인증서가 노출되면 Root CA를 교체하지 않는 이상 노출을 막을 방법이 없다는 단점이 있다.


## HTTP 인증 방식


### Static token file


파일을 토대로 인증이 가능하다. 

1. 자격증명 파일을 준비한다. 파일의 형식은  **`token,user,uid,"group1,group2,group3"`** 이다.
2. 컨트롤 노드에서 kube-apiserver.yaml 파일을 수정하여 `--token-auth-file=<파일 경로>` 옵션을 추가한다.
3. 이제 사용자는 (1)에서 준비한 token을 HTTP 헤더에 추가한다.

	```bash
	Authorization: Bearer 31ada4fd-adec-460c-809a-9e56ceb75269
	```


### Bootstrap tokens


새 클러스터를 생성하거나 기존 클러스터에 새 노드를 클러스터에 추가할 때 사용하기 위한 간단한 무기명 토큰이다. 동적으로 관리가 필요하기에 kube-system 네임스페이스에 생성한다. 만료되면 TokenCleaner 컨트롤러에 의해 제거된다. 


아래는 Bootstrap tokens의 예시 YAML 파일이다.


```yaml
apiVersion: v1
kind: Secret
metadata:
  # Name MUST be of form "bootstrap-token-<token id>"
  name: bootstrap-token-07401b
  namespace: kube-system

# Type MUST be 'bootstrap.kubernetes.io/token'
type: bootstrap.kubernetes.io/token
stringData:
  # Human readable description. Optional.
  description: "The default bootstrap token generated by 'kubeadm init'."

  # Token ID and secret. Required.
  token-id: 07401b
  token-secret: f395accd246ae52d

  # Expiration. Optional.
  expiration: 2017-03-10T03:22:11Z

  # Allowed usages.
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"

  # Extra groups to authenticate the token as. Must start with "system:bootstrappers:"
  auth-extra-groups: system:bootstrappers:worker,system:bootstrappers:ingress
```


이제 요청을 보낼 때, 헤더에 아래의 부분을 추가하면 인증받을 수 있다.


```bash
Authorization: Bearer <token_id>.<token-secret>
```


참고자료: [https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/)


### Service account tokens


kubectl을 이용하여 Service Account에 대한 토큰을 발급할 수 있다. 


```bash
$kubectl create token default
eyJhbGciOiJSUzI1NiIsImtpZCI6ImRRdXZzd2dCV...
```


발급받은 토큰을 통해 인증받을 수 있다.


```bash
curl -k --header "Authorization: Bearer {위의 토큰의 값}" https://{kubernetes-api-server}/api
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "192.168.49.2:8443"
    }
  ]
}%
```


## **OIDC(OpenID connect)** 방식


OIDC(OpenID Connect)는 OpenID Foundation에서 정의한 개방형 Authentication 표준이며, 컨슈머 애플리케이션의 SSO를 목적으로 JSON 형식으로 개발했다. 


![openid-connect-digram-blue.png](/assets/img/post/인증(Authentication)/2.png)


출처: OpenID([https://openid.net/developers/how-connect-works/](https://openid.net/developers/how-connect-works/))


우리가 어떤 웹 서비스에 회원가입할 때, 구글이나 네이버, 카카오 등을 통해 빠르게 회원 가입할 수 있다. 이때 OpenID Connect가 사용된다. 예를 들어 어떤 사용자가 A 웹사이트를 회원 가입할 때 구글로 인증을 선택하고, 구글의 로그인을 진행하면 구글에서 웹 서버로 인증 코드를 발급하는 방식이다.


즉, 클라이언트는 외부시스템(IdP)에서 인증을 받으며 토큰이 생성된다. 서버는 이 토큰을 확인하여, 인증을 진행한다.


### OpenID Connect Tokens


쿠버네티스에서도 똑같은 원리로 작동한다. 우리가 IdP(Identity Provider)에 로그인하여 인증을 진행한다. 인증이 완료되면 토큰을 발급해 준다. 토큰을 통해 api-server에 접근하면 api-server에서 토큰의 유효성을 확인한 뒤 인증이 완료된다.


![Untitled.png](/assets/img/post/인증(Authentication)/3.png)


출처: Kubernetes Docs([https://kubernetes.io/docs/reference/access-authn-authz/authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens))


OpenID Connect를 설정하려면 kube-apiserver.yaml 파일을 수정해야 한다.


```yaml
- --oidc-issuer-url=<OAuth IDP URL>
- --oidc-client-id=<client-id>
- --oidc-ca-file=/etc/kubernetes/pki/<idp-ca.crt>
```


실습과 관련해서, 커피고래님이 관련한 실습 내용을 자세하게 설명해주셨다. [커피고래님 블로그](https://coffeewhale.com/kubernetes/authentication/oidc/2020/05/04/auth03/)를 참고하는 것을 추천한다.


## Webhook


웹훅은 API간에 이벤트 기반 통신을 지원하는 HTTP 콜백 기능이다. 이벤트가 발생했을 때, 특정 작업을 트리거할 수 있다.


### Webhook Token Authentication


쿠버네티스 API 서버에도 웹훅을 구현할 수 있는 방법이 존재한다. 내가 인증 정보(Token)를 보내면 웹훅을 통해 외부 시스템에서 인증을 확인할 수 있다.


웹훅을 사용하려면 kube-apiserver.yaml 파일을 수정해야 한다.


```yaml
...
    - --authentication-token-webhook-config-file=<webhook 관련 YAML 파일>
```


공식문서에 나와있는 webhook YAML 파일 예시


```yaml
# Kubernetes API version
apiVersion: v1
# kind of the API object
kind: Config
# clusters refers to the remote service.
clusters:
  - name: name-of-remote-authz-service
    cluster:
      # CA for verifying the remote service.
      certificate-authority: /path/to/ca.pem
      # URL of remote service to query. Must use 'https'. May not include parameters.
      server: https://authz.example.com/authorize

# users refers to the API Server's webhook configuration.
users:
  - name: name-of-api-server
    user:
      client-certificate: /path/to/cert.pem # cert for the webhook plugin to use
      client-key: /path/to/key.pem          # key matching the cert

# kubeconfig files require a context. Provide one for the API Server.
current-context: webhook
contexts:
- context:
    cluster: name-of-remote-authz-service
    user: name-of-api-server
  name: webhook
```


## Proxy


프록시 서버를 통해 인증을 진행한다. 외부 시스템을 통해 인증을 진행한다는 것은 웹훅과 유사하나, 쿠버네티스 뒷단이 아닌 앞단에서 인증을 진행한다는 것이 차이점이다. 즉, 웹훅은 **사용자 > k8s > 외부시스템**이고 프록시는 **사용자 > 외부 시스템 > k8s**이다.


### Authenticating Proxy


우리가 요청을 보내면, 프록시 서버에서 인증을 진행하고 Header를 통해 사용자의 정보를 kube-api-server로 전달한다. 단순 인증 여부가 아닌 사용자의 정보를 보내는 이유는 인증 단계가 끝나면 권한을 확인하는 인가의 단계때문이다. 헤더를 통해 사용자의 정보를 보내면 쿠버네티스는 해당 정보를 가지고 인가의 단계를 진행하고 최종적으로 요청한 명령을 수행한다.


#### 실습


이번에도 [커피고래님의 블로그](https://coffeewhale.com/kubernetes/authentication/proxy/2020/05/06/auth05/)를 통해 실습을 진행할 수 있었다. kube-apiserver.yaml 파일을 수정하고, 프록시 서버를 띄우면 구축이 완료된다.


```yaml
    # API 서버 인증서 설정
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    # Client CA 설정
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    # Proxy 헤더 설정
    - --requestheader-username-headers=X-Remote-User
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
```


## 참고자료


[https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)


[https://kubernetes.io/docs/reference/access-authn-authz/webhook/](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)


[https://medium.com/@extio/kubernetes-authentication-with-oidc-simplifying-identity-management-c56ede8f2dec](https://medium.com/@extio/kubernetes-authentication-with-oidc-simplifying-identity-management-c56ede8f2dec)


[https://coffeewhale.com/kubernetes/authentication/proxy/2020/05/06/auth05/](https://coffeewhale.com/kubernetes/authentication/proxy/2020/05/06/auth05/)


[https://www.samsungsds.com/kr/insights/oidc.html](https://www.samsungsds.com/kr/insights/oidc.html)


[https://openid.net/developers/how-connect-works/](https://openid.net/developers/how-connect-works/)


[https://medium.com/swlh/how-we-effectively-managed-access-to-our-kubernetes-cluster-38821cf24d57](https://medium.com/swlh/how-we-effectively-managed-access-to-our-kubernetes-cluster-38821cf24d57)

