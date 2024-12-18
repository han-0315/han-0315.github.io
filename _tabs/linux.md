<!-- ---
layout: page
title: Linux
icon: fas fa-terminal
order: 2
---
아래는 GPT가 작성한 기본 적인 툴이다. 해당 방식은 Redhat 강의와 실무를 해보면서 다시 파악하도록 한다. 또 여기서는 Linux 내용만 다루는 것이 아닌 X86 서버에 대한 기본적인 내용도 포함한다. 즉 OS 아랫단의 내용도 포함한다.

### Linux 기본
- 리눅스란 무엇인가
- 리눅스의 역사
- 커널과 유저 스페이스

### 커맨드 라인 인터페이스 (CLI)
- 기본 명령어
  - 파일 및 디렉토리 관리: `ls`, `cd`, `cp`, `mv`, `rm`, `mkdir`, `rmdir`
  - 파일 내용 보기: `cat`, `less`, `more`, `head`, `tail`
  - 검색 및 필터링: `grep`, `find`, `awk`, `sed`
- 권한 관리
  - `chmod`, `chown`, `chgrp`
- 프로세스 관리
  - `ps`, `top`, `htop`, `kill`, `nice`, `renice`
- 패키지 관리
  - `apt`, `yum`, `dnf`, `pacman`

### 파일 시스템
- 파일 시스템 구조와 계층
- 마운트와 언마운트 (`mount`, `umount`)
- 디스크 관리
  - 파티션 관리 (`fdisk`, `gdisk`)
  - 파일 시스템 생성 (`mkfs`)
  - 디스크 사용량 확인 (`df`, `du`)

### 사용자 및 그룹 관리
- 사용자 계정 생성 및 관리 (`useradd`, `usermod`, `userdel`)
- 그룹 관리 (`groupadd`, `groupmod`, `groupdel`)
- 패스워드 관리 (`passwd`)
- sudo와 권한 상승

### 네트워킹
- 네트워크 설정 파일 이해하기
- 네트워크 인터페이스 관리 (`ifconfig`, `ip`)
- 네트워크 진단 도구
  - `ping`, `traceroute`, `netstat`, `ss`, `nslookup`, `dig`
- SSH를 이용한 원격 접속

### 시스템 관리
- 시스템 부팅 과정과 `systemd`
- 서비스 및 데몬 관리 (`systemctl`)
- 로그 관리 (`syslog`, `journalctl`)
- 시간 및 날짜 설정 (`timedatectl`)

### 보안
- 방화벽 설정
  - `iptables`, `nftables`, `ufw`, `firewalld`
- SELinux와 AppArmor
- 파일 시스템 암호화
- SSH 보안 설정

### 스크립팅 및 자동화
- 셸 스크립트 작성
- 환경 변수와 별칭
- 작업 스케줄링
  - `cron`, `at`
- 자동화 도구
  - `expect`, `Ansible`
- 디버깅 도구
  - `gdb`, `strace`, `ltrace`

### 고급 주제
- 시스템 성능 모니터링 및 튜닝
  - `vmstat`, `iostat`, `sar`, `perf`
- 리눅스에서의 네트워크 파일 시스템
  - NFS, Samba

### 기타
- 리눅스에서의 데이터 백업 및 복원 (`tar`, `rsync`)
- 텍스트 편집기 사용법
  - `vim`, `nano` -->