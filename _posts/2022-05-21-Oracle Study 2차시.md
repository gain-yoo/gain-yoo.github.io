---
layout: single
title: "[Database] Oracle Study 2차시"
excerpt: "VMware - Oracle 설치"
categories: Database
tag: [VMware, Database, DB, 데이터베이스, Linux, 리눅스, OS 설치, Virtual Machine, 가상환경, SCSI, Disk, Partitioning, 파티셔닝, Oracle, 오라클, mount, LVM, SWAP]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

# 1. 지난 과제

##  IDE / SATA / SCSI / NVMe

- 비교 표
  
	| IDE | SATA | SCSI | NVMe |
	| --- | --- | --- | --- |
	| 40핀, 병렬 | 7핀, 직렬 | 50/68핀, 병렬 | 병렬 |
	| 133.3MB/S | SATA3=600MB/S,  SATA2=300MB/S,  SATA1=150MB/S | 320MB/S | 읽기 3GB/S |
	| 1개의 채널에 장치 2개씩 연결, 최대 4개 |  | 서버, 워크스테이션용 | 서버, 데스크탑 |
	| 가장 오래됨 |  | 다른 애들은 메인보드에 포함되어 있지만 얘만 별도 ⇒ 메인보드(BIOS)의 통제를 받지 않음 |  |
	| 1m 이내 | 1m 이내 | 길다 | 바로 |
	| 480/5600/7200rpm | 데이터 바로 연결 | 데이터 바로 연결 | 데이터 바로 연결 |
- SATA+SCSI = SAS
- 메가 이진 바이트(Mega binary byte)
    - MiB 1024 *(2의 3승)*
    - MB 1000 *(10의 3승)*
- USB 속도
    - USB 1.0의 속도는 1.5Mbps - 12Mbps(1.5MB/s)
    - USB 2.0속도는 무려 480Mbps(60MB/s)
    - USB 3.0 최대 5Gbps(625MB/s)의 속도
    - USB 3.0과 USB 3.1은 2배 곧 최대 10Gbps(1,250MB/s)

## LVM

- 하나의 볼륨으로 묶고 쪼개기 가능
- 데이터 복구 시 물리적/논리적 방법
    - 물리적 복구 SCSI
    - LVM 소프트웨어적 복구
- PE < Physical Volume < Volume Group
- Logical Volume은 디렉토리에 mount해서 사용할 수 있는 Volume으로, Physical Volume에 포함되어 있는 여러 PE들을 합쳐서 구성
    
    

## Terminal

- hostname 지정 : Case Insensitive
- 시스템 종료 : `shutdown -h now` (hlt의 약자)
- 암호 변경 시, `Authentication token manipulation error`
    - **user** 본인 암호 변경할 때는 암호 규칙을 따라줘야 함
    - 하지만 **root**로 다른 user 암호 변경할 때는 암호 규칙 안따름
    
    ```jsx
    [giyoo@study_gain ~]$ su
    	Password: 
    [root@study_gain giyoo]# passwd
    	Changing password for user root.
    	New password: 
    	BAD PASSWORD: The password is a palindrome
    	Retype new password: 
    	passwd: all authentication tokens updated successfully.
    [root@study_gain giyoo]# passwd giyoo
    	Changing password for user giyoo.
    	New password: 
    	BAD PASSWORD: The password is a palindrome
    	Retype new password: 
    	passwd: all authentication tokens updated successfully.
    [root@study_gain giyoo]# exit
    	exit
    ```
    

# 2. Oracle 설치
  
> 💡 **vi 편집기 옵션 알아두기**

1. 해당 vm 오른쪽 마우스 > Settings > Options > Shared Folders > `Enable this share`
    - `Enable this share` : Read & Write
    - Read only
2. database 디렉토리 mount
    - mount 되지 않아서 scp로 직접 넣어줌
    - ip 네트워크 설정을 맞춰준 다음에 임의로 /(root) 하위에 `/abc` 디렉토리 생성하여 복붙
    - 디렉토리 권한 rwx
        - x 권한이 없으면 디렉토리 접근 불가
3. 리눅스 종료 명령어
    - init
    - reboot
    - halt
    - shutdown
    - sync : 메모리에 있는 것을 디스크로 동기화
        - shutdown 하기 전에 sync를 해줘야 함  

## a. Unpack Files

```jsx
[root@study_gain abc]# ll
	total 4
	drwxr-xr-x. 8 root root 4096 Apr  2 13:47 database
[root@study_gain abc]# cd database/
[root@study_gain database]# ll
	total 24
	drwxr-xr-x. 12 root root 4096 Apr  2 13:46 doc
	drwxr-xr-x.  4 root root 4096 Apr  2 13:46 install
	drwxr-xr-x.  2 root root   58 Apr  2 13:46 response
	drwxr-xr-x.  2 root root   33 Apr  2 13:46 rpm
	-rw-r--r--.  1 root root 3226 Aug 15  2009 runInstaller
	drwxr-xr-x.  2 root root   28 Apr  2 13:46 sshsetup
	drwxr-xr-x. 14 root root 4096 Apr  2 13:47 stage
	-rw-r--r--.  1 root root 5402 Aug 17  2009 welcome.html
```

- 압축 해제한 database 파일 확인

## b. Hosts File

```jsx
[root@study_gain ~]# vi /etc/hosts
	127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
	::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
	192.168.182.100 ygiora11g.localdomain ygiora11g
	:wq!
[root@study_gain ~]# ping ygiora11g.localdomain
	PING ygiora11g.localdomain (192.168.182.200) 56(84) bytes of data.
	64 bytes from ygiora11g.localdomain (192.168.182.200): icmp_seq=1 ttl=64 time=0.038 ms
	64 bytes from ygiora11g.localdomain (192.168.182.200): icmp_seq=2 ttl=64 time=0.038 ms
	64 bytes from ygiora11g.localdomain (192.168.182.200): icmp_seq=3 ttl=64 time=0.079 ms
	^C
	--- ygiora11g.localdomain ping statistics ---
	3 packets transmitted, 3 received, 0% packet loss, time 1999ms
	rtt min/avg/max/mdev = 0.038/0.051/0.079/0.021 ms
```

- 127.0.0.1로 hostname 설정해주면 잘못 찾아갈 수 있어서 그건 vm 하나만 띄울 때만!

## c. Oracle Installation Prerequisites

### Automatic Setup

```jsx
[root@study_gain database]# yum install oracle-rdbms-server-11gR2-preinstall
```

> Setup은 automatic 대신 **manual**로 진행할 것!

### Manual Setup

1. `/etc/sysctl.conf` 파일 수정
    
    ```jsx
    [root@study_gain database]# vi /etc/sysctl.conf
    	fs.aio-max-nr = 1048576
    	fs.file-max = 6815744
    	kernel.shmall = 2097152
    	kernel.shmmax = 536870912
    	kernel.shmmni = 4096
    	# semaphores: semmsl, semmns, semopm, semmni
    	kernel.sem = 250 32000 100 128
    	net.ipv4.ip_local_port_range = 9000 65500
    	net.core.rmem_default=262144
    	net.core.rmem_max=4194304
    	net.core.wmem_default=262144
    	net.core.wmem_max=1048586
    	:wq!
    [root@study_gain database]# /sbin/sysctl -p
    	fs.file-max = 6815744
    	kernel.sem = 250 32000 100 128
    	kernel.shmmni = 4096
    	kernel.shmall = 2097152
    	kernel.shmmax = 536870912
    	kernel.panic_on_oops = 1
    	net.core.rmem_default = 262144
    	net.core.rmem_max = 4194304
    	net.core.wmem_default = 262144
    	net.core.wmem_max = 1048576
    	net.ipv4.conf.all.rp_filter = 2
    	net.ipv4.conf.default.rp_filter = 2
    	fs.aio-max-nr = 1048576
    	net.ipv4.ip_local_port_range = 9000 65500
    //현재 커널 파라미터 값 변경
    [root@study_gain database]# vi /etc/security/limits.conf
    	oracle              soft    nproc   2047
    	oracle              hard    nproc   16384
    	oracle              soft    nofile  4096
    	oracle              hard    nofile  65536
    	oracle              soft    stack   10240
    	:wq!
    [root@study_gain database]# vi /etc/pam.d/login
    	session    required     pam_limits.so
    	:wq!
    ```
    
2. 패키지 설치
    
    ```jsx
    yum install binutils -y
    yum install compat-libstdc++-33 -y
    yum install compat-libstdc++-33.i686 -y
    yum install gcc -y
    yum install gcc-c++ -y
    yum install glibc -y
    yum install glibc.i686 -y
    yum install glibc-devel -y
    yum install glibc-devel.i686 -y
    yum install ksh -y
    yum install libgcc -y
    yum install libgcc.i686 -y
    yum install libstdc++ -y
    yum install libstdc++.i686 -y
    yum install libstdc++-devel -y
    yum install libstdc++-devel.i686 -y
    yum install libaio -y
    yum install libaio.i686 -y
    yum install libaio-devel -y
    yum install libaio-devel.i686 -y
    yum install libXext -y
    yum install libXext.i686 -y
    yum install libXtst -y
    yum install libXtst.i686 -y
    yum install libX11 -y
    yum install libX11.i686 -y
    yum install libXau -y
    yum install libXau.i686 -y
    yum install libxcb -y
    yum install libxcb.i686 -y
    yum install libXi -y
    yum install libXi.i686 -y
    yum install make -y
    yum install sysstat -y
    yum install unixODBC -y
    yum install unixODBC-devel -y
    yum install zlib-devel -y
    yum install elfutils-libelf-devel -y
    ```
    
3. groups and users 추가
    
    ```jsx
    [root@study_gain database]# groupadd -g 54321 oinstall
    [root@study_gain database]# groupadd -g 54322 dba
    [root@study_gain database]# groupadd -g 54323 oper
    [root@study_gain database]# groupadd -g 54324 backupdba
    [root@study_gain database]# groupadd -g 54325 dgdba
    [root@study_gain database]# groupadd -g 54326 kmdba
    [root@study_gain database]# groupadd -g 54327 asmdba
    [root@study_gain database]# groupadd -g 54328 asmoper
    [root@study_gain database]# groupadd -g 54329 asmadmin
    [root@study_gain database]# useradd -g oinstall -G dba,oper oracle
    ```
    

### Additional Setup

1. oracle user 비밀번호 변경 및 SELINUX 설정
    
    ```jsx
    [root@study_gain database]# passwd oracle
    	Changing password for user oracle.
    	New password: 
    	BAD PASSWORD: The password is a palindrome
    	Retype new password: 
    	passwd: all authentication tokens updated successfully.
    [root@study_gain database]# vi /etc/selinux/config
    	SELINUX=permissive
    	:wq!
    [root@study_gain database]# setenforce Permissive
    ```
    
2. oracle을 **database** 설치 파일넣고 **DATA**에는 데이터 디스크
    
    ```jsx
    [root@study_gain database]# mkdir /oracle
    [root@study_gain database]# mkdir /DATA
    ```
    
3. oracle 설치할 디렉토리 생성 및 권한 변경
    
    ```jsx
    [root@study_gain database]# mkdir -p /oracle/app/oracle/product/11.2.0.4/db_1
    [root@study_gain ~]# chown -R oracle:oinstall /oracle
    [root@study_gain ~]# chmod -R 775 /oracle
    [root@study_gain ~]# chown -R oracle:oinstall /DATA
    [root@study_gain ~]# chmod -R 775 /DATA
    [root@study_gain ~]# ll /
    	total 32
			//중략.......
    	drwxrwxr-x.   2 oracle oinstall    6 Apr  2 14:17 DATA
    	drwxrwxr-x.   3 oracle oinstall   16 Apr  2 14:50 oracle
    ```
    
4. 콘솔에서 작업하거나 SSH 터널링을 사용하지 않는 경우 **root**로 로그인하고 **xhost** 명령어 실행
    
    ```jsx
    [root@gain_study ~]$ xhost +
    	access control disabled, clients can connect from any host
    ```
    
    - **xhost error** 뜨면 아래와 같이 추가해 주기
        
        ```jsx
        [oracle@study_gain ~]# env | grep DISPLAY
        [oracle@study_gain ~]# export DISPLAY=:0
        [oracle@study_gain ~]# env | grep DISPLAY
        	DISPLAY=:0
        ```
                
5. `/home/oracle/.bash_profile`에 Oracle 설정 값 추가
    
    ```jsx
    [root@study_gain database]# vi /home/oracle/.bash_profile
    	# .bash_profile
    	
    	# Get the aliases and functions
    	if [ -f ~/.bashrc ]; then
    		. ~/.bashrc
    	fi
    	
    	# User specific environment and startup programs
    	
    	PATH=$PATH:$HOME/.local/bin:$HOME/bin
    	
    	export PATH
    	
    	# Oracle Settings
    	TMP=/tmp; export TMP
    	TMPDIR=$TMP; export TMPDIR
    	
    	ORACLE_HOSTNAME=ygiora11g.localdomain; export ORACLE_HOSTNAME
    	ORACLE_UNQNAME=YGIORA; export ORACLE_UNQNAME
    	ORACLE_BASE=/oracle/app/oracle; export ORACLE_BASE
    	ORACLE_HOME=$ORACLE_BASE/product/11.2.0.4/db_1; export ORACLE_HOME
    	ORACLE_SID=DB11G; export ORACLE_SID
    	ORACLE_TERM=xterm; export ORACLE_TERM
    	PATH=/usr/sbin:$PATH; export PATH
    	PATH=$ORACLE_HOME/bin:$PATH; export PATH
    	
    	LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib; export LD_LIBRARY_PATH
    	CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH
    	:wq!
    ```
    
    - UNQNAME = YGIORA
    - hostname = ygiora11g
6. 환경변수 확인
    
    ```jsx
    [giyoo@study_gain ~]$ su - oracle
    	Password: 
    [oracle@study_gain ~]$ env | grep ORACLE
    	ORACLE_UNQNAME=YGIORA
    	ORACLE_SID=DB11G
    	ORACLE_BASE=/oracle/app/oracle
    	ORACLE_HOSTNAME=ygiora11g.localdomain
    	ORACLE_TERM=xterm
    	ORACLE_HOME=/oracle/app/oracle/product/11.2.0.4/db_1
    ```
    
7. 소유 및 실행 권한 변경 후, **runInstaller** 실행
    
    ```jsx
    [root@study_gain /]# chown -R oracle:oinstall /abc
    [root@study_gain /]# ll
    	total 32
			//중략.......
    	drwxr-x--x.   3 oracle oinstall   21 Apr  2 13:46 abc
    ```
    
    ```jsx
    [oracle@study_gain ~]$ cd /abc/
    [oracle@study_gain abc]$ ll
    	total 4
    	drwxr-xr-x. 8 oracle oinstall 4096 Apr  2 13:47 database
    [oracle@study_gain abc]$ cd database/
    [oracle@study_gain database]$ ll
    	total 24
    	drwxr-xr-x. 12 oracle oinstall 4096 Apr  2 13:46 doc
    	drwxr-xr-x.  4 oracle oinstall 4096 Apr  2 13:46 install
    	drwxr-xr-x.  2 oracle oinstall   58 Apr  2 13:46 response
    	drwxr-xr-x.  2 oracle oinstall   33 Apr  2 13:46 rpm
    	-rw-r--r--.  1 oracle oinstall 3226 Aug 15  2009 runInstaller
    	drwxr-xr-x.  2 oracle oinstall   28 Apr  2 13:46 sshsetup
    	drwxr-xr-x. 14 oracle oinstall 4096 Apr  2 13:47 stage
    	-rw-r--r--.  1 oracle oinstall 5402 Aug 17  2009 welcome.html
    [oracle@study_gain database]$ chmod 775 -R *
    [oracle@study_gain database]$ ll
    	total 24
    	drwxrwxr-x. 12 oracle oinstall 4096 Apr  2 13:46 doc
    	drwxrwxr-x.  4 oracle oinstall 4096 Apr  2 13:46 install
    	drwxrwxr-x.  2 oracle oinstall   58 Apr  2 13:46 response
    	drwxrwxr-x.  2 oracle oinstall   33 Apr  2 13:46 rpm
    	-rwxrwxr-x.  1 oracle oinstall 3226 Aug 15  2009 runInstaller
    	drwxrwxr-x.  2 oracle oinstall   28 Apr  2 13:46 sshsetup
    	drwxrwxr-x. 14 oracle oinstall 4096 Apr  2 13:47 stage
    	-rwxrwxr-x.  1 oracle oinstall 5402 Aug 17  2009 welcome.html
    [oracle@study_gain database]$ ./runInstaller
    ```
    

## d. Installation

> 💡 oracle 설치 화면 깜박임이 심하면 해상도를 1024로 변경

1. Configure Security Updates > `체크 해제`
    - Email과 passwd 의미없음
    - DB 방화벽은 **Whitelist** 설정이기 때문
2. Select Install Option > `Create and configure a database`
3. System Class > `Server Class`
4. Grid Options > `Single instance database installation`
    1. DB 1대
    2. Activce ↔ StandBy ⇒ HA
    3. Active ↔ Active (Active 두대 이상)
        - 스토리지 공유
        - RAC (동일 머신 두대 이상, 스토리지 필요)
5. Typical Install Configuration > `Advanced install`
6. Database Edition > `Enterprise Edition`
7. Installation Location > Oracle Base : `/oracle/app/oracle`, Software Location : `/oracle/app/oracle/product/11.2.0.4/db_1`
8. Create Inventory > Inventory Directory : `/oracle/app/oraInventory` > oraInventory Group Name : `oinstall` 확인
9. Configuration Type > `General Purpose / Transaction Processing`
10. Database Identifiers > **/home/oracle/.bash_profile**에서 설정한 *<u>SID</u>* 확인해서 입력 > Global database name : `DB11G` 입력 시 Oracle Service Identifier (SID) 동일하게 입력됨
11. Configuration Options > Memory : `40%` 확인 > `Character Sets : Use Unicode (AL32UTF8)` > Sample Schema : `Create database with sample schemas` 체크
    - Sample Schema : 운영에 설치안함 *(용량 차지하므로 사용안함)*
12. Management Options > `Next`
13. Database Storage > File System > Specify database file location : `/oracle/app/oracle/oradata`
14. Backup and Recovery > Do not enable automated backups
15. Schema Passwords > `Use the same password for all accounts` 체크
16. Operating System Groups > Database Administrator (OSDBA) Group : `dba` > Database Operator (OSOPER) Group : `oper`
17. Prerequisite Checks > `Ignore All` 체크
18. Summary > `Finish`
19. Install Product
20. Finish
    
    ![Untitled](2%E1%84%8E%E1%85%A1%E1%84%89%E1%85%B5%208ac04b2abd1c4310b242fa0c8eb9ad3b/Untitled%201.png)
    
    ```jsx
    [oracle@gain_study oraInventory]$ su -
    	Password: 
    	Last login: Fri Apr  8 19:20:20 EDT 2022 on pts/0
    [root@gain_study ~]# cd /oracle/app/oraInventory
    [root@gain_study oraInventory]# ./orainstRoot.sh
    	Changing permissions of /oracle/app/oraInventory.
    	Adding read,write permissions for group.
    	Removing read,write,execute permissions for world.
    	
    	Changing groupname of /oracle/app/oraInventory to oinstall.
    	The execution of the script is complete.
    [root@gain_study oraInventory]# cd /oracle/app/oracle/product/11.2.0.4/db_1/
    [root@gain_study db_1]# ./root.sh
    	Running Oracle 11g root.sh script...
    	
    	The following environment variables are set as:
    	    ORACLE_OWNER= oracle
    	    ORACLE_HOME=  /oracle/app/oracle/product/11.2.0.4/db_1
    	
    	Enter the full pathname of the local bin directory: [/usr/local/bin]: 
    	   Copying dbhome to /usr/local/bin ...
    	   Copying oraenv to /usr/local/bin ...
    	   Copying coraenv to /usr/local/bin ...
    
    	Creating /etc/oratab file...
    	Entries will be added to the /etc/oratab file as needed by
    	Database Configuration Assistant when a database is created
    	Finished running generic part of root.sh script.
    	Now product-specific root actions will be performed.
    	Finished product-specific root actions.
    ```
    

# 3. SQL 실행

## a. 실행 중인 oracle 서버 확인
- `ps -ef | grep ora_`

    ```jsx
    [oracle@study_gain ~]$ ps -ef | grep ora_
    	oracle    2658     1  0 16:31 ?        00:00:00 ora_dbrm_DB11G
    	oracle    2660     1  0 16:31 ?        00:00:00 ora_psp0_DB11G
    	oracle    2662     1  0 16:31 ?        00:00:00 ora_dia0_DB11G
    	oracle    2664     1  9 16:31 ?        00:00:02 ora_mman_DB11G
    	oracle    2666     1  0 16:31 ?        00:00:00 ora_dbw0_DB11G
    	oracle    2668     1  0 16:31 ?        00:00:00 ora_lgwr_DB11G
    	oracle    2673     1  0 16:31 ?        00:00:00 ora_ckpt_DB11G
    	oracle    2675     1  0 16:31 ?        00:00:00 ora_smon_DB11G
    	oracle    2677     1  0 16:31 ?        00:00:00 ora_reco_DB11G
    	oracle    2680     1  1 16:31 ?        00:00:00 ora_mmon_DB11G
    	oracle    2682     1  0 16:31 ?        00:00:00 ora_mmnl_DB11G
    	oracle    2684     1  0 16:31 ?        00:00:00 ora_d000_DB11G
    	oracle    2686     1  0 16:31 ?        00:00:00 ora_s000_DB11G
    	oracle    2724     1  0 16:31 ?        00:00:00 ora_qmnc_DB11G
    	oracle    2738     1  0 16:31 ?        00:00:00 ora_cjq0_DB11G
    	oracle    2740     1  3 16:31 ?        00:00:00 ora_j000_DB11G
    	oracle    2742     1  0 16:31 ?        00:00:00 ora_j001_DB11G
    	oracle    2744     1  0 16:31 ?        00:00:00 ora_q000_DB11G
    	oracle    2746     1  0 16:31 ?        00:00:00 ora_q001_DB11G
    	oracle    2753  2422  0 16:31 pts/0    00:00:00 grep --color=auto ora_
    ```
    
## b. SQL 실행
- `sqlplus / as sysdba` : sysdba 계정으로 DB 접속
- `shutdown immediate` : DB 내리기
- `startup` : DB 올리기
- `select sysdate from dual;` : 현재 날짜 출력
                
    
    ```jsx
    [oracle@study_gain ~]$ sqlplus / as sysdba
    
    	SQL*Plus: Release 11.2.0.1.0 Production on Sat Apr 2 16:32:05 2022
    	
    	Copyright (c) 1982, 2009, Oracle.  All rights reserved.
    	
    	
    	Connected to:
    	Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
    	With the Partitioning, OLAP, Data Mining and Real Application Testing options
    
    	SQL> shutdown immediate
    		Database closed.
    		Database dismounted.
    		ORACLE instance shut down.
    	SQL> !ps
    	  PID TTY          TIME CMD
    	 2422 pts/0    00:00:00 bash
    	 2825 pts/0    00:00:00 sqlplus
    	 2993 pts/0    00:00:00 ps
    	SQL> startup
    		ORACLE instance started.
    		
    		Total System Global Area 1269366784 bytes
    		Fixed Size		    2212976 bytes
    		Variable Size		  754977680 bytes
    		Database Buffers	  503316480 bytes
    		Redo Buffers		    8859648 bytes
    		Database mounted.
    		Database opened.
    	SQL> select sysdate from dual;
    	
    		SYSDATE
    		---------
    		08-APR-22
    	SQL> shutdown immediate
    		Database closed.
    		Database dismounted.
    		ORACLE instance shut down.
    	SQL> exit
    		Disconnected from Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
    		With the Partitioning, OLAP, Data Mining and Real Application Testing options
    [oracle@study_gain ~]$ ps -ef | grep ora_
    	oracle    3034  2422  0 16:33 pts/0    00:00:00 grep --color=auto ora_
    ```
    
- 만약 DB가 내려가 있으면 `idle instance` 출력
    
    ```jsx
    [oracle@study_gain ~]$ sqlplus / as sysdba
    	
    	SQL*Plus: Release 11.2.0.1.0 Production on Sat Apr 16 14:48:55 2022
    	
    	Copyright (c) 1982, 2009, Oracle.  All rights reserved.
    	
    	Connected to an idle instance.
    ```
    
# 참고 사이트

- Linux LVM : [https://www.sharedit.co.kr/posts/1234](https://www.sharedit.co.kr/posts/1234)
- Oracle Database 11g Release 2 (11.2) Installation On Oracle Linux 7 (OL7) : [https://oracle-base.com/articles/11g/oracle-db-11gr2-installation-on-oracle-linux-7](https://oracle-base.com/articles/11g/oracle-db-11gr2-installation-on-oracle-linux-7)