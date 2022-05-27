---
layout: single
title: "[Database/Oracle] VMware - Oracle Linux 7.3 설치"
excerpt: "Oracle 스터디 1차시"
categories: Database
tag: [VMware, Database, DB, 데이터베이스, Linux, 리눅스, OS 설치, Virtual Machine, 가상환경, SCSI, Disk, Partitioning, 파티셔닝, Oracle, 오라클, mount, LVM, SWAP]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

처음으로 데이터베이스에 대한 스터디를 시작했는데요..!  
  
이번 포스팅에서는 VMware에 Virtual machine을 만들고 Oracle을 설치하기 위해 리눅스 OS를 먼저 설치할 것입니다~  
  


  
## 1. 설치 과정 (VMware Workstation)

### Create a New Virtual Machine
설치 과정은 아래와 같습니다.  
  
1. `Custom (advanced)` > `Workstation 16.2.x`
2. `Installer disc image file (iso):` 해당 파일 선택 > 네이밍 규칙대로 이름 지정
3. Number of processors : `1`, Number of cores per processor : `2`
4. Memory for this virtual machine : `3072` MB
5. `Use network address translation (NAT)`
6. I/O Controller Types : `LSI Logic (Recommended)`
7. Virtual disk type : `SCSI (Recommended)`
8. Disk : `Create a new virtual disk`
9. Maximum disk size (GB) : `100GB`, `Store virtual disk as a single file`
    - 1MB x 100개와 100MB x 1개의 속도를 비교해 본다면 100MB짜리 하나로 돌아가는 디스크가 더 빠르다는 것을 생각할 수 있다. 그래서 속도를 생각하여 `Store virtual disk as a single file`로 선택하였다. ⇒ **RAID** 관련

### Oracle Linux 7.3 설치

1. 기본 값으로 설치 진행
2. Software Selection > `Server with GUI` > `Compatibility Libraries`, `Development Tools` 체크
    - Virtualization Host : OS를 가상으로 여러 개 올릴 때, 최소한 설치. ex) log server, 인증 서버
    - Compatibility Libraries : 호환되는 라이브러리 설치
    - Development Tools : 개발 툴에서 필요한 라이브러리
3. KDUMP > `Enable kdump` 체크 해제
    - KDUMP : 복구할 때 사용
4. Network & Host name > ON > Configure > IPv6 Settings : Method : `Ingnore` > IPv4 Settings : Method : `Manual`, Address : `192.168.100.130`, Netmask : `255.255.255.0`, Gateway : `192.168.100.2`, DNS servers :  `8.8.8.8, 8.8.4.4`
    - Hardware Address
        - MAC Address
        - 16진수
        - 고유 주소
    - IP Address : 192.168.182.128
    - MTU
        - 패킷 단위
        - 최대 9000 byte
        - 네트워크 설정할 때 사용
    - DNS servers
        - 도메인 주소 ↔ IP 주소
        - Google DNS Server IP : 8.8.8.8 / 8.8.4.4
        - KT DNS Server IP : 168.126.63.1
5. Sucurity Policy > Apply security policy : `OFF`
6. Installation Destination > Local Standard Disks 클릭하지 말고 > 아래로 스크롤 > Other Storage Options : `I will configure partitioning` 체크 > `Done` 클릭 > `LVM` > `swap`, `/boot`, `/` 메모리 아래 사진과 같이 추가 > `Accept` > `Begin Installation`

    ![Untitled](https://user-images.githubusercontent.com/100563973/169363382-bda62668-7ba8-4db9-8812-77348d22a740.png)
    
    - LVM(Logical Volume Manager)
        - 큰 디스크 용량이 필요하여 작게 쪼개어진 용량을 모은 볼륨 디스크
        - 2 GB 이상되면 LVM으로 표시
        - 기본 partition
            - /(root)
            - SWAP
                - 물리적인 메모리가 부족할 경우, 임시적으로 사용
                - FIFO 방식에 의해 마지막 메모리를 SWAP으로 넘김
                - 메모리의 1.5배~2배
        - 추가 partition
            - /boot : 부팅 시, OS 업그레이드 시
            - /var : 로그 파일 저장 위치
            - /home : 사용자 홈 디렉토리
            - /DATA
                - 명확한 구분을 위해 대문자 사용
                - 윈도우는 case-insensitive. 리눅스는 case-sensitive
7. Root Password

## 2. Study 내용

### 네이밍 규칙
- 빈칸 없이
- 구분자: _(언더바)
- ORA_설치버전_비트 수_목적
    - ex) ORA_7.3_x64_ora_11

### 32bit/64bit
- 응용프로그램 32비트 사용 이유
    - 메모리, 디스크, 채널

### Linux 버전
- personal
- workstation : 특정 업무에 특화
- server : 제일 많은 기능과 제일 비쌈
    - file 공유
    - print 공유

    
### 작업 환경
1. dev : 최소 사양
2. stg : 운영과 비슷한 환경 (하드웨어 사양은 반으로)
3. qa : 운영과 똑같이
4. prod
> qa와 prod는 cpu는 따로 쓰지만 디스크는 같이 씀
> 
> ⇒ 비용 문제 : storage, DB license, backup, 모니터링, 유지보수, 인력
> 

### Database
1. WEB : apache
    - 정적 : 이미지, text, 영상
2. WAS : Tomcat
    - 동적 : Query Data
3. DB : Maria
  
### Oracle
- Oracle이 MySQL 인수. 점유율 50% 이상  
    - 아래 링크를 확인해 보면 Database 랭킹을 확인해 볼 수 있다.  
        [DB-Engines Ranking](https://db-engines.com/en/ranking)
- sun이라는 회사를 Oracle이 매수
    - Solaris OS + CPU(Storage) + M/B + Java
    - DB + Server
- Exadata

## 3. 추가로 알아야 할 부분

- **RAID**
- **LVM**
- 파일시스템
    - **FAT (File Allocation Table)** : 디지털 카메라 등에 장착되는 대부분의 메모리 카드와 수많은 컴퓨터 시스템에 널리 쓰이는 컴퓨터 파일 시스템 구조
    - **NTFS (New Technology File System)** : Microsoft Windows의 파일 시스템으로, MS-DOS부터 쓰인 FAT를 대체하기 위해 1993년 Windows NT 3.1과 함께 발표했다.
- **DHCP (Dynamic Host Configuration Protocol)** : 호스트의 IP주소와 각종 TCP/IP 프로토콜의 기본 설정을 클라이언트에게 자동적으로 제공해주는 프로토콜
    - Dynamic IP
    - Static IP
        - Server는 서비스를 제공하며 연결되어 있는 제품이 많기 때문에 IP가 변경되면 안됨


## 4. 다음 과제

- [ ] 네 가지 인터페이스 방식에 대한 차이 공부해 오기  
    - [ ] IDE(Intelligent Drive Electronics)
    - [ ] SCSI(Small Computer System Interface)
    - [ ] SATA(Serial ATA)
    - [ ] NVMe
- [ ] LVM 공부해 오기  
    - [[소개] LVM(Logical Volume Manager) - 개념](https://tech.cloud.nongshim.co.kr/2018/11/23/lvmlogical-volume-manager-1-%EA%B0%9C%EB%85%90/)
    
- [ ] oracle linux 설치해 오기 - 여러 번 설치 연습 필요

