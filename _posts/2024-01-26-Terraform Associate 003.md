---
layout: post
title: Terraform Associate 003 후기
date: 2024-01-25 21:00 +0900 
description: Terraform Associate 003 취득 후기
category: [자격증, Terraform] 
tags: [Terraform, 자격증, IaC] 
pin: false
math: true
mermaid: true
---
Terraform Associate 003 취득 후기
<!--more-->


## Terraform Associate 003 취득 후기


취득 6개월 만에 쓰는 Terraform 003 후기, 시험을 준비하시는 분들께 도움이 되었으면 좋겠습니다.


### 배경


단순하지만 재밌는 Terraform을 자주 쓰다가 제대로 공부를 해봐야겠다고 생각이 들었다. 꼼꼼하게 공부할 목적으로 Terraform 자격증을 찾았고, 이번에 003으로 업데이트되면서 v1.0 이상의 환경(현재와 유사한)으로 바뀌었다는 소식을 듣고 바로 접수했다. 


### 시험 환경


앞서 말했듯이 “**v1.0 이상”**의 환경이다.  시험 형태는 객관식이거나 T/F 문제로 구성되었다.


시험시간은 60분이다.  원격으로 참여하며 PSI를 통해 본다. 응시료는 약 100,000 내외였던 것으로 기억한다. Linux Foundation 시험에 비하면 상당히 저렴하다. 대신 실습시험이 아닌 필기시험이다.


문제 중 70%이상 맞출 경우 합격이며 유효기간 2년이다.


### 유형


주로 나왔던 시험유형은 다음과 같다. 시험을 볼 당시 003에 대한 자료가 부족하여, 002 Dump를 주로 풀었다. 실제 나온 시험의 유형과는 많이 달랐다. 지금은 003에 대한 자료가 많을 것 같다?!(6개월이 지났으니) 내가 시험을 봤을 때 주로 나왔던 유형은 다음과 같다.

- ~은 언제 적용되는가? (워크플로 관련 혹은 Terraform Cloud 관련)
	- `lock.state.tf`  파일은 언제 업데이트 되는가?
	- Sentinel이 적용되는 것은 언제인가?
- ~명령어를 통해 확인할 수 있는가? (다수 출제)
	- ex) output 명령어 결과 확인할 수 있는 것은?
	- `state-locking` 을 통해 얻을 수 있는 것?
- 설명 혹은 코드를 주고 알맞은 명령어를 묻는 문제(다수 출제)
	- state 파일만 업데이트하는 명령어는?
	- ~~타입의 변수가 아닌 것은?
- IaC 관련 설명으로 올바른 것은?
	- IaC의 핵심 원칙 혹은 원리가 아닌 것은?
- 하시코프 관련 문제(많이 나오진 않으나, Terraform과 연관이 없어 보여 당황했다.)
	- 하시코프의 제품은 아닌 것은?
- 기타
	- CI/CD 파이프라인에서 시크릿 정보를 넘기는 방법은?

### 공부


공부는 주로 Terraform 공식 문서를 통해 진행했다. [Terraform](https://developer.hashicorp.com/terraform/tutorials/certification-003/associate-study-003?product_intent=terraform)에서 Associate를 위한 학습 안내서가 나와 있다. 코드도 함께 있으니, 제법 공부할 만하다. 다만 한국어는 지원하지 않는 것 같다. 


CloudNet에서 진행하는 ‘테라폼으로 시작하는 IaC’ 책으로 진행하는 Terraform 스터디[T101]’ 스터디를 막바지에 병행했다. 하시코프에 다니시는 현직자분들도 함께 참여하여 많은 도움이 되었다. 관련 스터디 내용은 [링크](https://www.handongbee.com/categories/terraform/)에서 확인할 수 있다.


Terraform을 혼자 쓸때는 로컬에서 돌려, 협업과 관련된 기능을 몰랐다. 자격증을 공부하면서 실무에서 사용하는  원격저장소(Backend) 등 여러 기능과 모듈, 코드를 관리하는 방법에 대해 알아 도움이 많이 됐다. 관련 포스팅은 [여기](https://www.handongbee.com/posts/Backend(Remote-State)/)에서 확인할 수 있다.


### 시험후기


웹캠이 초점을 맞추지 못해 신분증으로만 1시간넘게 고생했다. 저렴한 웹캠이니, 초점을 정말 못 맞춘다. PSI에서는 선명한 신분증이 아니면 허락해 주지 않았다.~~(내가 보기엔 충분히 내용을 확인할 수 있었지만)~~


객관식이고, 내용에 어려운 것이 없어 시험시간은 약 20분 내로 끝났던 것으로 기억한다. (6개월 전이라 가물가물하다.) 이제 CKS, AWS-SAA 취득에 도전해 보려고 한다.


[아래는 뱃지 사진]


![Untitled.png](/assets/img/post/Terraform%20Associate%20003/7.png)

