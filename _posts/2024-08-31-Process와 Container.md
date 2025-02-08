---
layout: post
title: Process와 Container
date: 2024-08-31 09:00 +0900 
description: KANS 스터디 1주차 Process와 Container
category: [Docker, Process] 
tags: [KANS, CloudNet, Kubernetes, Network, KANS#1] 
pin: false
math: true
mermaid: true
---
KANS 스터디 1주차 Process와 Container
<!--more-->


> 💡 **KANS 스터디**  
> CloudNet에서 주관하는 KANS(**K**ubernetes **A**dvanced **N**etworking **S**tudy)으로 쿠버네티스 네트워킹 스터디입니다. 아래의 글은 스터디의 내용을 기반으로 작성했습니다.  
>   
> 스터디에 관심이 있으신 분은 [CloudNet Blog](/c9dfa44a27ff431dafdd2edacc8a1863)를 참고해주세요.


	CloudNet에서 주관하는 KANS(**K**ubernetes **A**dvanced **N**etworking **S**tudy)으로 쿠버네티스 네트워킹 스터디입니다. 아래의 글은 스터디의 내용을 기반으로 작성했습니다.


	스터디에 관심이 있으신 분은 [CloudNet Blog](/c9dfa44a27ff431dafdd2edacc8a1863)를 참고해주세요.


### 시작하기에 앞서


우선 가상화가 왜 필요한지 알아보자. 가상화가 필요한 이유는 크게 아래와 같다.

- 효율화: 하나의 서버를 쪼개서 더 많은 용도로 사용할 수 있다.
- 사용성: 하나의 호스트에서 목적에 맞는 운영체제를 다양하게 띄울 수 있다.

하지만, Vmware와 같은 가상화 툴을 이용하면, 아래와 같은 추가 Layer가 생긴다.


> “Host OS > **Vmware >** **Guest HW(가상화 NIC 등) > Guest OS** > (Guest) Application”


Docker와 같은 컨테이너 기술은 논리적인 격리 기술을 이용하여, HW를 가상화하지 않고 **OS(Kernel)를 공유하는 방식**이기에 더 효율적으로 리소스를 사용할 수 있다.


![image.png](/assets/img/post/Process와%20Container/1.png)


출처: [https://www.atlassian.com/ko/microservices/cloud-computing/containers-vs-vms](https://www.atlassian.com/ko/microservices/cloud-computing/containers-vs-vms)


####  [Process](https://www.youtube.com/watch?v=xewZYX1e5R8)


OS를 배울 때, 프로세스를 배운다. 프로세스는 프로그램의 인스턴스이며, 실행되고 있는 프로그램을 의미한다. 컨테이너 또한 새로운 리소스가 아닌 논리적으로 **잘 격리된 프로세스**이며, 격리된 환경 속에서 Docker와 같은 컨테이너 엔진을 통해 호스트와 연결된다.


이번 포스트에서는 컨테이너에 들어가기 앞서, 본질인 프로세스와 관련된 명령어를 자세하게 살펴본다.


## 프로세스 살펴보기


### Init Process


리눅스 시스템에서는 **기존의 프로세스를 fork**하는 방식으로 새로운 프로세스가 생성된다. 새로운 명령어를 주고 싶으면 exec()를 통해 코드를 변경한다. COW(Copy on Write)방식이기에 비용이 적게들어, 나쁘지 않은 방식이다. 그렇기에 부팅시 태초의 프로세스가 실행되는데, 이것이 Init Process이다. Init 프로세스는 처음 작동할 때 필요한 프로그램들을 작동시킨다. 


우리는 systemd를 통해 nginx와 같은 서비스를 부팅시 자동실행하도록 설정할 수 있는데, systemd 또한 init 프로세스 중 하나이다. 또한, Init 프로세스는 주로 좀비프로세스를 회수하여 종료시킨다.


**참고자료**

- [https://www.fosslinux.com/134356/systemd-vs-init-decoding-the-linux-boot-process.htm](https://www.fosslinux.com/134356/systemd-vs-init-decoding-the-linux-boot-process.htm)

### 프로세스 생성과정

1. 기존 프로세스에서 **fork**를 진행하여 생성하고, 필요한 동작을 **exec를** 별도의 명령어를 부여한다.
	1. `fork()`가 성공하면 부모프로세스는 자식 PID를 반환받고, 자식은 0을 리턴값으로 받는다.
2. 자식프로세스는 부모 프로세스와 모두 같다. (PCB, Data, Heap, Stack)!
3. 생성된 프로세스는 부모 프로세스의 현재 상태를 PCB로 가지며, 메모리 영역(data)도 그대로 복사한다.
	1. fork를 포함하여, 이전의 소스코드는 복사하지 않는다. fork 이후 코드만 복사한다.
	2. 메모리 영역도 그대로 복사한다. COW(copy on write) 방식을 이용한다.
	3. 프로세스에서 data 참고 방식: heap or stack의 가상주소 > Page Table > 실제 메모리
	4. 프로세스가 소유하고 있는 가상주소는 그대로 복사한다. 다만, 어떤 데이터의 값이 변경되면 실제 메모리 공간을 할당하고, 페이지 테이블은 해당 공간을 가리킨다.

```bash
$vmmap 81959 # 현재 프로세스 ID
Process:         a.out [81959]
Path:            /Users/USER/*/a.out
Load Address:    0x1027e4000
Identifier:      a.out
Version:         0
Code Type:       ARM64
Platform:        macOS
Parent Process:  a.out [81958]

Date/Time:       2024-04-15 21:13:32.728 +0900
Launch Time:     2024-04-15 21:12:39.611 +0900
OS Version:      macOS 14.3.1 (23D60)
Report Version:  7
Analysis Tool:   /usr/bin/vmmap

Physical footprint:         785K
Physical footprint (peak):  785K
Idle exit:                  untracked
----

Virtual Memory Map of process 81959 (a.out)
Output report format:  2.4  -- 64-bit process
VM page size:  16384 bytes
```


### 주요 명령어


여기서는 프로세스와 관련된 명령어를 정리한다.

- ps
	- 프로세스 선택 옵션: -a(all except session leader, not associated terminal),  -e(all), -u(user), -p(pid)
	- 출력 옵션: -f(full), -l(Long), -x(Include **eXecuted** not associated terminal process), H(Hierarchical-tree)
	- —sort {특정 필드}

	```bash
	$ ps aux --sort pid
	USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
	root           1  0.2  0.6 167460 12772 ?        Ss   20:57   0:03 /sbin/init
	root           2  0.0  0.0      0     0 ?        S    20:57   0:00 [kthreadd]
	root           3  0.0  0.0      0     0 ?        I<   20:57   0:00 [rcu_gp]
	root           4  0.0  0.0      0     0 ?        I<   20:57   0:00 [rcu_par_gp]
	root           5  0.0  0.0      0     0 ?        I<   20:57   0:00 [slub_flushwq]
	root           6  0.0  0.0      0     0 ?        I<   20:57   0:00 [netns]
	```

- pstree
	- -p(pid), -u(uid), -a(all arg), -n(Numerical sort - pid sort), -h(highlight - 특정 프로세스 강조)

		```bash
		$ pstree -p $$
		bash(2957)───pstree(5161)
		```

- pgrep
	- -l(name + pid), -u(uid), -n(newest) -c(cnt), -o(oldest)

	```bash
	pgrep -l -u ubuntu bash
	2746 bash
	2957 bash
	```

- top -d(sec), -n(반복횟수), -u(user), -p(pid)

	```bash
	top -d 1 -n 5 -u root
	```

- htop: colorful top

### /proc


실시간으로 커널이 프로세스에 대한 정보를 `/proc` 경로로 업데이트 진행한다. `/proc`가 없다면, `ps`와 같은 명령어도 동작하지 않는다. `/proc/{pid}`에서 cpu, mem을 포함한 HW 정보와, 각 프로세스별 cmd, fd, stat 정보를 확인할 수 있다.


#### Sleep 프로세스를 동작시키고, 정보 확인하기

- terminal 1

```bash
# terminal 1
sleep 10000
```

- terminal 2

```bash
## 프로세스 정보 확인
pgrep sleep
7024

## 프로세스 Dir 확인
tree /proc/$(pgrep sleep) -L 1
/proc/7024
├── arch_status
├── attr
├── autogroup
├── auxv
├── cgroup
├── clear_refs
├── cmdline
├── comm
...
├── timerslack_ns
├── uid_map
└── wchan

## 프로세스 cmd 
cat /proc/$(pgrep sleep)/cmdline ; echo
sleep10000
```


### 마치며


이번 포스팅에서는 컨테이너에 근간이 되는 프로세스에 대해 살펴봤다. 다음 포스트에서는 어떻게 컨테이너를 격리시킬 수 있었는지 다양한 격리방식에 대해 살펴볼 예정이다.

