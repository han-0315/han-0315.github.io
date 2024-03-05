---
layout: post
title: Docker Storage 알아보기
date: 2024-01-22 20:00 +0900 
description: Docker Strage 알아보기
category: [Docker, Storage] 
tags: [UFS, LXC, Docker, Storage] 
image: 
  path: /assets/img/logo/docker/logo.png
pin: false
math: true
mermaid: true
---
Docker Storage 알아보기
<!--more-->


## Docker Storage 알아보기 #1


> Docker이전의 컨테이너 기술인 Linux Container와 관련된 포스트는 [여기](https://www.handongbee.com/posts/LXC(Linux-Container)/)에서 확인할 수 있습니다.


Docker는 유니온 마운트(파일 시스템)과 Docker Hub라는 원격 저장소를 기본적으로 제공함으로 유저들에게 선택받을 수 있었다. **이미지의 기본적인 구조는 UFS 방식을 사용하여 중복된 부분 없이 레이어를 효율적으로 관리할 수 있으며 여러 레이어를 합쳐 이미지로 만드는 것이 가능하다.** Docker의 Layer 아키텍처를 이해하려면 UFS에 대해 알아야한다. UFS는 유니온 파일시스템으로 여러 개의 파일시스템을 하나로 마운트할 수 있다. 즉, 각 Layer의 내용을 논리적으로 병합 가능하다.


이제 UFS부터 자세하게 알아보자


### UFS 필요성


컨테이너에서는 기존 리눅스의 파일시스템에 더불어, Union File system(UFS)를 사용한다. ext4 등 파일시스템은 그 자체로 충분하지만, 컨테이너에서 Layer 아키텍처를 운용하기 위해선 UFS가 필요하다. 


기존의 파일시스템의 주요 문제는 다음과 같다.

1. 비효율적인 공간사용
기존의 파일시스템은, 같은 이미지를 사용해도 인스턴스를 생성할 때마다 물리적 공간이 필요하다. 즉, 기존의 것을 이용하는 것이 아닌 복사하여 사용한다.
2. 부트스트랩 지연

	컨테이너는 고도화된 프로세스이다. 프로세스를 생성하는 방법은 기존 프로세스를 Fork 한다. 즉, 부모 프로세스와 똑같은 복사본을 만든다. 만약, 복사해야 하는 메모리가 크다면 시간이 늘어난다. 


### **UFS 란?**


유니온 파일 시스템은 다른 파일 시스템위에서 작동한다. 파일시스템이 다른 여러 디렉터리를 단일 디렉터리로 마운트한다. 아래의 그림과 같은 결과를 낼 수 있다. UFS의 종류로는 UnionFS, AUFS, OverlayFS가 있다. [링크](https://medium.com/@knoldus/unionfs-a-file-system-of-a-container-2136cd11a779)에서 자세한 사진과 설명을 확인할 수 있다.


**특징에 대해 알아보자**

- 여러 레이어를 논리적으로 병합할 수 있다.(물리적인 행위 없이)
	- 레이어들을 합쳐서 하나의 이미지를 만들 수 있다.
- 읽기 전용 하위레이어, 쓰기 가능한 상위 레이어로 구성된다.
	- 컨테이너 내부에서 여러 행동을 해도 베이스 이미지는 변하지 않는다.
- 하위 레이어부터 읽기 시작하며, 덮어 쓰는 방식이다.
	- 위에 있는 레이어가 이전 레이어들을 덮어쓴다.
- COW(copy on write)
	- 내용을 수정한 경우에만 복사한 뒤 쓰기 작업을 하여 레이어를 추가로 생성한다. 수정하지않는다면, 별도의 복사본 없이 원본의 내용을 참조한다. 결국 수정을 진행해도, 하위 레이어에게 영향을 주지 않는다.

### 직접 확인하기


현재 실행중인 mysql 컨테이너에 대한 UFS를 확인한다. overlay2를 스토리지 드라이버로 사용하고 있으며, 관련 내용은 아래와 같다.

- LowerDir: 이미지(하위)레이어들을 의미하는 디렉터리로, 여러개의 디렉터리가 참조된 것을 확인할 수 있다.
- UpperDir: 컨테이너 레이어에 해당하며, 쓸 수 있는 레이어이다.
- MergeDir: 모든 레이어들을 논리적으로 병합한 디렉터리이다.

하나의 컨테이너를 생성하고 분석하면, 다음과 같이 UFS 파일시스템을 확인할 수 있다.


```bash
$ docker inspect db9d062688a7
[
    {
				...
        "Path": "docker-entrypoint.sh",
        "Args": [
            "mysqld"
        ],
...
"GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/00593ad613202ac2699eb9b09dde4c83aa77608ca0288eb2512c0a71f885dff4-init/diff:/var/lib/docker/overlay2/d437f4b8d67dc5c54dc2bddac279e02fb428441ef289a31ceda1f6058e979428/diff:/var/lib/docker/overlay2/48ab14e09f5c034ad73ad8a9a8d83560b1f81befa23663bd9eb3b10ba1652f9a/diff:/var/lib/docker/overlay2/e5d14078ec49922b7deaa154ad483c2b316691ab89c574196e77bbb79b30c2cd/diff:/var/lib/docker/overlay2/938c8c24954661d9a2b3dfcebbbbe1a26f852dc2075fb6b588fb3a5ce5cd7f4b/diff:/var/lib/docker/overlay2/5fd72d8d14da267b3e09fe0f616f187356112c1146208ae5f7fed4db1217e793/diff:/var/lib/docker/overlay2/616a42343210fd51a81760f6ad701eb149d8eec58a5761763bce6350400ea4a6/diff:/var/lib/docker/overlay2/aa3807c9c668b67e64e22fa4108201444991d9abf773cbe7434c8feabfe6a71e/diff:/var/lib/docker/overlay2/a828814633670fc0530ec9532f97adbc27fcfb2ed70d9b2d70271925528953d7/diff:/var/lib/docker/overlay2/075f6481941c77bcd17a8e9c170af3b935516c2912775e3878457d9c713482ec/diff:/var/lib/docker/overlay2/171447619b0e0b315905d28f84fb77f22c0bb001ff22cf70a22a3ab146a8cd32/diff:/var/lib/docker/overlay2/f21cfcce699c4655ff01b5069be844326d3d7de51a56d8b993444ce7e545533b/diff",
                "MergedDir": "/var/lib/docker/overlay2/00593ad613202ac2699eb9b09dde4c83aa77608ca0288eb2512c0a71f885dff4/merged",
                "UpperDir": "/var/lib/docker/overlay2/00593ad613202ac2699eb9b09dde4c83aa77608ca0288eb2512c0a71f885dff4/diff",
                "WorkDir": "/var/lib/docker/overlay2/00593ad613202ac2699eb9b09dde4c83aa77608ca0288eb2512c0a71f885dff4/work"
            },
            "Name": "overlay2"
        },
```


Merged 디렉터리에 들어가, 컨테이너의 현재 환경을 확인할 수 있다. 아래의 내용은 컨테이너의 파일시스템의 환경과 같다.


```bash
$ pwd
/var/lib/docker/overlay2/00593ad613202ac2699eb9b09dde4c83aa77608ca0288eb2512c0a71f885dff4/merged
$ ls
bin   dev                         entrypoint.sh  home  lib64  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint-initdb.d  etc            lib   media  opt  root  sbin  sys  usr
```


### Docker storage drivers


스토리지 드라이버는 여기서 읽기전용 레이어(Image Layers)가 아닌 위에 R/W 레이어에서 사용된다. R/W 레이어는 컨테이너가 종료되면 그대로 사라지는 레이어이며, 임시 데이터를 저장하는 데 주로 사용된다. 


컨테이너의 수명과 상관없이 보존해야 하는 데이터는 Docker Volume을 이용한다. 


즉, 상태를 저장하고 싶은 데이터는 별도의 Volume을 이용하고 다른 데이터들은 컨테이너 라이프사이클에 맞게 소멸된다. 


**Storage Drivers 특징**

- 스토리지 드라이버의 쓰기 속도는 대체로 기본 파일 시스템 성능보다 낮으며, 특히 CoW 파일 시스템을 사용하는 스토리지 드라이버의 경우 특히 성능이 낮다.
- 스토리지 드라이버는 애플리케이션에서 데이터를 보존하고, 그 과정에서 성능이 우수한 드라이버를 선택한다. 결국, 애플리케이션 맞춤으로 선택한다.

자세한 내용은 [Docker Docs](https://docs.docker.com/storage/storagedriver/)에서 자세한 내용을 확인할 수 있다.


> COW: copy on write 의 약자로 파일 시스템의 알고리즘 중 하나이다. 같은 것이 있으면 그대로 참조해서 사용하고, 만약 수정이 필요한 경우에만 복사한 뒤 수정하여 사용한다. 그렇기에 최대한 쓰기 연산을 적게 하고, 데이터를 효율적으로 관리할 수 있다.


> 스토리지 드라이버를 효과적으로 사용하려면, Docker가 이미지를 빌드하고 저장하는 방법과 컨테이너에서 이러한 이미지를 사용하는 방법을 아는 것이 중요하다. 


### Docker Storage 확인하기

- 스토리지 드라이버 정보 확인

```bash
$ docker info | grep Storage
 Storage Driver: overlay2
```

- 테스트 용 Nginx 컨테이너 배포

```bash
docker run --name nginx -d nginx
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
faef57eae888: Pull complete
76579e9ed380: Pull complete
cf707e233955: Pull complete
91bb7937700d: Pull complete
4b962717ba55: Pull complete
f46d7b05649a: Pull complete
103501419a0a: Pull complete
Digest: sha256:08bc36ad52474e528cc1ea3426b5e3f4bad8a130318e3140d6cfe29c8892c7ef
Status: Downloaded newer image for nginx:latest
d949252e0584572b87b63b008107d103da8df65ac29f2d8c763798852e4703a0
```

- 위에서 받은 이미지 확인하기

```bash
docker image inspect nginx | jq  '.[].RootFS'
{
  "Type": "layers",
  "Layers": [
    "sha256:24839d45ca455f36659219281e0f2304520b92347eb536ad5cc7b4dbb8163588",
    "sha256:b821d93f6666533e9d135afb55b05327ee35823bb29014d3c4744b01fc35ccc5",
    "sha256:1998c5cd2230129d55a6d8553cd57df27a400614a4d7d510017467150de89739",
    "sha256:f36897eea34df8a4bfea6e0dfaeb693eea7654cd7030bb03767188664a8a7429",
    "sha256:9fdfd12bc85b7a97fef2d42001735cfc5fe24a7371928643192b5494a02497c1",
    "sha256:434c6a715c30517afd50547922c1014d43762ebbc51151b0ecee9b0374a29f10",
    "sha256:3c9d04c9ebd5324784eb9a556a7507c5284aa7353bac7a727768fed180709a69"
  ]
}
```

- 아래의 명령어를 실행하여, 위에서 출력된 레이어의 정보를 확인할 수 있다.

```bash
$ ls /var/lib/docker/image/overlay2/layerdb/sha256
```

- 이제 docker에서 UFS를 사용하는 것을 확인해보자
	- LowerDir: 이미지(하위)레이어들을 의미하는 디렉터리로, 보시면 여러개의 디렉터리가 참조된 것을 확인할 수 있음.
	- UpperDir: 컨테이너 레이어에 해당하며, 쓸 수 있는 레이어
	- MergeDir: 모든 레이어들을 논리적으로 병합한 디렉터리
	- WorkDir: 작업 디렉터리

```bash
docker inspect nginx | jq '.[].GraphDriver'
{
  "Data": {
    "LowerDir": "/var/lib/docker/overlay2/fe135358fd9db7c5765617d7ade03e4c79cbba547900294387ae671a87cb8357-init/diff:/var/lib/docker/overlay2/e13d5f94e53ffc5736b195ef36801cca3b96f6ee41c412de0edecef715224839/diff:/var/lib/docker/overlay2/9d50aa2c0106f0f1e5dcd3972886a905a68277e9bdf1450d9a8ac473afb1df85/diff:/var/lib/docker/overlay2/8fe92dce406096e9fee68f4b058598ffcdec82b0780127d03fbca64a208175bf/diff:/var/lib/docker/overlay2/0b2a82f200bc226cc3990510fbf20b90b1636c31a22f56bf1b7a51c841e14e1a/diff:/var/lib/docker/overlay2/6d24ac348a6a007edf2d3a6032ff5c47ce23d6d9be372f872e742f52a55056f6/diff:/var/lib/docker/overlay2/9d752b12c59dbfdddb113e02160f2a8fdba0434cd82e4b3fdc65d1c361fe27b9/diff:/var/lib/docker/overlay2/d288c36a0b8a499f448f7bdf16bc8f5b1908f3aa4679755ae5833381cd847d32/diff",
    "MergedDir": "/var/lib/docker/overlay2/fe135358fd9db7c5765617d7ade03e4c79cbba547900294387ae671a87cb8357/merged",
    "UpperDir": "/var/lib/docker/overlay2/fe135358fd9db7c5765617d7ade03e4c79cbba547900294387ae671a87cb8357/diff",
    "WorkDir": "/var/lib/docker/overlay2/fe135358fd9db7c5765617d7ade03e4c79cbba547900294387ae671a87cb8357/work"
  },
  "Name": "overlay2"
}
```

- 이제 파일을 변경하여, 구조가 어떻게 변경되나 확인한다.
	- TEST 컨테이너 내부에 들어가 TEST 파일을 생성한다.

```bash
$ docker exec -it nginx /bin/bash
root@d949252e0584:/# ls
bin   dev		   docker-entrypoint.sh  home  lib32  libx32  mnt  proc  run   srv  tmp  var
boot  docker-entrypoint.d  etc			 lib   lib64  media   opt  root  sbin  sys  usr
root@d949252e0584:/# touch TEST
root@d949252e0584:/# echo "Hello World" >> TEST
root@d949252e0584:/# ls
TEST  boot  docker-entrypoint.d   etc	lib    lib64   media  opt   root  sbin	sys  usr
bin   dev   docker-entrypoint.sh  home	lib32  libx32  mnt    proc  run   srv	tmp  var
```

- 생성결과, 아래와 같이 UpperDir에 수정이 생기고 mergedDir에 반영된 것을 확인할 수 있다.

```bash
$ ls diff/
etc  root  run  TEST  var

$ ls merged/
bin   dev                  docker-entrypoint.sh  home  lib32  libx32  mnt  proc  run   srv  TEST  usr
boot  docker-entrypoint.d  etc                   lib   lib64  media   opt  root  sbin  sys  tmp   var
```

- 당연히 하위 이미지에는 영향이 없다.

```bash
$ cat lower
l/GGR2QTICSDMGGT435LDE3Q5K57:l/F3Z3MQKSQBX4427T3BLVW7WQ3V:l/NVY3E2UN46ZACWXGXTDH6KZFAH:l/KSAXXJG4HUDPRHNFTREQRW6GQP:l/OEDWEIG5DYVXQIGINX5EX46AKC:l/UD5R4HZWACASR7M2HB7JTCCLTY:l/4GIAVUELBN3MA3FQSY6CCFRLHL:l/2Y2JFZUFNEIAHWYTZPCBMSG23A
```

- `docker diff` 명령어를 통해, 다시 한번 변경사항을 확인한다.
	- C는 변경, A는 추가, D는 삭제를 의미합니다.

```bash
$ docker diff d949252e0584
A /TEST
C /etc
C /etc/nginx
C /etc/nginx/conf.d
C /etc/nginx/conf.d/default.conf
...
```


우리가 생성한 파일인 TEST가 맨위에 `A`로 잘 확인된다.


**이제,** **`docker commit`** **명령어를 통해 이미지를 만들어보자**


현재 우리가 작업한 nginx 컨테이너를 다시 이미지로 만들어본다. 최종적으론 기존 nginx 이미지에 방금 우리가 만든 TEST에 대한 Layer가 합쳐져 이미지가 생성된다.

- 이미지 생성

```bash
$ docker commit d949252e0584 nginx:test
sha256:fbbf85915445603546fb8d734316fd1230b27d0ca9930589e773a85016b556b1
```

- 이미지 확인

```bash
$ docker images
REPOSITORY         TAG           IMAGE ID       CREATED         SIZE
nginx              test          fbbf85915445   8 seconds ago   187MB
nginx              latest        021283c8eb95   6 days ago      187MB
mysql              latest        041315a16183   6 days ago      565MB
```

- 테스트 확인

```bash
docker run -it nginx:test bash
root@725046191e34:/# ls
TEST  boot  docker-entrypoint.d   etc	lib    lib64   media  opt   root  sbin	sys  usr
bin   dev   docker-entrypoint.sh  home	lib32  libx32  mnt    proc  run   srv	tmp  var
root@725046191e34:/# cat TEST
Hello World
```


Upper Layer가 합쳐진 새로운 이미지가 잘 생성된 것을 확인할 수 있다.


### 정리

1. Docker는 Layer를 통해 이미지를 효율적으로 관리할 수 있으며, DockerHub를 통해 누구던지 이미지를 원격 저장소에 올리고 다운받을 수 있다. Docker가 유저들의 선택을 받은 한가지 이유이다.
2. Image를 관리할 때 UFS 방식을 사용한다. UFS 방식은 다른 파일시스템 위에서 동작하는 파일시스템으로 여러 파일시스템을 합칠 수 있다. Docker는 이를 통해 여러 Layer를 하나로 합쳐 관리한다.
3. UFS의 병합 방식에 대해 간단히 설명하면 다음과 같다. UFS는 lower, upper, work, merged 디렉터리가 있으며 lower은 읽기만 가능하며, upper은 읽고 쓰는 것이 가능하다. lower & upper의 모든 이미지를 아래에서부터 덮어쓰는 방식으로 병합하여 merged를 형성한다.

### 참고자료


[https://medium.com/@sudiksha1701/a-deeper-dive-in-docker-4f71d38be570](https://medium.com/@sudiksha1701/a-deeper-dive-in-docker-4f71d38be570)


[https://docs.docker.com/storage/storagedriver/](https://docs.docker.com/storage/storagedriver/)


[https://medium.com/@knoldus/unionfs-a-file-system-of-a-container-2136cd11a779](https://medium.com/@knoldus/unionfs-a-file-system-of-a-container-2136cd11a779)

