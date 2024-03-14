---
layout: post
title: 
date: 2023-12-01 11:00 +0900
description: Terraform Basic 정리
category: [IaC, Terraform]
tags: [IaC, Terraform, Basic] 
image:
  path: /assets/img/logo/Terraform/logo.png
  alt: Terraform Logo
pin: false
math: true
mermaid: true
---
Terraform Basic 정리
<!--more-->


## Terraform이란?


Terraform은 코드로 인프라를 관리하는 도구이다. 유사한 도구로는 AWS의 CloudFormation이 있는데, AWS에 치중된 CloudFormation과 다르게 Terraform은 오픈소스(?)로 Azure 등 다양한 클라우드 서비스를 제공한다는 장점이 있다.


테라폼은 아래와 같은 특징이 있다.

- **immutable(불변):** 일반적인 상황에서는 한번 배포된 리소스는 수정할 수 없다. 다만, 인스턴스 타입 등은 리소스를 재생성하지 않고 수정할 수 있다.
	- 살짝 주제를 벗어났지만, EC2의 경우 인스턴스 타입을 변경할 경우, 스토리지는 유지되나, 메모리는 날아간다. - 그래서 동일한 서비스를 진행하나 다운타임이 생긴다.
- declarative(선언형): C언어와 같이 명령형으로 작성하는 것이 구조체를 선언하듯 구체적인 특성을 정의함.
- **State file:** 리소스에 대한 상태를 보관하는 파일이다. ID 값을 통해 상태를 보관하며, state 파일은 그대로 있는 데 임의로 클라우드에서 리소스에 대한 상태를 변경하면 리소스를 추적할 수 없어, Terraform이 제대로 작동하지 않는다.
	- Backends 블럭을 통해 상태를 원격으로 관리할 수 있다. VCS에 커밋하고 Terraform 클라우드를 사용하여 효율적으로 관리할 수 있다.
- **modules:** 재사용가능한 모듈을 지원하며, Terraform Registry에 공개적으로 모듈을 업로드할 수 있고 하시코프가 검증도 진행한다.

### 구성요소


Terraform은 크게 테라폼 코드를 실행하는 Terraform Core(CLI)와 클라우드 서비스 제공자와 관련된 Plugins(Provider, Provisioner)으로 이루어져있다. 


이외에도 Terraform 모듈을 공유하는 저장소인 Registry, 현재 인프라의 상태를 관리하여 인프라 변경사항을 추적할 수 있는 State 파일 등이 추가적으로 존재한다.


### What is IaC?


`Iac(infrastructre as Code)`


코드를 통해 인프라를 관리하고 프로비저닝(Code를 통해 인프라를 생성, 변경, 삭제)하는 것을 의미한다.

- HashiCorp(하시코드)에서 제작한 오픈소스 Iac
- Declarative(선언형) : 원하는 최종 결과를 정의.
- 불변 : 인프라가 배포된 후, 업데이트가 아닌 재생성 및 교체
	- 몇가지 특성은 업데이트가 가능 ex) EC2 인스턴스 타입, 하지만 리소스는 초기화된다.
- Golang
- API로 모든 항목 관리

**장점**

- **코드를 관리하는 여러 툴, 재사용성의 강점**

	Git 과 같은 형상 관리와 연동하여(하지만, 깃을 저장하지 않고 별도의 registry를 통해 관리한다.), 변경 이력을 남겨 문제 원인을 파악하기 쉽다.


테라폼 문서에서는 아래와 같이 인프라 라이프사이클을 정의한다.


Day 0 : 초기 인프라를 프로비저닝 한다.


Day 1 : OS 및 애플리케이션 구성을 완료한다.


Day 1 ~ Day N : 구성 관리를 진행한다. **(Chef, Ansible, Docker 와 같은 도구를 Terraform에서 사용할 수 있다.)**


추가적으로, 위의 사이트에서는 다음과 같은 영상을 제공한다. 영어이고, 중복된 내용이 많아 직접 수강하진 않고 Summarize를 통해 요약했다.




## 아키텍처


테라폼은 논리적으로 크게 Terraform core, Terraform Plugins 으로 나뉜다. Terraform Core는 원격 프로시저 호출(RPC)을 사용하여 플러그인과 통신하며, 플러그인(프로바이더)는 관련 라이브러리를 통해 클라우드 API를 호출한다. 

![](/assets/img/post/Terraform%20Basic/1.jpg)
<p align="center">
[출처: https://developer.hashicorp.com/terraform/plugin]
</p>

### Terraform Core(CLI)


Terraform 코드를 실행하는 바이너리 파일으로, CLI 명령어를 사용하면 실행되는 프로그램이다. 이것은 사용자가 작성한 Terraform 코드를 읽고, 해당 코드에 기술된 리소스를 생성, 수정, 또는 삭제해야 하는지 판단하여, 필요에 따라 플러그인을 통해 플랫폼의 API를 호출한다.


### Terraform Plugins


RPC를 통해 Terraform Core에서 플러그인을 호출한다. 각 플러그인은 AWS와 같은 특정 플랫폼에 대한 API 등의 정보를 가지고 있으며, Core가 요청한 작업(생성, 삭제, 수정)을 수행할 수 있다. 플러그인은 “**프로바이더”**라는 형태로 제공된다. “**Init 명령어를 통해 필요한 플러그인을 검색하고 다운로드한다.”**


## 사용예시


### **Multi-Cloud Deployment**


한번에 다양한 제공자(클라우드)에 배포할 수 있다.

- [다중 클라우드 쿠버네티스에 배포하기(eks & aks)](https://developer.hashicorp.com/terraform/tutorials/networking/multicloud-kubernetes)
	- consul과 mesh gateway를 통해 두 독립적인 클러스터를 연결한다.? 관련 내용은 공부해야 할 것 같다.

### **Application Infrastructure Deployment, Scaling, and Monitoring Tools**


다양한 애플리케이션 계층에 대한 리소스를 관리할 수 있고, 계층간의 종속성을 자동으로 처리할 수 있다. 예를 들면, Terraform은 DB Tier를 배포 후 웹 서버(애플리케이션 티어)를 배포한다.

- [Datadog를 통한 모니터링 자동화](https://developer.hashicorp.com/terraform/tutorials/applications/datadog-provider)
- [블루-그린 및 카나리 배포에 애플리케이션 로드밸런서 사용하기](https://developer.hashicorp.com/terraform/tutorials/aws/blue-green-canary-tests-deployments)
	- 네트워킹 리소스(VPC, 보안 그룹, 로드 밸런서)와 웹 서버를 Blue 환경으로 프로비저닝합니다.
	- Green 환경으로 사용할 두 번째 웹 서버 세트를 프로비저닝합니다.
	- Terraform 구성에 기능 토글을 추가하여 잠재적인 배포 전략을 정의하고, 기능 토글을 사용하여 카나리아 테스트를 수행하고 Green 환경으로 점진적으로 변경할 수 있다.

### **Self-Service Clusters**


대규모의 조직에서는 반복적인 인프라 요청을 받을 수 있다. 그렇기에 테라폼에서 “self-serve” 모델을 구축하여 이를 해결할 수 있다. 이는 Terraform 모듈을 통해 만들 수 있다. 또한, Terraform Cloud를 사용하여 ServiceNow와 같은 티켓 시스템과 연동할 수 있다. 

- [ServiceNow 서비스와 통합하기(Terraform Cloud)](https://developer.hashicorp.com/terraform/cloud-docs/integrations/service-now)

### **Policy Compliance and Management**


리소스에 대한 정책을 테라폼을 통해 관리할 수 있다. Sentienl을 사용하여 테라폼이 인프라를 변경하기 전 규정 준수 및 거버넌스 정책을 자동으로 실행할 수 있다. Sentinel은 유료이다.(Terraform Cloud or 거버너스 티어만 가능)

- [Control Costs with Policies](https://developer.hashicorp.com/terraform/tutorials/cloud-get-started/cost-estimation)
- [Sentinel documentation](https://content.hashicorp.com/api/assets?product=terraform&version=refs%2Fheads%2Fv1.1&asset=website%2Fdocs%2Fcloud%2Fsentinel%2Findex.html)

### **Software Defined Networking**


테라폼을 통해 Network 자동화를 진행할 수 있다. Consul에 연결하여 서비스 상태 및 모니터링 도구인 Consul-Terraform-Sync에 의해 수행된다. 


아래와 같은 아키텍처로 Consul에서 네트워크 구성을 정의하면 테라폼이 자동적으로 인식하고, 인프라 제공자(Route 53등등)에 프로비저닝하는 구조인 것 같다.


- [consul-terraform-sync-intro](https://developer.hashicorp.com/consul/tutorials/network-infrastructure-automation/consul-terraform-sync-intro)
- [consul-terraform-sync-terraform-enterprise](https://developer.hashicorp.com/consul/tutorials/network-infrastructure-automation/consul-terraform-sync-terraform-enterprise)

### **Kubernetes**


테라폼에서 쿠버네티스 클러스터를 배포하고, 리소스를 관리할 수 있다. 서비스, 파드 등 리소스를 배포할 수 있는 것은 처음 알았다. 

- [튜토리얼](https://developer.hashicorp.com/terraform/tutorials/kubernetes/kubernetes-provider)

### **Parallel Environments**


어느정도 규모가 있는 조직에서는 Dev, Staging, Prod 등 같은 환경이지만, 별도로 구성해야 한다. 이런 인프라를 구축하고 유지하는 것은 점점 더 어려워지며 테라폼을 통해 쉽게 해결할 수 있다. 특히 Workspace기능을 통해 같은 코드를 각 워크스페이스마다 분리하여 구성가능하다. 

