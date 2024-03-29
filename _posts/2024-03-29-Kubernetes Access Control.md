---
layout: post
title: Kubernetes Access Control
date: 2024-03-29 09:00 +0900 
description: 쿠버네티스 API 접근 제어 체계 정리하기
category: [Kubernetes, Security] 
tags: [Kubernetes, kube-apiserver, 인증, 인가, Admission-Controllers] 
pin: false
math: true
mermaid: true
---


쿠버네티스 API 접근 제어 체계 정리하기
<!--more-->


## 인증/인가


쿠버네티스는 모든 행동이 API(kube-apiserver)를 통해 진행된다. API를 사용할 때는 인증이 필요하다. 인증 방법에 대해서는 나중에 살펴보고, 여기서는 API를 통해 어떤 과정으로 요청이 허가되는지 알아본다.


### 인증과정


> 💡 **순서**
> 1. Client API 요청(ex. Kubectl)  
> 2. 인증(Authentication)  
> 3. 인가(Authorization)  
> 4. Admission Controllers  
> 5. 명령실행(ex Create Pod)


#### 인증


인증에서는 해당 클라이언트가 적합한 대상인지 확인한다. kubectl로 명령을 진행하면, ./kube/config 에 있는 `certificate-authority-data`를 이용하여 판단한다. 


#### 인가


인증이 완료되면, 해당 유저에 대한 Role을 확인하여 권한이 있는지 확인한다. Role에는 Resource(리소스)와 Verbs(연산)가 존재한다. 만약 내가 파드를 생성하려하면, 나의 Role의 리소스에는 파드가 존재해야 하며, 연산에도 `Create`를 포함해야 한다.


#### Admission Controllers


인가까지 마무리되면, Admission Controllers에 의해 확인된다. 인가는 큰 범위의 조건이면 이제 세부조건을 판단한다. 컨트롤러에 의해 판단도 진행되지만, 판단이 아닌 특정 동작을 취할 수도 있다. 예를 들어, 내가 ‘A’라는 네임스페이스에 리소스를 생성할 때, 네임스페이스 ‘A’가 존재하지 않는다면 자동으로 생성하도록 정할 수 있다.


Admission Controllers는 세부조건을 확인하는 ‘validating’과 ‘mutating’이 존재한다. validating은 세부조건이 유효한지 확인하며, mutating은 바로 위의 예시와 같이 요청과 관련된 객체를 수정한다.


> 💡 **예시  
> 인증:** 놀이공원에 입장할 때 나의 입장권과 신분증을 보여주는 것  
> **인가:** 이후 놀이기구에 입장할 때, 나의 티켓이 해당 놀이기구를 탈 수 있는 권한이 있는지 파악하는 것  
> ‘**Admission Controllers’:** 탑승 전에, 나의 키와 몸무게가 규정에 맞지 않는다면 탑승하지 못한다. 이와 같은 세부 조건을 ‘**Admission Controllers’**가 판단한다고 생각하면 된다.


세부조건 사항으로는 아래와 같은 것들이 있다. [공식문서](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook)에서 자세한 내용을 확인할 수 있다.

- validating
	- ImagePolicyWebhook
	- NamespaceExists
- mutating
	- NamespaceAutoProvision
	- DefaultStorageClass(Storage Class가 없으면, Default로 설정한다.)
- validating & mutating
	- AlwaysPullImages

#### 명령 수행


Admission Controllers까지 진행되면, 명령이 수행된다. ex) Create Pod


명령에 대한 Object는 Admission Controllers 단계에서 초기와는 다르게 수정될 수 있다. (만약, mutating이 적용된다면)


### 현재 **Authorization**모드


주로 RBAC으로 인증/인가를 처리하지만, 아닐수도 있다. 그럴 때는 kube-apiserver 파드를 확인한다.


```bash
$ kubectl describe pod kube-apiserver-controlplane -n kube-system | grep mode
      --authorization-mode=Node,RBAC
```


### 권한 확인


유저인 경우: `{명령어} --as {User}`


```bash
$ kubectl get pods --as dev-user
```


### 권한만드는 법


[RBAC](https://www.notion.so/7530019ba1594a0da4b7786645a1de3e) 방식으로 권한을 만드는 예시이다. 네임스페이스 범위안에 있는 리소스는 Role로 진행되며, 그외 Cluster 범위는 ClusterRole로 권한을 제어할 수 있다.


권한을 생성하고 RoleBinding, ClusterRoleBinding 을 통해 특정 주체와 결합해야 한다.

- Role/ClusterRole

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---
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

- Role/ClusterRoleBinding

```yaml
---
apiVersion: rbac.authorization.k8s.io/v1
# This role binding allows "jane" to read pods in the "default" namespace.
# You need to already have a Role named "pod-reader" in that namespace.
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
  # "roleRef" specifies the binding to a Role / ClusterRole
  kind: Role #this must be Role or ClusterRole
  name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
---
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

