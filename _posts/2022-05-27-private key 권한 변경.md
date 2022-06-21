---
layout: single
title: "[AWS] EC2 SSH 접속 에러"
excerpt: "EC2에서 생성한 Keypair로 SSH 접속"
categories:
- Trouble Shooting
tag: [AWS, WSL, Windows Subsystem for Linux, 리눅스용 윈도우 하위 시스템, chmod, 파일 권한, ubuntu 20.04.4 LTS, chmod 400, SSH, private key]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

인스턴스에 접속하기 위해 ssh로 접근하는데 아래와 같은 에러 메세지가 떴다......whyrano whyrano......!!!!!

```java
gain@LAPTOP-NGE5O25S:/mnt/c/Users/User/Downloads$ ssh -i ./myk8s.pem ubuntu@13.125.127.182
	The authenticity of host '13.125.127.182 (13.125.127.182)' can't be established.
	ECDSA key fingerprint is SHA256:YQTjS5h0l/6FhEK7ApvX6vq9wbe88VG8JDQ9y95R3YM.
	Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
	Warning: Permanently added '13.125.127.182' (ECDSA) to the list of known hosts.
	@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
	@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
	@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
	Permissions 0555 for './myk8s.pem' are too open.
	It is required that your private key files are NOT accessible by others.
	This private key will be ignored.
	Load key "./myk8s.pem": bad permissions
	ubuntu@13.125.127.182's password:
```
<br>
❓❗ 권한 문제였다 ㄴ(ㅇ0ㅇ)ㄱ 

Private Key를 User 외 다른 모든 사람이 읽거나 쓸 수 있는 경우 SSH는 Key를 무시하고 에러 메세지를 뱉어낸다고 한다.  
그래서 최소한으로 User만 읽기 권한을 가져야 한다.

참고 ) [인스턴스 연결 문제 해결](https://docs.aws.amazon.com/ko_kr/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html#troubleshoot-unprotected-key)  

```java
gain@LAPTOP-NGE5O25S:/mnt/c/Users/User/Downloads$ ll myk8s.pem
	-rwxrwxrwx 1 gain gain 1678 May 27 14:59 myk8s.pem*
gain@LAPTOP-NGE5O25S:/mnt/c/Users/User/Downloads$ chmod 400 myk8s.pem 
gain@LAPTOP-NGE5O25S:/mnt/c/Users/User/Downloads$ ll myk8s.pem                               
	-r-------- 1 gain gain 1678 May 27 14:59 myk8s.pem
```

권한을 변경해 주니 아래와 같이 접속 성공!✌😁  
  
*권한 변경하려니 **에러 속의 에러** 발생! 이번엔 wsl 관련 문제였다. [[WSL] WSL2에서 chmod 미작동 에러 (ubuntu 20.04.4 LTS)](https://gain-yoo.github.io/trouble%20shooting/wsl_chmod/) 포스팅 참고*

```java
gain@LAPTOP-NGE5O25S:/mnt/c/Users/User/Downloads$ ssh -i ./myk8s.pem ubuntu@13.125.127.182
	Welcome to Ubuntu 22.04 LTS (GNU/Linux 5.15.0-1005-aws x86_64)
	
		* Documentation:  https://help.ubuntu.com
		* Management:     https://landscape.canonical.com
		* Support:        https://ubuntu.com/advantage
	
		System information as of Fri May 27 18:20:23 KST 2022        
	
		System load:  0.3359375         Processes:                143  Usage of /:   9.0% of 38.60GB   Users logged in:          0  
		Memory usage: 21%               IPv4 address for docker0: 172.17.0.1
		Swap usage:   0%                IPv4 address for ens5:    192.168.10.10
	
	
	27 updates can be applied immediately.
	21 of these updates are standard security updates.
	To see these additional updates run: apt list --upgradable     
	
	
	Last login: Fri May 27 16:07:30 2022 from 125.178.215.17       
(🐤 |kubernetes-admin@kubernetes:default) root@k8s-m:~#
```