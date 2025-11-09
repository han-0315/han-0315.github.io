---
layout: post
title: Completely Fair Scheduler 살펴보기
date: 2025-11-09 09:00 +0900 
description: Completely Fair Scheduler 살펴보기
category: [Linux, 스케줄링] 
tags: [k8s, linux, cfs, os] 
pin: false
math: true
mermaid: true
---
Linux 스케줄링 방식인, CFS 살펴보기
<!--more-->


쿠버네티스에서 CPU 리소스 관리 방식을 이해하려면, 리눅스 스케줄링에 대한 지식부터 알아야하기에 이곳에서는 CFS에 대해 자세히 알아본다. 


## CFS 스케줄링


CFS는 Completely Fair Scheduler의 약자로, O(1)의 **공정성 문제를 해결**하기 위해 도입된 알고리즘이다. 현재 리눅스에서 사용하는 스케줄링 방식이다.


우선, O(1) 방식에는 어떤 문제가 있었는지 살펴보자.


(1) 시간을 분배하는 값이 고정적이라 우선순위에 비례하는 공정성이 부족하다. 


`ex) priority 39 = 100ms, p1 = 10ms p0 = 5ms` 


(2) 기아 상태 발생 가능성


정해진 타임 슬라이스, 혹은 그 이상의 시간동안 CPU를 할당받지 못하는 경우가 존재함


ex) 새로운 task가 계속 생겨나면, expired queue에 있는 프로세스는 기아 상태가 될 수 있다.


위와 같은 문제로 본인의 프로세스가 설정해둔 우선순위에 비해 덜 할당받는다는 체감 불공정이 나왔다. 그렇다면 CFS는 이런 문제를 어떻게 해결했을 지 아래에서 자세히 알아보자.


### 스케줄링 방식


CFS에서는 실시간 처리가 필요한 Task와 아닌 Task를 별도로 분리한다. 우리는 공정성을 살펴볼거니 실시간 처리가 필요하지 않은 Normal Task 스케줄링 방식에 대해 살펴본다.


(당연히 실시간 처리가 필요한 경우, Normal 보다 우선권을 가진다.)


CFS는 공정함을 유지하기 위해, 크게 2가지 원칙을 설정했다.


**(1) ideal, precise multi-tasking CPU**


**(2) 각 스케줄링 기간에 모든 Task가 한번씩은 실행될 수 있게 한다. (물론 이를 완벽히 보장하진 않는다.)**


(1)은 kernel docs에 나와있는 설명으로, 내가 이해한 바로는 모든 Task를 똑같이 동시에 돌릴 수 있다고 가정한다는 것이다. 즉 2개의 Task가 있다면 동시에 50%씩 cpu를 사용할 수 있다.


하지만, 실제 CPU 코어가 1개라면 이것은 불가능하다. 그렇기에 가상 실행시간(VR)을 도입한다. VR의 의미는 “이상적인 환경에서는 지금쯤 이 수치만큼 실행됐어야 한다”라는 기대치이다.






스케줄링을 진행할 때 VR이 가장 작은 Task를 반복적으로 선택하여 결과적으로 VR이 비슷해지도록 만든다.


스케줄링된 Task는 리스케줄링될 때, CPU를 사용한만큼 VR에 점유시간을 더한다. 






1,2를 진행하면 이상적으로는 모든 프로세스가 CPU를 점유하는 시간은 동일하다. 하지만 실제로는 Task에 우선순위가 존재한다. 이를 반영하기 위해 CFS에서는 VR이 증가하는 시간에 가중치를 더한다.


`vruntime += actual_runtime * NICE_?_LOAD / se->load.weight`


Task의 weight가 클수록 분모에 있기에 vruntime이 가중치에 비례하여 천천히 증가한다.






한번의 스케줄링 기간동안 특정 Task가 할당받는 이상적인 분배 시간은 아래와 같다.


`time_slice = sched_period * (se->load.weight / cfs_rq->load.weight)`
ex) 우선순위가 1, 3인 Task가 존재하면 실제 1000ms동안 cpu를 점유하는 시간은 250ms, 750ms이다.


#### 최소 보장 시간


하나의 Task가 한번의 틱마다 너무 적게 실행된다면, 그에 따른 컨텍스트 스위칭 비용이 더 커진다. CFS에서는 하나의 Task가 CPU를 점유할 때 적어도 이 시간만큼은 리스케줄링되지 않도록 보장하는 시간을 정해놨다.


#### Scheduling Latency


스케줄링 지연시간은 스케줄 단위로, 모든 Task가 한번씩은 실행될 수 있는 시간이다. 만약 스케줄링 지연시간(Scheduling Latency)이 `Task의 개수 * 최소 보장시간`보다 작으면 `Task의 개수 * 최소 보장시간`으로 바꾼다. 결국 하나의 turn에는 모든 task가 공평하게 한번씩은 실행되도록 유도한다. 


#### 신규 Task


새로 생성된 Task들은 VR이 0으로 초기화되기 때문에, 기존의 task가 오랫동안 실행되서 VR이 상당히 높다면 새로 생성된 Task가 너무 길게 실행되며, 이는 새로운 불평등을 초래한다. 그래서 CFS에서는 초기값으로 0이 아닌 기존의 VR중 제일 작은 값을 부여한다.


```c++
static void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se, int flags)
{
    if (!(flags & ENQUEUE_WAKEUP || flags & ENQUEUE_WAKING))
        se->vruntime += cfs_rq->min_vruntime;
    update_curr(cfs_rq);
}

```


### Red Black tree


위의 VR 방식의 스케줄링을 구현하는 자료구조로 Red Black Tree를 사용한다.


매 Tick마다 가장 작은 VR을 선택하면 되기에, Heap을 사용하면 되는 것 아닌가라는 생각이 들 수 있다. 스케줄링에서는 Task를 제거해야할 일이 많다. ex) block i/o, sleep, cgroup


그렇기에 heap보다는 self balancing tree의 일종인 RB tree(AVL)를 사용한다.


![image.png](/assets/img/post/Completely%20Fair%20Scheduler/1.png)


출처: [https://www.geeksforgeeks.org/dsa/introduction-to-red-black-tree/](https://www.geeksforgeeks.org/dsa/introduction-to-red-black-tree/)


### 마치며


kubelet이 CPU를 제어하는 방식을 이해하려면, 리눅스 스케줄링에 대한 지식부터 알아야하기에 이곳에서는 CFS 스케줄링에 대해 알아봤다. 다음 포스팅에서는 쿠버네티스 노드(kubelet)에서 관리하는 리소스와 관리하는 방식에 대해 알아보려고 한다.

