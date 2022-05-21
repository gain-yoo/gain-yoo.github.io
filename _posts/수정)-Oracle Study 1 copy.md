---
layout: single
title: "[Database] Oracle Study 1차시"
excerpt: "VMware - Linux OS 설치"
categories: Database
tag: [VMware, Database, DB, 데이터베이스, Linux, 리눅스, OS 설치, Virtual Machine, 가상환경, SCSI, Disk, Partitioning, 파티셔닝, Oracle, 오라클, mount, LVM, SWAP]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

<aside>
💡 RedHat 공식 문서

</aside>

## 1. 설치 과정 (VMware Workstation)

### Create a New Virtual Machine

`Custom (advanced)` > `Workstation 16.2.x` > `Installer disc image file (iso):` 해당 파일 선택 > 네이밍 규칙대로 이름 지정 > Number of processors : `1`, Number of cores per processor : `2` >  Memory for this virtual machine : `3072` MB > `Use network address translation (NAT)` > I/O Controller Types : `LSI Logic (Recommended)` > Virtual disk type : `SCSI (Recommended)` > Disk : `Create a new virtual disk` > Maximum disk size (GB) : `100GB`, `Store virtual disk as a single file` (이게 더 빠름. 아까 100M x 1과 같은 이치)

### Linux OS 설치

1. 기본 값으로 설치 진행
2. Software Selection > `Server with GUI` > `Compatibility Libraries`, `Development Tools` 체크
    - Virtualization Host : OS를 가상으로 여러 개 올릴 때, 최소한 설치. ex) log server, 인증 서버
    - Compatibility Libraries : 라이브러리 호환되는 거 설치
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

### **CPU**

1. 원가
2. 성능 유지 또는 상승

### **Linux 버전**

- personal
- workstation : 특정 업무에 특화
- server : 제일 많은 기능과 제일 비쌈
    - file 공유
    - print 공유

### **작업 환경**

1. dev : 최소 사양
2. stg : 운영과 비슷한 환경 (하드웨어 사양은 반으로)
3. qa : 운영과 똑같이
4. prod

> qa와 prod는 cpu는 따로 쓰지만 디스크는 같이 씀
> 
> 
> ⇒ 비용 문제 : storage, db license, backup, 모니터링, 유지보수, 인력
> 

### Database

1. WEB : apache
    - 정적 : 이미지, text, 영상
2. WAS : Tomcat
    - 동적 : Query Data
3. DB : Maria

### Oracle

- Oracle이 MySQL 인수. 점유율 50% 이상
    
    [DB-Engines Ranking](https://db-engines.com/en/ranking)
    
- sun이라는 회사를 Oracle이 매수
    - Solaris OS + CPU(Storage) + M/B + Java
    - DB + Server
- Exadata

## 3. 추가로 알아야 할 사항

Q. 1M x 100개 or 100M x 1개 속도 빠른 것은?

⇒ 100M x 1개

⇒ **RAID** 관련

- 파일시스템
    - FAT (File Allocation Table) : 디지털 카메라 등에 장착되는 대부분의 메모리 카드와 수많은 컴퓨터 시스템에 널리 쓰이는 컴퓨터 파일 시스템 구조
    - NTFS (New Technology File System) : Microsoft Windows의 파일 시스템으로, MS-DOS부터 쓰인 FAT를 대체하기 위해 1993년 Windows NT 3.1과 함께 발표했다.
- DHCP (Dynamic Host Configuration Protocol) : 호스트의 IP주소와 각종 TCP/IP 프로토콜의 기본 설정을 클라이언트에게 자동적으로 제공해주는 프로토콜
    - Dynamic IP
    - Static IP
        - Server는 서비스를 제공하며 연결되어 있는 제품이 많기 때문에 IP가 변경되면 안됨
- **LVM**

## 4. 다음 과제

- [ ]  네 가지 인터페이스 방식에 대한 차이 공부해 오기
    - IDE(Intelligent Drive Electronics)
        - HDD, CD-ROM 등을 연결하는 40핀의 **병렬** 인터페이스 규격, 최대 16bit 전송
        - 1개의 채널에 장치 2개씩 연결, 최대 4개
        - 마스터, 슬레이브
    - SCSI(Small Computer System Interface)
        - **병렬**
        - IDE 방식으로 불가능했던 데이터 처리 속도 한계 해소
        - RAID 기능 등 부가 기능 사용 가능
        - 가격 대비 최고 성능
        - 다중 장치 지원 및 확장된 케이블 길이, 더 빠른 전송 속도 및 마더 보드 또는 사용 가능한 커넥터에 다중 호스트 어댑터 설치 가능한 기능으로 가장 자주 사용됨
        - hot plug
        
        
    - SATA(Serial ATA)
        - 하드디스크나 CD-ROM과 같이 데이터 전송이 주 목적
        - **직렬** 인터페이스 이용한 규격으로 독립적 작용
        - 대역폭 높음
    - NVMe
        - 컴퓨터의 고속 PCIe 버스를 통해 SSD와 같은 플래시 메모리 저장 장치에 있는 데이터에 빠르게 액세스할 수 있게 해주는 전송 프로토콜
        - **병렬**, 비휘발성
- [ ]  LVM 공부해 오기
    
    [[소개] LVM(Logical Volume Manager) - 개념](https://tech.cloud.nongshim.co.kr/2018/11/23/lvmlogical-volume-manager-1-%EA%B0%9C%EB%85%90/)
    
- [ ]  oracle linux 설치해 오기 - 여러 번 설치 연습
- 책 읽어오기
    - [ ]  Part 03 Chapter 01 전체
    - [ ]  Part 03 Chapter 02 Section 06 까지
