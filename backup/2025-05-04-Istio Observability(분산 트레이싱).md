---
layout: post
title: istio Observability(분산 트레이싱)
date: 2025-05-03 09:02 +0900 
description: istio Observability(분산 트레이싱) 살펴보기
category: [Kubernetes, Network] 
tags: [istio, CloudNet, Kubernetes, Network, istio#4, tracing] 
pin: false
math: true
mermaid: true
---
istio Observability(분산 트레이싱) 살펴보기
<!--more-->


### 분산 트레이싱이란


모놀리식이 아닌, MSA로 이루어진 구조라면, 해당 서비스 간의 호출을 추적하는 것이 필요하다. 그래야 어디서 문제가 생겼는지 쉽게 파악할 수 있다. 


이를 위해선 서비스 간 호출을 나타내는 상관관계(correlation)ID와 서비스 간 호출 그래프를 거치는 특정 요청을 나타내는 트레이스(trace)ID를 남기고, 엔진은 이를 통해 호출 그래프에서 어디서 문제가 발생했는지 정보를 알려준다. 


특히, istio는 모든 Pod의 사이드카로 붙어 통신을 제어하다보니 이런 메타데이터를 쉽게 추가할 수 있다.


### 동작 방식


요청 중 자신이 처리하는 부분을 나타내는 스팬을 만들고, 이를 오픈트레이싱 엔진에 보낸 뒤 트레이스 콘텍스트를 다른 서비스로 전파한다. 분산 트레이싱 엔진은 이런 스팬과 트레이스 콘텍스트를 사용해 트레이스를 구축할 수 있다.  


*여기서 트레이스란 서비스 간의 인과 관계를 말하며 방향, 타이밍과 기타 디버깅 정보를 보여준다.


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/1.png)


#### 오픈소스 분산 트레이싱 구현체

- 예거 Jaeger(여기서는 해당 솔루션을 사용해서 실습한다.)
- 집킨 Zipkin
- 라이트스텝 Lightstep
- 인스타나 Instana

## 실습 


### 설치


control plane 노드에 접속하여 istio sample jaeger yaml을 배포한다. 


```bash
docker exec -it myk8s-control-plane bash
---
kubectl apply -f istio-$ISTIOV/samples/addons/jaeger.yaml
deployment.apps/jaeger created
service/tracing created
service/zipkin created
service/jaeger-collector created
```


로컬에서 접속하기 편하게 NodePort로 노출한다.


```bash
kubectl patch svc -n istio-system tracing -p '{"spec": {"type": "NodePort", "ports": [{"port": 80, "targetPort": 16686, "nodePort": 30004}]}}'

service/tracing patched
```


해당 포트로 [localhost:30004](http://localhost:30004/) 로 접근하면 아래와 같이 JAEGER UI를 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/2.png)


#### istio 설정


이제 JAEGER에 대한 트레이싱을 진행하도록 istio operator에 설정을 진행한다.


```yaml
# cat ch8/install-istio-tracing-zipkin.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  meshConfig:
    defaultConfig:
      tracing:
        sampling: 100
        zipkin:
          address: zipkin.istio-system:9411
```


위의 설정을 배포한다.


```bash
docker exec -it myk8s-control-plane bash
---
istioctl install -y -f install-istio-tracing-zipkin.yaml
✔ Istio core installed                                                                                             
✔ Istiod installed                                                                                                 
✔ Ingress gateways installed                                                                                       
- Pruning removed resources                                                                                          Removed Deployment:istio-system:istio-egressgateway.
  Removed Service:istio-system:istio-egressgateway.
  Removed ServiceAccount:istio-system:istio-egressgateway-service-account.
  Removed RoleBinding:istio-system:istio-egressgateway-sds.
  Removed Role:istio-system:istio-egressgateway-sds.
  Removed PodDisruptionBudget:istio-system:istio-egressgateway.
✔ Installation complete                                                                                            Making this installation the default for injection and validation.

Thank you for installing Istio 1.17.  Please take a few minutes to tell us about your install/upgrade experience!  https://forms.gle/hMHGiwZHPU7UQRWe9

```


또, 여기선 추가로 실습을 위해 외부로 통신하는 httbin 리소스를 배포한다.


```bash
kubectl apply -n istioinaction -f ch8/tracing/thin-httpbin-virtualservice.yaml

gateway.networking.istio.io/coolstore-gateway configured
virtualservice.networking.istio.io/thin-httbin-virtualservice created
serviceentry.networking.istio.io/external-httpbin-org created
kubectl get gw,vs,serviceentry -n istioinaction

NAME                                            AGE
gateway.networking.istio.io/coolstore-gateway   4h16m

NAME                                                            GATEWAYS                HOSTS                          AGE
virtualservice.networking.istio.io/thin-httbin-virtualservice   ["coolstore-gateway"]   ["httpbin.istioinaction.io"]   2s
virtualservice.networking.istio.io/webapp-virtualservice        ["coolstore-gateway"]   ["webapp.istioinaction.io"]    4h16m

NAME                                                    HOSTS             LOCATION        RESOLUTION   AGE
serviceentry.networking.istio.io/external-httpbin-org   ["httpbin.org"]   MESH_EXTERNAL   DNS          2s

```


도메인 설정도 같이 진행


```bash
echo "127.0.0.1       httpbin.istioinaction.io" | sudo tee -a /etc/hosts

Password:
127.0.0.1       httpbin.istioinaction.io

```


이제 httpbin에 접속해보면 아래와 같은 결과를 볼 수 있다.  X-B3-* 헤더는 Jaeger관련 헤더로 “헤더와 ID 자동 주입”을 확인할 수 있다.


```bash
curl -s http://httpbin.istioinaction.io:30000/headers | jq

{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.istioinaction.io",
    "User-Agent": "curl/8.7.1",
    "X-Amzn-Trace-Id": "Root=1-68164ac9-214ab03e1d7d97914cbe3acd",
    "X-B3-Sampled": "1",
    "X-B3-Spanid": "067b5caddfbe3f32",
    "X-B3-Traceid": "551498939a026368067b5caddfbe3f32",
    "X-Envoy-Attempt-Count": "1",
    "X-Envoy-Decorator-Operation": "httpbin.org:80/*",
    "X-Envoy-Internal": "true",
    "X-Envoy-Peer-Metadata": "ChQKDkFQUF9DT05UQUlORVJTEgIaAAoaCgpDTFVTVEVSX0lEEgwaCkt1YmVybmV0ZXMKHAoMSU5TVEFOQ0VfSVBTEgwaCjEwLjEwLjAuMjMKGQoNSVNUSU9fVkVSU0lPThIIGgYxLjE3LjgKnAMKBkxBQkVMUxKRAyqOAwodCgNhcHASFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKEwoFY2hhcnQSChoIZ2F0ZXdheXMKFAoIaGVyaXRhZ2USCBoGVGlsbGVyCjYKKWluc3RhbGwub3BlcmF0b3IuaXN0aW8uaW8vb3duaW5nLXJlc291cmNlEgkaB3Vua25vd24KGQoFaXN0aW8SEBoOaW5ncmVzc2dhdGV3YXkKGQoMaXN0aW8uaW8vcmV2EgkaB2RlZmF1bHQKMAobb3BlcmF0b3IuaXN0aW8uaW8vY29tcG9uZW50EhEaD0luZ3Jlc3NHYXRld2F5cwoSCgdyZWxlYXNlEgcaBWlzdGlvCjkKH3NlcnZpY2UuaXN0aW8uaW8vY2Fub25pY2FsLW5hbWUSFhoUaXN0aW8taW5ncmVzc2dhdGV3YXkKLwojc2VydmljZS5pc3Rpby5pby9jYW5vbmljYWwtcmV2aXNpb24SCBoGbGF0ZXN0CiIKF3NpZGVjYXIuaXN0aW8uaW8vaW5qZWN0EgcaBWZhbHNlChoKB01FU0hfSUQSDxoNY2x1c3Rlci5sb2NhbAouCgROQU1FEiYaJGlzdGlvLWluZ3Jlc3NnYXRld2F5LTk5NmJjNmJiNi0yOWNwdwobCglOQU1FU1BBQ0USDhoMaXN0aW8tc3lzdGVtCl0KBU9XTkVSElQaUmt1YmVybmV0ZXM6Ly9hcGlzL2FwcHMvdjEvbmFtZXNwYWNlcy9pc3Rpby1zeXN0ZW0vZGVwbG95bWVudHMvaXN0aW8taW5ncmVzc2dhdGV3YXkKFwoRUExBVEZPUk1fTUVUQURBVEESAioACicKDVdPUktMT0FEX05BTUUSFhoUaXN0aW8taW5ncmVzc2dhdGV3YXk=",
    "X-Envoy-Peer-Metadata-Id": "router~10.10.0.23~istio-ingressgateway-996bc6bb6-29cpw.istio-system~istio-system.svc.cluster.local"
  }
}
```


### 데이터 확인


이제 실제 UI에 들어가서 확인해보자. 


[http://127.0.0.1:30004](http://127.0.0.1:30004/) 접속한다. 접속 후 namespace를 istio-ingress로 선택한다.


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/3.png)


특정 요청에 대해 선택하면, 아래와 같이 좀 더 세부적인 SPAN 정보를 볼 수 있다.


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/4.png)


### 커스터마이징


#### 트레이스의 태그


트레이스의 태그, 메데이트를 추가할 수 있다. 애플리케이션에 대한 커스텀 정보를 담아 이를 스팬에 추가가 가능한데 저술시점으로 다음과 같은 3가지 유형의 태그가 가능하다고 한다.

- 명시적으로 값 지정하기 Explicitly specifying a value
- 환경 변수에서 값 가져오기 Pulling a value from environment variables
- 요청 헤더에서 값 가져오기 Pulling a value from request headers

여기서는 custom_tag에 “Test Tag”라는 글자를 고정적으로 넣어보자.


```bash
cat ch8/webapp-deployment-zipkin-tag.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: webapp
  name: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      annotations:
        proxy.istio.io/config: |
          tracing:
            sampling: 100
            customTags:
              custom_tag:
                literal:
                  value: "Test Tag"
            zipkin:
              address: zipkin.istio-system:9411
      labels:
        app: webapp
...
```


이제 웹에서 확인해보면, 다음과 같이 custom tag가 달린 것을 확인할 수 있다.


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/5.png)


#### 백엔드 분산 트레이싱 엔진


Telemetry API로 제공되지 않는 부트스트랩 레벨의 고급 트레이싱 옵션(예: 커스텀 HTTP 헤더, 샘플링 플래그 등)도 직접 조정이 가능하다. 


여기서는 커스텀 부트스트램 설정을 진행해본다.

- 현재 webapp에 대한 istio 트레이싱 설정 확인

```bash
root@myk8s-control-plane:/# istioctl pc bootstrap -n istioinaction deploy/webapp -o json | jq .bootstrap.tracing
{
  "http": {
    "name": "envoy.tracers.zipkin",
    "typedConfig": {
      "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
      "collectorCluster": "zipkin",
      "collectorEndpoint": "/api/v2/spans",
      "traceId128bit": true,
      "sharedSpanContext": false,
      "collectorEndpointVersion": "HTTP_JSON"
    }
  }
}
```


새로운 bootstrap 배포


```bash
cat ch8/istio-custom-bootstrap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-custom-zipkin
data:
  custom_bootstrap.json: |
    {
      "tracing": {
        "http": {
          "name": "envoy.tracers.zipkin",
          "typedConfig": {
            "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
            "collectorCluster": "zipkin",
            "collectorEndpoint": "/zipkin/api/v1/spans",
            "traceId128bit": "true",
            "collectorEndpointVersion": "HTTP_JSON"
          }
        }
      }
    }%                                                                                                             
```


webapp에 설정 변경


```bash
cat ch8/webapp-deployment-custom-boot.yaml | grep boot
        sidecar.istio.io/bootstrapOverride: "istio-custom-zipkin"
```


다시 webapp에 대한 istio 트레이싱 설정 확인하면 아래와 같이 `collectorEndpoint`가 변경된 것을 확인할 수 있따.


```bash
root@myk8s-control-plane:/# istioctl pc bootstrap -n istioinaction deploy/webapp -o json | jq .bootstrap.tracing
{
  "http": {
    "name": "envoy.tracers.zipkin",
    "typedConfig": {
      "@type": "type.googleapis.com/envoy.config.trace.v3.ZipkinConfig",
      "collectorCluster": "zipkin",
      "collectorEndpoint": "/zipkin/api/v1/spans",
      "traceId128bit": true,
      "collectorEndpointVersion": "HTTP_JSON"
    }
  }
}
```


설정이 진행되면 아래와 같이 webapp에 대한 지표가 사라진 것을 볼 수 있다. 이는 Jaeger의 엔드포인트가 


/api/v1/spans (v1 JSON) or /api/v2/spans (v2 JSON)으로 향해야하는데 우리가 이걸 부트스트램으로 변경했기 때문이다. 


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/6.png)


다시 원복하면


![image.png](/assets/img/post/Istio%20Observability(분산%20트레이싱)/7.png)


위와 같이 webapp도 정상적으로 보이는 것을 확인할 수 있다.

