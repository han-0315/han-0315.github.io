---
layout: post
title: Kubernetes Application 관련 정리
date: 2024-03-27 15:00 +0900 
description: CKAD 공부할 겸, 애플리케이션 관련 내용을 정리한다.
category: [Kubernetes, Application] 
tags: [CKAD, Kubernetes, Application, Pod, Sidecar] 
pin: false
math: true
mermaid: true
---
CKAD 공부할 겸, 애플리케이션 관련 내용을 정리한다.
<!--more-->


CKAD는 CKA와는 다르게 Application과 관련된 내용만 나온다. CKAD의 내용은 CKA와 많이 겹치나, 오히려 영역은 줄어든 느낌이다. 그렇기에 전반적인 내용보다는 Application 관련된 내용 중 헷갈리는 부분이나 주요 부분에 대해 정리한다.


## ARGS vs COMMANDS


### Docker 


공식문서에서 표시된 [Dockerfile](https://docs.docker.com/reference/dockerfile/)에 대한 설명을 확인하면, ENTRYPOIN는 컨테이너가 어떤 실행파일을 실행할지 설정하는 명령어이다. 


반면 CMD는 컨테이너를 실행할 때, 사용되는 명령어다. Dokcerfile에는 하나의 CMD 명령어만 가능하며, 여러 개가 있을 경우 마지막 명령어만 적용된다. (즉, 파라미터와 Dockerfile로 둘다 설정하면 나중 순서인 파라미터 명령어가 실행된다.)


Bash로 예를 들면, ENTRYPOINT는 `/bin/bash` 이고, CMD는 ls 혹은 find 등이 될 수 있다.


### Kubernetes


쿠버네티스에서 파드를 실행시킬 때, 위에서 진행했던 것과 같이 ENTRYPOINT와 CMD를 설정할 수 있다. 다만 용어가 조금 다르다. ENTRYPOINT는 commands로, CMD는 args로 표현된다.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      commands: ["/bin/sh", "-c"]
      args:
        - echo "Hello, Kubernetes!"
      env:
        - name: COLOR
          value: "RED"

```


## 환경변수 관련


아래의 간단한 파드 YAML이 있을 때, 여기서 우리는 환경변수를 가져오는 방법에 대해 알아본다. 아래에서는 가장 기본적인 방법으로 값을 읽어온다. 읽어오는 방법은 직접 명시하는 방법, Configmap을 사용하는 방법, Secret 리소스를 사용하는 방법 총 3가지가 있다.


**[기본 형태]**


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      commands: ["/bin/sh", "-c"]
      args:
        - echo "Hello, Kubernetes!"
      env:
        - name: COLOR
          value: "RED"

```


**[Configmap]**


```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_COLOR: "red"
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      env:
        - name: COLOR
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_COLOR

```


**[Secret]**


```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: app-config
data:
  APP_COLOR: cmVkCg== # base64 encoded value of "red"
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      env:
        - name: COLOR
          valueFrom:
            SecretKeyRef:
              name: app-config
              key: APP_COLOR

```


## Application 수준 Security


### Container


컨테이너 수준에서 실행할 수 있는 권한을 축소함. Run as User를 설정하여, Root 유저가 아닌 별도의 유저로 설정한다. 이럴 경우 네임스페이스에 의해 컨테이너의 PID가 1이 되며, 내가 다른 프로세스와 논리적으로 격리되어 보안수준이 높아진다.


### Pod


파드에서도 컨테이너 단위 수준의 보안을 진행할 수 있다. 


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]

```


### Services Account


파드는 생성되면, 기본적으로 하나의 서비스 계정이 생성된다. 서비스 계정과 연결되는 토큰도 생성되며, 해당 토큰을 통해 파드는 Kubernetes API를 사용할 수 있다.


그래서 만약, 파드가 Services Account Token을 자동으로 생성하는 것을 원하지 않는다면 아래와 같이 설정해주면 된다.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      automountServiceAccountToken: false

```


## 리소스(최소, 최대)


컨테이너 단위로 필요한 리소스를 명시할 수 있다. (최소, 최대 명시)


cpu는 1000m = 1core, memory는 1Mi = 1MB


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: ubuntu
      image: ubuntu:latest
      resources:
        limits:
          memory: "200Mi"
          cpu: "500m"
        requests:
          memory: "100Mi"
          cpu: "200m"

```


## Multi Container Pattern


멀티컨테이너 디자인 패턴에는 Sidecar, Ambassador, Adapter 3가지 종류가 있다. 각각은 메인 컨테이너에 대해 어떤 기능을 수행하는 지에 따라 명칭이 달라진다. 또한, 네트워크와 관련된 역할을 수행해도 범위가 커지면 사이드카라고도 부른다고 한다. ex) envoy proxy


**Sidecar** : **기능 확장**


**Ambassador** : **네트워크 프록시**


**Adapter** : **출력, 형식 변환**


### Sidecar 컨테이너


쿠버네티스 공식문서를 확인하면, 아래와 같이 사이드카 컨테이너를 정의한다. 


> 사이드카 컨테이너는 동일한 파드 내에서 메인 애플리케이션 컨테이너와 함께 실행되는 보조 컨테이너이다. 메인 애플리케이션의 동작을 변경하지 않고, 추가적인 서비스를 수행한다. 주로 로깅, 모니터링, 데이터 동기화를 제공하며 컨테이너의 기능을 향상 및 확장하는데 사용한다.


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
```


## Pod probes(상태 검사)


우리는 파드 수준에서 현재 애플리케이션이 정상적으로 동작하는지 Probe를 통해 확인할 수 있다. 프로브는 다음과 같은 방법으로 상태를 검사한다. 


### 프로브 수행 방식


프로브가 상태검사를 진행하는 방식은 아래의 3가지로 수행된다. 애플리케이션 답게 웹과 관련된 검사가 2개 존재한다. 

1. exec: 지정된 명령어를 수행한다. 명령어가 정상적으로 수행되면 성공한 것으로 판단한다.
2. TCPSocket: 지정한 IP 주소와 포트에 대해 TCP 검사를 수행한다. 포트가 열려있다면, 성공한 것으로 판단한다.
3. HTTPGET: HTTP의 GET 요청을 수행하여, 응답상태코드가 2xx 혹은 3xx이면 성공한 것으로 간주한다.

### 프로브 종류


위와 같은 방법으로 우리는 애플리케이션의 상태를 확인할 수 있다. 애플리케이션의 상태를 파악하고, 문제가 있다면 파드를 종료하고 재생성한다. 프로브의 종류는 LivenessProbe, readinessProbe, startupProbe 3가지가 있으며 어떤 프로브도 존재하지 않으면 상태검사는 항상 성공이다.

- **livenessProbe** : 컨테이너가 동작 중인지 여부를 나타낸다. 만약 활성 프로브에 실패한다면, kubelet은 컨테이너를 종료 및 재생성한다.
- **readinessProbe** : 컨테이너가 작업을 수행할 준비가 되었는지 확인한다. 만약 해당 프로브가 실패한다면, 엔드포인트 컨트롤러는 파드에 연관된 모든 서비스들의 엔드포인트에서 파드의 IP 주소를 제거한다. 기본상태는 실패이며, 프로브가 동작해서 정상적으로 완료되면 성공으로 바뀐다.
- **startupProbe** : 컨테이너 내의 애플리케이션이 시작되었는지를 나타낸다. 해당 프로브가 존재하면, 성공할 때 까지 다른 나머지 프로브는 활성화 되지 않는다. 만약 스타트업 프로브가 실패하면, kubelet이 컨테이너를 죽이고, 정책에 맞게 재시작한다.

## Jobs / CronJobs


### Jobs


하나 이상의 파드를 생성하고, 지정된 수의 파드가 성공적으로 종료될 때까지 작업을 진행한다.


```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4


```


### CronJobs


정기적으로 반복하는 작업을 수행하기 위해 사용된다. 해당 작업은 무기한 반복된다. ex) 매주, 매일, 매년


언제 반복될 것인지는 시간은 _ _ _ _ _ 5개로 표현된다. 첫번째부터 분, 시, 일, 달, 주이다. 


아래는 [공식문서](https://kubernetes.io/ko/docs/concepts/workloads/controllers/cron-jobs/)에서 설명한 문법 형식이다.


```yaml
# ┌───────────── 분 (0 - 59)
# │ ┌───────────── 시 (0 - 23)
# │ │ ┌───────────── 일 (1 - 31)
# │ │ │ ┌───────────── 월 (1 - 12)
# │ │ │ │ ┌───────────── 요일 (0 - 6) (일요일부터 토요일까지;
# │ │ │ │ │                                   특정 시스템에서는 7도 일요일)
# │ │ │ │ │                                   또는 sun, mon, tue, wed, thu, fri, sat
# │ │ │ │ │
```


| 항목                     | 설명              | 상응 표현     |
| ---------------------- | --------------- | --------- |
| @yearly (or @annually) | 매년 1월 1일 자정에 실행 | 0 0 1 1 * |
| @monthly               | 매월 1일 자정에 실행    | 0 0 1 * * |
| @weekly                | 매주 일요일 자정에 실행   | 0 0 * * 0 |
| @daily (or @midnight)  | 매일 자정에 실행       | 0 0 * * * |
| @hourly                | 매시 0분에 시작       | 0 * * * * |


*로 표시된 주기 만큼 반복하는 것이다. 그래서 만약 아무 표시도 없는 * * * * *의 경우 1분마다 작업을 수행한다. 내가 *을 월에 두었으면, “ex) 매달 5일 자정에 실행한다” 같이 매달 작업이 반복된다.


## 참고자료


[https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)


[https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)


[https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)


[https://medium.com/finda-tech/kubernetes-pod의-진단을-담당하는-서비스-probe-7872cec9e568](https://medium.com/finda-tech/kubernetes-pod%EC%9D%98-%EC%A7%84%EB%8B%A8%EC%9D%84-%EB%8B%B4%EB%8B%B9%ED%95%98%EB%8A%94-%EC%84%9C%EB%B9%84%EC%8A%A4-probe-7872cec9e568)

