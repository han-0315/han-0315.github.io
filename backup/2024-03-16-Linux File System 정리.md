---
layout: post
title: Linux File System 정리
date: 2024-03-16 09:00 +0900 
description: Linux File System 정리
category: [Linux, File system] 
tags: [System, SE, Linux, Network]
pin: false
math: true
mermaid: true
---
Linux File System 정리
<!--more-->


## 파일시스템


컴퓨터의 하드디스크는 주로 SSD를 이용한다. 같은 SSD이지만, 사용하는 파일시스템은 다르다. 맥북에서는 APFS라는 파일시스템을 사용하는 반면 리눅스에서는 EXT4, XFS 등 다른 파일시스템을 사용한다. 파일시스템은 어떤 과정으로 실제 디스크에 데이터를 저장하는지 EXT 유형으로 알아보자.


**[Mac Book]**


```bash
$diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *500.3 GB   disk0
   1:             Apple_APFS_ISC Container disk1         524.3 MB   disk0s1
   2:                 Apple_APFS Container disk3         494.4 GB   disk0s2
   3:        Apple_APFS_Recovery Container disk2         5.4 GB     disk0s3
```


**[Amazone Linux]**


```bash
$ df -Th
Filesystem     Type      Size  Used Avail Use% Mounted on
devtmpfs       devtmpfs  468M     0  468M   0% /dev
tmpfs          tmpfs     477M     0  477M   0% /dev/shm
tmpfs          tmpfs     477M  408K  476M   1% /run
tmpfs          tmpfs     477M     0  477M   0% /sys/fs/cgroup
/dev/xvda1     xfs       8.0G  1.7G  6.3G  22% /
tmpfs          tmpfs      96M     0   96M   0% /run/user/0
```


## ext


나는 많은 파일시스템 중 ext를 선택한 이유는 현재 많은 리눅스 배포판에서 사용 중인 파일시스템이기에 그렇다.


우선 ext에 대해 알아보면, EXT는 Extended file system의 약자로 리눅스 파일시스템 중 하나이다. 1992년 4월 처음 리눅스에 도입되었고, 오늘날 많은 배포판에서 사용된다. 


### 구조


ext 파일시스템은 처음에 마운트할 때, 물리 디스크를 여러 개의 파티션으로 분리하여 관리한다. 하나의 파티션은 여러 Block Group으로 이루어져있다. 하나의 Block Group은 그림과 같이 Super Block, Inode, Data Block 등으로 구분된다.


즉, “`Disk` > `Partitions` > `Block Groups` > `Super Block, Inode, … ,Data Block`”으로 이뤄져있다. 실제 데이터는 Data Block에서 기록된다.


![structure.webp](/assets/img/post/Linux%20File%20System%20정리/2.webp)


사진 출처: [[https://recoverhdd.com/blog/the-ext-ext2-ext3-ext4-filesystem.html](https://recoverhdd.com/blog/the-ext-ext2-ext3-ext4-filesystem.html)]


우리는 Block Group 아래에서 구조를 살펴볼 것이다. 주로 나오는 요소들에 대한 설명을 하면 다음과 같다.

- `file_system_type`: 파일 시스템을 설명하는 구조체이다.
	- 파일시스템 유형의 이름과 mount 방식에 대한 정보를 가지고 있다.
- `super_block`: 마운트 당시 생성 및 초기화되며, 디스크와 파일시스템에 대한 정보를 가지고 있다.
- `Inode`: 파일의 메타데이터 역할을 하며, 파일마다 각각 Inode를 가지고 있다.
- `data block`: 실제 데이터가 존재하는 Block이다.
- `dentry`: 디렉터리와 관련된 정보를 제공한다.
- `File`: 작업에서 열린 파일과 관련된 정보를 제공한다.
	- `file operation` 정보도 넘겨주며 읽기, 쓰기 등이 있다.
- `task_struct`: 하나의 작업으로, 작업에서 파일을 생성할 경우 위의 파일객체를 참조한다.

### 파일 찾는 방법


이제 파일을 찾는 방식에 대해 한번 살펴보자. 우선 파일에 대한 메타정보는 inode에 있으니, inode를 찾아야 파일의 데이터를 찾을 수 있다. 또, 디렉터리는 data block에 폴더에 존재하는 파일의 inode 정보를 남겨둔다고 한다. 이를 통해 우리는 재귀적으로 파일을 찾을 수 있다.


우리가 만약, /test.c 경로에 위치한 파일을 찾는다면 다음과 같은 방식으로 진행된다. 

1. inode Table을 확인하여 ‘`/`’에 대한 inode의 정보를 확인한다.
2. (1)에서 확인한 정보로 data block을 찾아간다.
	1. 해당 데이터는 디렉터리이므로, 디렉터리에 존재하는 파일들의 inode 번호를 가지고 있다.
3. (2)에서 확인한 test.c의 inode 번호를 통해, inode table에서 inode 정보를 확인한다.
4. (1)과같이 inode 정보를 통해 data block을 찾아간다.

만약 `/a/b/c.c` 경로에 존재하는 파일이라면, 아래와 같은 순서로 이뤄진다. 


![dirinode.jpg](/assets/img/post/Linux%20File%20System%20정리/4.jpg)


[그림 출처: [https://pages.cs.wisc.edu/~bart/537/lecturenotes/s25.html](https://pages.cs.wisc.edu/~bart/537/lecturenotes/s25.html)]

