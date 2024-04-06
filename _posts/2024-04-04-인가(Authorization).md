---
layout: post
title: 쿠버네티스 인가(Authorization)정리
date: 2024-04-04 09:00 +0900 
description: 쿠버네티스 인가(Authorization)정리하기
category: [Kubernetes, Security] 
tags: [Kubernetes, kube-apiserver, 인가, Node, ABAC, RBAC, Webhook] 
pin: false
math: true
mermaid: true
---


쿠버네티스 API 접근 제어 체계 정리하기
<!--more-->


### 인가


쿠버네티스의 접근 통제 방식(**인증 > 인가 > Admission Control**) 중 인가(Authorization)에 대해 알아본다. ‘인가’ 단계까지 왔다는 것은 인증은 완료된 것이다. 우리는 이제 해당 사용자가 권한이 있는지 확인한다.


인가 방식으로는 아래의 4개가 존재한다.

- [Role Based Access Control](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Attribute Based Access Control](https://kubernetes.io/docs/reference/access-authn-authz/abac/)
- [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/)
- [Webhook Authorization](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)

하나씩 간략히 살펴보고, 주로 사용하는 RBAC은 자세히 살펴본다.


## **NODE**


Node Authorization은 kubelet을 위한 인가 방식이다. 사용자를 위한 방법이 아니다. kubelet은 각 노드에서 실행되는 에이전트로 파드와 컨테이너를 관리하는 임무를 받았다. 파드와 컨테이너를 관리하기 위해서는 api-server를 통해 파드 관련 정보를 얻거나 상태를 변경해야 한다. Node Authorization에서는 관련된 연산을 허용한다. 


> 💡 **연산목록**  
> - Read operations  
>   
> - Write operations  
>   
> - Auth-related operations

	- Read operations
		- services
		- endpoints
		- nodes
		- pods
		- secrets, configmaps, persistent volume claims and persistent volumes related to pods bound to the kubelet's node
	- Write operations
		- nodes and node status (enable the `NodeRestriction` admission plugin to limit a kubelet to modify its own node)
		- pods and pod status (enable the `NodeRestriction` admission plugin to limit a kubelet to modify pods bound to itself)
		- events
	- Auth-related operations
		- read/write access to the [CertificateSigningRequests API](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/) for TLS bootstrapping
		- the ability to create TokenReviews and SubjectAccessReviews for delegated authentication/authorization checks

Kubelet은 Node authorizer에 의해 권한을 부여받기 위해서는 `Group: system:nodes`, `User: system:node:<node_name>` 형식으로 식별되야 한다. 안전한 통신을 위한 TLS 인증서이 필요하다.


## ABAC 


Attribute-based access control의 약자로 속성을 기반으로 접근을 제어한다. 특정 사용자 혹은 리소스에 대해 직접 권한을 부여한다. 이 방식은 각각의 객체에 권한을 직접 설정해야 한다는 단점이 있다. 사용자가 늘어나면, 권한을 관리하는 것이 복잡해진다.


#### 실습


kube-apiserver.yml 파일에 다음과 같은 옵션을 넣어준다.
 `--authorization-policy-file=SOME_FILENAME` , `--authorization-mode=ABAC` 


```bash
# 아래의 api서버의 경로는 예시
$vi /etc/kubernetes/manifests/kube-apiserver.yaml
...
ExecStart=/usr/local/bin/kube-apiserver \\
    --advertise-address=${INTERNAL_IP} \\
    --allow-privileged=true \\
    --apiserver-count=3 \\
    --authorization-mode=ABAC \\ # ABAC 설정 필요
    --bind-address=0.0.0.0 \\
    --enable-swagger-ui=true \\
    --etcd-servers=https://127.0.0.1:2379 \\
    --event-ttl=1h \\
    --runtime-config=api/all \\
    --service-cluster-ip-range=10.32.0.0/24 \\
    --service-node-port-range=30000-32767 \\
    --v=2
		--authorization-policy-file=ABAC_example.json # ABAC 설정 파일 지정 필요
```


다음과 같이 하나의 사용자마다 권한을 부여할 수 있다. 아래의 예시 첫 번째 줄은 ‘alice’ 라는 사용자에게 클러스터에 대한 모든 접근권한을 부여하는 것이다. 아래와 같이 하나의 파일에 여러 줄로 권한을 부여하면 된다.


```json
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "alice",
        "namespace": "*",
        "resource": "*",
        "apiGroup": "*"
    }
},
{
    "apiVersion": "abac.authorization.kubernetes.io/v1beta1",
    "kind": "Policy",
    "spec": {
        "user": "kubelet",
        "namespace": "*",
        "resource": "pods",
        "readonly": true
    }
}
```


## **RBAC** 


역할 기반 접근 제어이다. AWS를 사용한 적이 있다면, IAM 정책을 떠올리면 된다. 하나의 정책에 사용자를 연결하여 관리한다. 권한을 부여하는 방식은 아래와 같다.

1. 역할을 생성한다. (ClusterRole or Role)
2. 바인딩 리소스를 통해 특정 사용자에게 연결한다. (ClusterRoleBinding or RoleBinding)

역할의 종류는 2개 있다. `Role` , `Cluster Role` 두 개의 차이점은 범위(Scope)이다. 네임스페이스 내부에 존재하는 리소스는 `Role`, Cluster Scope인 권한은 `Cluster Role`이다. 


RBAC 모드로 실행하기 위해서는 아래와 같이 kube-apiserver.yaml 파일에서 `--authorization-mode=...,RBAC`로 설정하면 된다.


```bash
$vi /etc/kubernetes/manifests/kube-apiserver.yaml
...
ExecStart=/usr/local/bin/kube-apiserver \\
    --advertise-address=${INTERNAL_IP} \\
    --allow-privileged=true \\
    --apiserver-count=3 \\
    --authorization-mode=RBAC \\ #RBAC
    --bind-address=0.0.0.0 \\
    --enable-swagger-ui=true \\
    --etcd-servers=https://127.0.0.1:2379 \\
    --event-ttl=1h \\
    --runtime-config=api/all \\
    --service-cluster-ip-range=10.32.0.0/24 \\
    --service-node-port-range=30000-32767 \\
```


#### Role


‘default’ 네임스페이스에 존재하는 파드의 조회, 생성, 업데이트 권한을 부여하는 Role이다.


```yaml
apiVersion: rbac.authorization.k8s.io/v1 
kind: Role
metadata:
	name: developer 
	namespace: default
rules:
- apiGroups: [""]
	resources: ["pods"]
	verbs: ["get", "create", "update"]
```


#### ClusterRole


Secrets 리소스에 대해 `get`, `watch`, `list` 연산 권한을 부여하는 ClusterRole 예시이다.


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  # "namespace" omitted since ClusterRoles are not namespaced
  name: secret-reader
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  # objects is "secrets"
  resources: ["secrets"]
  verbs: ["get", "watch", "list"]
```


#### RoleBinding 


```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
# You can specify more than one "subject"
- kind: User
  name: jane # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: developer 
  apiGroup: rbac.authorization.k8s.io
```


#### ClusterRoleBinding


default 네임스페이스 내에서 사용자 jane에 ‘developer’ 역할을 바인딩한다. 이제 jane은 default 네임스페이스 안에서 파드를 조회, 생성, 업데이트할 수 있다.


manager 그룹의 모든 사용자에게 클러스터 안에 있는 `secret` 리소스를 읽을 수 있는 ‘secret-reader’ 역할을 바인딩한다.


```yaml
apiVersion: rbac.authorization.k8s.io/v1
# This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```


## Webhook


웹훅 방식은 인증에서 다룬 내용과 유사하다. 외부 시스템을 통해 권한을 확인한다. kube-apiserver.yml 설정만 차이가 있고, 나머지는 동일하다. 자세한 내용은 [이전 포스팅](https://www.handongbee.com/posts/%EC%9D%B8%EC%A6%9D(Authentication)/#webhook)에서 확인할 수 있다.


```yaml
...
- --authorization-mode=Webhook,... \
- --authorization-webhook-config-file=/path/to/webhook-config.yaml
```


## 다중 인증 모드


**API config file**


여러 가지 모드를 선택할 수 있으며, 인가 방식은 순서대로 진행된다. 예를 들어, 아래와 같이 `Node,RBAC ,Webhook`으로 설정했다면 Node authorizer에 권한을 확인받고, RBAC에서 권한을 확인받은 뒤 Webhook을 통해 권한을 확인받는다. 이 중 어느 하나라도 허가를 받지 못하면, 해당 요청은 거부된다.


```bash
# 아래의 api서버의 경로는 예시
$vi /etc/kubernetes/manifests/kube-apiserver.yaml
...
ExecStart=/usr/local/bin/kube-apiserver \\
    --advertise-address=${INTERNAL_IP} \\
    --allow-privileged=true \\
    --apiserver-count=3 \\
    **--authorization-mode=Node, RBAC, Webhook \\**
    --bind-address=0.0.0.0 \\
    --enable-swagger-ui=true \\
    --etcd-servers=https://127.0.0.1:2379 \\
```


### 권한 확인(kubectl)


권한을 확인하는 명령어는 다음과 같다.


`k -n <namespace name> auth can-i <verbs> <resource> —-as <user or service account>` 


#### 유저 예시


```bash
kubectl auth can-i create deployments --as dev-user --namespace test
```


#### Service Account 예시


```bash
kubectl -n project-hamster auth can-i create secret \
  --as system:serviceaccount:project-hamster:processor
```


## 참고자료


[https://kubernetes.io/docs/reference/access-authn-authz/node/](https://kubernetes.io/docs/reference/access-authn-authz/node/)


[https://kubernetes.io/docs/reference/access-authn-authz/rbac/](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)


[https://kubernetes.io/docs/reference/access-authn-authz/webhook/](https://kubernetes.io/docs/reference/access-authn-authz/webhook/)


[https://yuminlee2.medium.com/kubernetes-authorization-part1-authorization-modes-overview-18538759e2d5#569a](https://yuminlee2.medium.com/kubernetes-authorization-part1-authorization-modes-overview-18538759e2d5#569a)


[https://beta.kodekloud.com/user/courses/cka-certification-course-certified-kubernetes-administrator](https://beta.kodekloud.com/user/courses/cka-certification-course-certified-kubernetes-administrator)

