---
layout: post
title: Proxy 살펴보기(Pingora)
date: 2024-10-19 09:00 +0900 
description: Proxy 살펴보기(Pingora)
category: [Network, Proxy] 
tags: [KANS, CloudNet, Network, KANS#7, Rust, Pingora ] 
pin: false
math: true
mermaid: true
---
Proxy 살펴보기(Pingora)
<!--more-->


### 들어가며


istio에서는 Envoy를 sidecar proxy로 선택하여 사용한다. istio의 Envoy이 선택이유를 찾다가 여러 Proxy를 확인할 수 있었는데, 그중에 Cloudflare에서 올해 4월에 오픈소스로 공개한 따끈따근한 Pingora라는 프록시가 있길래 이번 기회에 자세히 알아봤다. 


*바로 옆 부서에서 자사 서비스로 들어오는 대규모 트래픽을 처리하는 LB도 개발하고 있기에 관심이 더욱 갔다.


#### nginx


우선 Proxy에서는 빠질 수 없는 nginx에 대해 간단히 알아본다. 


apache httpd는 요청마다 connection을 제공하여 연결이 많아지면 서버가 동작하지 않는 c10k 문제가 있었다. 요청마다 프로세스를 생성하다보니, 동시접속자 수 만명을 감당하지 못하는 문제가 발생한다.(당시 컴퓨팅 리소스 기준)


![image.png](/assets/img/post/Proxy(Pingora)/1.png)


출처: [https://www.linkedin.com/pulse/architecture-nginx-built-scale-performance-inside-akshat-pattiwar/](https://www.linkedin.com/pulse/architecture-nginx-built-scale-performance-inside-akshat-pattiwar/)


nginx는 c10k 문제를 해결하기 위해 이벤트 기반 아키텍처를 도입한다. 프로세스는 master, worker 프로세스가 존재하며, master 프로세스는 클라이언트의 요청을 받고, 이를 worker에게 할당해주며 worker 프로세스는 넘겨준 작업을 처리한다. 즉, httpd와는 다르게 프로세스를 무작정 늘리지 않는다.


### Pingora


Pingora는 Rust 기반으로 제작된 Proxy 프레임워크로 Cloudflare에서 제작했다. Cloudflare는 nginx에서 Pingora로 전환하여 CPU와 메모리 사용량을 3분의 1로 줄일 수 있었다고 한다. 


우선 nginx에서 새로운 proxy를 개발하게된 이유를 살펴보고, Pingora에 대해 자세히 살펴본다.


### 새로운 proxy를 개발한 이유


#### 아키텍처 제한으로 인한 성능 저하 


nginx는 요청이 들어오면, 알맞은 worker에게 할당하고 worker가 작업을 처리하는 방식이다. cloudflare에서는 이런 아키텍처가 성능과 효율성을 저해한다고 한다. 


각 요청을 하나의 worker만 처리하며 선점이 불가능하기에, 작업 시간이 길어지는 요청이 들어오면 모든 요청에 대한 응답시간이 길어진다.


결국 CPU를 많이 사용하거나, I/O 작업을 진행하는 요청은 다른 요청의 속도를 저하한다. 


Cloudflare는 WAF로도 유명한 회사이다. WAF의 경우 이렇게 다른 요청의 속도를 저하는 문제가 주로 발생했다고 한다.


WAF의 경우, 특정 요청을 검사하여 잠재적인 위협을 찾아내는 역할을 한다. 대부분의 WAF 프로세스는 ms 수준이지만, 일부 요청의 경우 시간이 더 필요하다. 이럴 때 이벤트 루프기반이라면 다른 요청들도 같이 느려진다.


**이를 위해 nginx는 thread pool을 도입했다.**


“nginx 1.7.11버전 이후 thread pool을 도입하여, 오래걸리는 작업을 처리하는 전용 Thread를 지정할 수 있다.”


하지만, CloudFlare에서도 스레드 풀을 통해 연산이 어느 정도 많이 걸리는 요청들을 전용 스레드에 넘기면서 이를 어느 정도 보완할 수 있었지만, 많은 변수가 있었다고 한다.


#### Connection 재사용


Nginx에서는 아래의 그림과 worker 당 하나의 Connection Pool을 가지기에 특정 워커에서 Origin 서버와 Connection을 만들었어도 다른 Worker에서는 이를 활용할 수 없다.


![image.png](/assets/img/post/Proxy(Pingora)/2.png)


출처: [https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/)


하지만, Pingora는 멀티 프로세싱이 아닌 멀티 스레딩 아키텍처를 도입하여 하나의 Connection pool만 가진다. 이를 통해 connection 재사용 극대화하여 요청의 TTFB(time to first byte) 속도를 높일 수 있었다고 한다.


![image.png](/assets/img/post/Proxy(Pingora)/3.png)


출처: [https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/)


#### Nginx 커스터마이징의 어려움


Nginx의 업스트림을 따르면서, 추가 기능을 구현하는 것은 때로 너무 어려웠다라고 한다.


예를 들어, 요청에 대한 재시도 or Failover 할 때, 다른 origin 서버에 다른 request header를 붙여서 요청을 보내고 싶었지만, nginx는 이를 허용하지 않는다. 


#### C의 메모리 불안전성


nginx는 c언어로 구현되었기에 코드를 수정하다보면 메모리 불안정성이 있다. 우리가 C를 보완하기 위해 사용한 또 다른 언어는 Lua이다. 위험성은 적지만 성능도 떨어지며 더 복잡하다.


### Pingora


#### Pingora 설계

- Rust: 성능과 메모리 안정성을 위해 Rust를 선택
- Multi Threading: connection pool을 쉽게 공유하기 위해 멀티 스레딩을 선택
- nginx에서 쉽게 마이그레이션 가능하도록 nginx와 유사한 “life of a request” 이벤트 기반 인터페이스를 제공
- hyper라는 HTTP 라이브러리가 있었지만, 자사 맞춤형으로 트래픽 처리 유연성을 극대화하기위해 별도로 개발

#### Pingora 장점

- 연결 재사용률이 87.1% → 99.92%로 증가(신규 연결이 160배나 감소)
	- cloudflare의 모든 고객으로 범위를 넓히면, 매일 434년의 신규 연결에 대한 핸드쉐이크 시간을 절약할 수 있었다고 한다.
- lua 코드를 더 효율적으로 실행
	- NGINX에서는 lua 코드가 HTTP 헤더에 접근하려면 C 구조체에서 헤더를 읽고 Lua 문자열을 할당한 다음 Lua 문자열에 복사해야 합니다. 이후 Lua는 새 문자열도 가비지 수집도 필요하지만, Pingora에서는 그냥 문자열에 직접 접근할 수 있다.
- 메모리 안정성

모든 스레드에서 connection을 공유하면서, 전체 트래픽의 중앙값이 TTFB는 5ms, 95번째 백분위 수에서는 80ms가 감소했다고 한다.


예를 들어 100개의 요청이 있다면, 응답 속도 순으로 정렬했을 때 95번째의 요청 응답시간이 기존과 대비하여 80ms가 줄었다는 의미이다.


#### 추천하는 환경


아래와 같은 상황일 때, Pingora를 추천한다고 한다. 또한, [문서](https://blog.cloudflare.com/ko-kr/pingora-open-source/)에서 Pingora를 사용해 간단히 로드밸런서를 구축하는 핸즈온을 확인할 수 있다.

- **보안이 최우선인 경우:** 메모리 안전성
- **서비스가 성능에 민감한 경우:** 멀티 스레드 아키텍처 덕분에 connection 재사용, lua 최적화로 성능 향상 가능
- **광범이한 사용자에 대한 프로그래밍이 필요한 경우:** Pingora 프레임워크가 제공하는 API를 통해 맞춤형 고급 게이트웨이 또는 로드 밸런서를 구축할 수 있다.

### Envoy와 Pingora


Envoy는 CNCF 프로젝트 중 하나로 cilium(cni), istio(service mesh), contour(ingress) 등 여러 쿠버네티스 관련 프로젝트에서 사용된다. 구글 IBM 리프트(Lyft)가 중심이 되어 개발하고 있으며, C++로 구성된다.


Envoy 또한 Pingora와 같이 multi Threading 아키텍처를 사용하여 Connection Pool을 효율적으로 관리한다.


**개인적으로 느끼는 차이점**


Envoy는 좀 더 클라우드 환경과 마이크로서비스에 초점을 맞춰 개발되고 있고, Pingora는 대규모 트래픽을 처리하는 LB쪽으로 개발되고 있는 것 같다. Pingora는 Rust라 C++에 비해 메모리 안정성이나, 개발하기 편할 것 같지만 커뮤니티는 CNCF에 비해 활발하지 않을 것 같다.


### 참고 자료


nginx 사용시 문제상황

- 비선점으로 인한 문제 [https://blog.cloudflare.com/the-problem-with-event-loops/](https://blog.cloudflare.com/the-problem-with-event-loops/)
- Disk I/O로 인한 문제 [https://blog.cloudflare.com/how-we-scaled-nginx-and-saved-the-world-54-years-every-day/](https://blog.cloudflare.com/how-we-scaled-nginx-and-saved-the-world-54-years-every-day/)

Pingora 전환 관련

- 구체적인 전환 이유: [https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/)
- 오픈소스 공개 관련된 한글 문서: [https://blog.cloudflare.com/ko-kr/pingora-open-source/](https://blog.cloudflare.com/ko-kr/pingora-open-source/)

Envoy

- [https://www.envoyproxy.io/docs/envoy/v1.32.0/intro/arch_overview/intro/threading_model](https://www.envoyproxy.io/docs/envoy/v1.32.0/intro/arch_overview/intro/threading_model)
