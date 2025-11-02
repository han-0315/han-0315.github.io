---
layout: post
title: Node Life Cycle
date: 2025-11-02 09:00 +0900 
description: Node Life Cycle 살펴보기
category: [Kubernetes, Node] 
tags: [Node, Lifecycle, Evict, Kubelet] 
pin: false
math: true
mermaid: true
---



노드가 클러스터에 투입되고 제거될 때까지 전체적인 노드 라이프 사이클에 대해 정리한다.
<!--more-->


### 등록(Registration)


서버 혹은 VM을 클러스터에 등록하기 위해선 Container Runtime Engine, kubelet와 같은 필수 모듈을 설치하고, 인증서를 준비해야 한다. 이후 kubelet이 실행되면 kubelet이 kubelet config에 있는 노드명으로 kube-apiserver에 요청하여 등록한다. 


노드 스펙은 KubeletConfiguration으로 등록된다. 노드 등록 관련 설정 기본값은 true이다. 만약, 노드의 자동 등록을 막고 싶다면 `KubeletConfiguration`에서  `"registerNode": false`로 설정이 필요하다. 이 경우 kubelet이 실행되어도 자동 등록되지 않는다. 

- kubectl proxy로 API 서버 proxy를 만든 후 노드 스펙을 질의하면 구체적인 노드 스펙을 볼 수 있다.

`curl -X GET http://127.0.0.1:8001/api/v1/nodes/<node-name>/proxy/configz`



```bash
curl -s -X GET http://127.0.0.1:8001/api/v1/nodes/test-worker/proxy/configz | jq
{
  "kubeletconfig": {
    "enableServer": true,
    "staticPodPath": "/etc/kubernetes/manifests",
    "podLogsDir": "/var/log/pods",
    "syncFrequency": "1m0s",
    "fileCheckFrequency": "20s",
    "httpCheckFrequency": "20s",
    "address": "0.0.0.0",
    "port": 10250,
    "tlsCertFile": "/var/lib/kubelet/pki/kubelet.crt",
    "tlsPrivateKeyFile": "/var/lib/kubelet/pki/kubelet.key",
    "rotateCertificates": true,
    ...
    "registerNode": true,
    ...
    "localStorageCapacityIsolation": true,
    "containerRuntimeEndpoint": "unix:///run/containerd/containerd.sock",
    "failCgroupV1": false
  }
}

```


### 상태(Status)


노드의 상태는 Ready, NotReady, Unknown 3가지로 존재한다. kubelet이 kube-apiserver와 정상적으로 통신되면 Ready 상태를 유지한다. 


Ready, NotReady는 말 그대로 노드의 정상/비정상 상태를 나타낸다. 그렇다면 Unknown은 어떤 경우에 발생할까? Unknown은 노드의 상태 체크 과정에서, 정상적이진 않으나 아직 비정상으로 확정하기 어려운 상태를 의미한다. 자세한 내용은 아래의 헬스체크 부분에서 살펴본다.


### 헬스체크(Health check)


Lease라는 리소스를 통해 헬스체크가 이뤄진다. Lease는 동시성 관리를 위한 Lock 기능이다. 기본적으로 주요 컴포넌트들의 리더 선출 과정에서 사용된다. 컨트롤러, 스케줄러는 HA를 위해 1개가 아닌 다수의 파드로 분산되어 있으며 lease를 소유자가 없는 경우 모든 인스턴스가 lease를 소유하기 위해 노력한다. 획득한 경우 leaseDurationSeconds 기간안에 갱신(renew)해야 소유권을 유지할 수 있다.


![image.png](/assets/img/post/Node%20Lifecycle/1.png)


출처: [https://k8sjp.connpass.com/event/162343/](https://k8sjp.connpass.com/event/162343/)


노드의 헬스체크도 lease를 통해 진행한다. 각 노드는 자신의 이름에 대응하는 lease가 있으며, kubelet이 `leaseDurationSeconds` 기간안에 kube-apiserver와 통신하지 못하면 만료된다. kube-controller-manager는 만료를 감지하고 `node-monitor-grace-period` 동안 노드 상태 업데이트가 없었던 게 맞다면 노드 상태를 NotReady로 변경한다.


위와 같은 방식 말고 OS의 shutdown 및 CRI 상태 등을 고려해서 NotReady로 변경하는 경우도 있다고 한다. 
kubelet Config에 `GracefulNodeShutdown` 설정으로 kubelet이 서버의 셧다운을 인지하고 파드를 grafeful하게 종료시키는 옵션도 존재한다. 


관련 문서: [https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#enabling-graceful-node-shutdown](https://kubernetes.io/docs/concepts/cluster-administration/node-shutdown/#enabling-graceful-node-shutdown)


### NotReady


Ready에서 NotReady로 상태가 변하면, 무슨 일이 일어날까?


1) Node 관련 Taint가 업데이트된다. 


아래 Taint가 추가되어 스케줄링이 거부된다.

- `node.kubernetes.io/not-ready:NoExecute` (Ready가 “False”일 때)
- `node.kubernetes.io/unreachable:NoExecute` (Ready가 “Unknown”일 때)

2) Pod는 일정 시간 후 Evict 된다.


tolderationSeconds(default 300s)을 기다린 후 TaintEvictionController에 의해 evict된다. 파드의 상태는 노드의 상태가 NotReady가 된 경우 이를 바로 반영하며 관련 Endpoint도 업데이트된다.


하지만 일정 시간동안 1개의 파드는 정상적인 상태가 아니기에 노드(서버) 점검이 필요하면 Drain 후 점검하는 것이 안전하다.


### 제거(Delete)


등록은 자동으로 이뤄지지만, 삭제는 kube-apiserver를 통해 진행한다. 만약 etcd에서 노드 정보를 제거했어도 kubelet이 실행 중이라면 kubelet 설정에 의해 자동으로 등록될까?


kubelet은 시작하는 과정에서 노드 등록을 진행하므로 kubelet을 재실행하지 않는다면, kubelet이 실행 중이어도 노드는 다시 등록되지 않는다.

- 노드 확인

```bash
kubectl get nodes 
NAME                 STATUS   ROLES           AGE     VERSION
test-control-plane   Ready    control-plane   5m19s   v1.31.0
test-worker          Ready    <none>          5m8s    v1.31.0
test-worker2         Ready    <none>          5m8s    v1.31.0
```

- 테스트 노드 제거

```bash
kubectl delete nodes test-worker
node "test-worker" deleted
```


```bash
kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
test-control-plane   Ready    control-plane   5m31s   v1.31.0
test-worker2         Ready    <none>          5m20s   v1.31.0
```



노드가 자동으로 재등록되지 않는 걸 볼 수 있다. 그렇다면, 재등록이 필요한 경우에는 어떻게 해야할까?


→ kubelet을 재실행해주면 된다.


- kubelet restart(in worker)

```bash
systemctl restart kubelet
```


kubelet 로그를 확인해보면 먼저 1차적으로 초기 생성시 등록했던 로그를 확인할 수 있다. 이후 노드 제거 후 재실행과정에서 등록하는 걸 확인할 수 있다.


*재등록시 `Error updating node status` 로그가 바로 보이나, 이후 문제없이 상태가 동기화된다.


```bash
journalctl -u kubelet --since "1 hour ago" | grep -A2 'Attempting to register node' 
Nov 02 09:04:12 test-worker kubelet[214]: I1102 09:04:12.877102     214 kubelet_node_status.go:72] "Attempting to register node" node="test-worker"
Nov 02 09:04:12 test-worker kubelet[214]: I1102 09:04:12.880518     214 kubelet_node_status.go:75] "Successfully registered node" node="test-worker"
Nov 02 09:04:12 test-worker kubelet[214]: I1102 09:04:12.888921     214 kuberuntime_manager.go:1633] "Updating runtime config through cri with podcidr" CIDR="10.244.2.0/24"
--
Nov 02 09:09:38 test-worker kubelet[773]: I1102 09:09:38.114053     773 kubelet_node_status.go:72] "Attempting to register node" node="test-worker"
Nov 02 09:09:38 test-worker kubelet[773]: I1102 09:09:38.117757     773 kubelet_node_status.go:75] "Successfully registered node" node="test-worker"
Nov 02 09:09:38 test-worker kubelet[773]: E1102 09:09:38.117903     773 kubelet_node_status.go:535] "Error updating node status, will retry" err="error getting node \"test-worker\": node \"test-worker\" not found"
```



kubectl로 노드 리스트를 확인하면 아래와 같이 다시 등록된 test-worker를 확인할 수 있다.


```bash
kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
test-control-plane   Ready    control-plane   5m39s   v1.31.0
test-worker          Ready    <none>          2s      v1.31.0
test-worker2         Ready    <none>          5m28s   v1.31.0
```


### 마무리하며


이번 포스팅에서는 노드의 라이프 사이클에 대해 간단히 정리해봤다. 다음 긍레서는 (1) 노드의 리소스 관리와,  (2)  kubelet의 Pod Lifecycle 관리 방법에 대해 살펴보려고 한다. 

