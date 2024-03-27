---
layout: post
title: Linux Link 정리
date: 2024-03-16 09:00 +0900 
description: Linux Link 정리
category: [Linux, Link] 
tags: [Linux, Link, File, HardLink, SoftLink] 
pin: false
math: true
mermaid: true
---
Linux Link 정리
<!--more-->


## 링크


링크는 바로가기라고 생각하면 된다. 어떤 파일이 있는데, 해당 파일 위치로 가지않고 바로 실행할 수 있게 한다. 이것이 링크이다. 아래 링크의 종류를 살펴보며 자세히 알아보자.


링크의 종류는 하드 링크, (심볼릭)소프트 링크 2개가 있다. 


![Untitled.png](/assets/img/post/Linux%20Link%20정리/1.png)


FileName / HardLink / SoftLink : 디렉터리에서 보여지는 범위


Data, Data(Pointer) : 실제 하드디스크에 저장된 값


**[하드링크]**


원본 데이터와 직접적으로 연결시켜 만든다. 위에 그림에서 하드링크는 원본 데이터의 inode를 가리킨다.


**[심볼릭 링크]**


하드링크와는 다르게, 자신의 Inode가 존재한다. 즉 아예 다른 파일이며, 하드디스크에서는 원본 파일의 inode를 가리키는 값이 들어간다. 즉 포인터와 유사하며, 바로가기처럼 원본 파일에 대한 참조데이터를 가지고 있다. 


### Inode 구조


Inode의 연산의 함수명을 보면, 우리가 자주 사용하는 명령어와 유사하다. 특히 mkdir와 같은 경우 자주 사용되는 리눅스 명령어이다.


```c
struct inode_operations {
        int (*create) (struct user_namespace *, struct inode *,struct dentry *, umode_t, bool);
        struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
        int (*link) (struct dentry *,struct inode *,struct dentry *);
        int (*unlink) (struct inode *,struct dentry *);
        int (*symlink) (struct user_namespace *, struct inode *,struct dentry *,const char *);
        int (*mkdir) (struct user_namespace *, struct inode *,struct dentry *,umode_t);
        int (*rmdir) (struct inode *,struct dentry *);
        int (*mknod) (struct user_namespace *, struct inode *,struct dentry *,umode_t,dev_t);
        int (*rename) (struct user_namespace *, struct inode *, struct dentry *,
                       struct inode *, struct dentry *, unsigned int);
        int (*readlink) (struct dentry *, char __user *,int);
        const char *(*get_link) (struct dentry *, struct inode *,
                                 struct delayed_call *);
        int (*permission) (struct user_namespace *, struct inode *, int);
        struct posix_acl * (*get_acl)(struct inode *, int, bool);
        int (*setattr) (struct user_namespace *, struct dentry *, struct iattr *);
        int (*getattr) (struct user_namespace *, const struct path *, struct kstat *, u32, unsigned int);
        ssize_t (*listxattr) (struct dentry *, char *, size_t);
        void (*update_time)(struct inode *, struct timespec *, int);
        int (*atomic_open)(struct inode *, struct dentry *, struct file *,
                           unsigned open_flag, umode_t create_mode);
        int (*tmpfile) (struct user_namespace *, struct inode *, struct file *, umode_t);
        int (*set_acl)(struct user_namespace *, struct inode *, struct posix_acl *, int);
        int (*fileattr_set)(struct user_namespace *mnt_userns,
                            struct dentry *dentry, struct fileattr *fa);
        int (*fileattr_get)(struct dentry *dentry, struct fileattr *fa);
};
```


### 테스트 해보기


**[테스트 폴더 생성 및 테스트 파일 생성]**


`ls -ali` 명령어를 통해 inode의 ID도 출력한다. 우리가 실습할 테스트 파일의 inode_id는 **18508464**인 것을 확인할 수 있다.


```bash
~ % mkdir link_test
~ % cd link_test 
link_test % ls
link_test % echo "Link test file" > test
link_test % ls -ali     
total 8
18508460 drwxr-xr-x    3 handongmin  staff    96 12  7 23:55 .
   38137 drwxr-xr-x@ 111 handongmin  staff  3552 12  7 23:55 ..
18508464 -rw-r--r--    1 handongmin  staff    15 12  7 23:55 test

```


**[링크 생성]**


가리키고 있는 inode number를 확인하면 `18508464`로 test와 hardlink는 같은 것을 확인할 수 있다.


반면 소프트링크는 다르다. 위에서 봤던 사진과 동작방식이 일치하는 것을 확인할 수 있다.


```bash
link_test % ln -s test softlink
link_test % ln test hardlink
link_test % ls -ali
total 16
18508460 drwxr-xr-x    5 handongmin  staff   160 12  7 23:56 .
   38137 drwxr-xr-x@ 111 handongmin  staff  3552 12  7 23:55 ..
18508464 -rw-r--r--    2 handongmin  staff    15 12  7 23:55 hardlink
18508477 lrwxr-xr-x    1 handongmin  staff     4 12  7 23:56 softlink -> test
18508464 -rw-r--r--    2 handongmin  staff    15 12  7 23:55 test

```


**[원본 파일 삭제]**


```bash
link_test % rm test 
link_test % ls
hardlink	softlink
```


**[소프트 링크]**


```bash
link_test % cat softlink
cat: softlink: No such file or directory

```


**[하드 링크]**


```bash
link_test % cat hardlink 
Link test file
```


이름이 같은 새로운 파일 생성, 생성 후 소프트 링크로 다시 cat 명령어를 사용하면 새로운 파일에 대한 정보를 얻을 수 있다.


```bash
link_test % echo "new Test" > test
link_test % cat softlink 
new Test
```


test 파일을 지웠을 때 링크를 따라간 결과 hard는 그대로 파일을 읽어오지만, soft는 아닌 것을 확인할 수 있다. 만약 같은 이름의 파일을 아래와 같이 하나 더 추가하면 softlink는 이를 가리키게 된다.

