---
layout: post
title: istio Observability(DashBoard)
date: 2025-05-03 09:01 +0900 
description: istio Observability(DashBoard) 살펴보기
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#4, DashBoard, Grafana, Kiali] 
pin: false
math: true
mermaid: true
---
istio DashBoard 살펴보기
<!--more-->


## **대시보드 도구 사용해보기**


### **Grafana**


그라파나는 운영하면서 적어도 한번쯤은 봤을 법한 대중적인 시각화 도구이다. 여기서는 그라파나를 통해 이전 포스트에서 확인한 지표들을 확인한다.


#### 설치 및 대시보드 설정


그라파나에 대한 설치자체는 프로메테우스를 배포하면서 같이 진행되었다. 여기서는 별도 대시보드 설정을 진행한다.


```bash
cd ch8

kubectl -n prometheus create cm istio-dashboards \
--from-file=pilot-dashboard.json=dashboards/\
pilot-dashboard.json \
--from-file=istio-workload-dashboard.json=dashboards/\
istio-workload-dashboard.json \
--from-file=istio-service-dashboard.json=dashboards/\
istio-service-dashboard.json \
--from-file=istio-performance-dashboard.json=dashboards/\
istio-performance-dashboard.json \
--from-file=istio-mesh-dashboard.json=dashboards/\
istio-mesh-dashboard.json \
--from-file=istio-extension-dashboard.json=dashboards/\
istio-extension-dashboard.json
configmap/istio-dashboards created
```


그라파나 오퍼레이터가 configmap을 인식하도록 별도 label을 달아준다.


```bash
kubectl label -n prometheus cm istio-dashboards grafana_dashboard=1

configmap/istio-dashboards labeled
```


#### control plane 관련 대시보드


앞서 살펴봤듯이 istiod와 관련된 지표 xDS API의 업데이트 횟수, 동기화 시간을 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/1.png)


추가로 Envoy와 관련된 xDS에 대한 상세한 정보들도 확인가능하다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/2.png)


#### 서비스 대시보드


여기에서는 별도 커스텀 메트릭이나 특정 엔보이 지표를 활성화하지 않아, 가장 대표적인 지표만 보여준다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/3.png)


### **Kiali**


Kiali 또한 그라파나와 같은 시각화 도구이다. 다만 집중하는 부분이 다른데, kiali는 실시간으로 갱신되는 메트릭을 사용해 서비스가 서로 **어떻게 통신하는지에 대한 방향 그래프**를 구축하는데 집중한다고 한다.


#### 설치


우선 helm repo를 추가한다.


```bash
helm repo add kiali https://kiali.org/helm-charts
helm repo update 
```


아래의 명령어로 kiali를 배포한다.


```bash
helm install --namespace kiali-operator --create-namespace --version 1.63.2 kiali-operator kiali/kiali-operator

NAME: kiali-operator
LAST DEPLOYED: Sun May  4 01:10:14 2025
NAMESPACE: kiali-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Welcome to Kiali! For more details on Kiali, see: https://kiali.io

The Kiali Operator [v1.63.2] has been installed in namespace [kiali-operator]. It will be ready soon.
You have elected not to install a Kiali CR. You must first install a Kiali CR before you can access Kiali. The operator is watching all namespaces, so you can create the Kiali CR anywhere.

If you ever want to uninstall the Kiali Operator, remember to delete the Kiali CR first before uninstalling the operator to give the operator a chance to uninstall and remove all the Kiali Server resources.

(Helm: Chart=[kiali-operator], Release=[kiali-operator], Version=[1.63.2])
```


이제 관련된 CRD YAML을 배포한다.(프로메테우스와 예거 설정을 진행한다.)


```bash
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  namespace: istio-system
  name: kiali
spec:
  istio_namespace: "istio-system"  
  istio_component_namespaces:
    prometheus: prometheus
  auth:    
    strategy: anonymous # 익명 접근 허용
  deployment:
    accessible_namespaces:
    - '**'
  external_services:    
    prometheus: # 클러스터 내에서 실행 중인 프로메테우스 설정
      cache_duration: 10
      cache_enabled: true
      cache_expiration: 300
      url: "http://prom-kube-prometheus-stack-prometheus.prometheus:9090"    
    tracing: # 클러스터 내에서 실행 중인 예거 설정
      enabled: true
      in_cluster_url: "http://tracing.istio-system:16685/jaeger"
      use_grpc: true
```


배포 확인


```bash
kubectl get deploy,svc -n istio-system kiali
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kiali   1/1     1            1           48s

NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                          AGE
service/kiali   NodePort   10.200.1.186   <none>        20001:30003/TCP,9090:31393/TCP   48s
```


이제 우리가 접속하기 편하게 서비스를 NortPort로 노출한다.


```bash
kubectl patch svc -n istio-system kiali -p '{"spec": {"type": "NodePort", "ports": [{"port": 20001, "targetPort": 20001, "nodePort": 30003}]}}'

service/kiali patched
```


#### 탭 확인하기


여기서는 kaili의 탭을 직접 들어가보며 각 탭이 구체적으로 어떤 기능이 있는지 확인해본다.


##### Overview


해당 탭에서는 네임스페이스와 각 네임스페이스에서 실행 중인 애플리케이션의 개수가 표기된다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/4.png)


##### Graph


서비스 매시의 트래픽 흐름을 보여주는 방향 그래프로, 아래와 같은 요소를 표시해준다.

- 트래픽의 이동과 흐름 Traversal and flow of traffic
- 바이트 수, 요청 개수 등 Number of bytes, requests, and so on
- 여러 버전에 대한 여러 트래픽 흐름(예: 카나리 릴리스나 가중치 라우팅)
- 초당 요청 수 Requests/second; 총량 대비 여러 버전의 트래픽 비율
- 네트워크 트래픽에 기반한 애플리케이션 상태 health
- HTTP/TCP 트래픽
- 빠르게 식별할 수 있는 네트워크 실패

우리가 반복적으로 테스트하고 있는 istioninaction namespace의 webapp 및 catalog와 관련된 정보를 그래프로 확인하면 아래와 같다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/5.png)


##### Workloads

- Overview : Pods of the service, Istio configuration applied to it, and a graph of the upstreams and downstreams
- Traffic : Success rate of inbound and outbound traffic
- Logs : Application logs, Envoy access logs, and spans correlated together Inbound Metrics and Outbound Metrics Correlated with spans
- Traces : The traces reported by Jaeger
- Envoy : The Envoy configuration applied to the workload, such as clusters, listeners, and routes

여기서는 webapp의 Workload를 확인해본다. 우선 트래픽을 확인해보면, ingress → webapp → catalog로 향하는 flow를 볼 수 있다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/6.png)


여기서는 Metrics 지표를 확인할 수 있다. catalog로 들어오는 지표를 확인해보자.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/7.png)


추가로 envoy에 대한 설정도 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(DashBoard)/8.png)


#### 주요한 장점

- **VirtualService가 존재하지 않는 Gateway를 가리킬 때**

Kiali는 지정된 Gateway 리소스가 클러스터에 실제 존재하는지 확인하고, 없을 경우 경고 아이콘으로 표시한다. 이를 통해 잘못된 Gateway 이름 오타나 삭제된 Gateway를 참조하는 문제를 바로 식별할 수 있다.

- **존재하지 않는 대상(destinations)으로의 라우팅**

VirtualService의 destination.host나 subset이 실제 Service 또는 DestinationRule에 정의되어 있지 않을 때, Kiali UI에서 붉은색 오류 표시와 함께 상세 원인을 제공해준다.

- **동일 호스트에 대해 둘 이상의 VirtualService 정의**

같은 호스트명에 대해 중복된 VirtualService가 있을 경우, 트래픽 분산이나 라우팅 충돌이 발생할 수 있다. Kiali는 중복 정의된 VirtualService를 목록으로 보여 주고, 우선순위 충돌 여부를 진단해준다.

- **찾을 수 없는 서비스의 subset**

DestinationRule에 정의된 subset이 실제 Service의 레이블 Selector 값과 일치하지 않아 생성되지 않으면, 알림을 제공한다. 

