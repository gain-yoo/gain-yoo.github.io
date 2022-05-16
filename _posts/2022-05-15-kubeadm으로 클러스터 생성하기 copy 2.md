---
layout: single
title: kubeadm으로 클러스터 생성하기 With NCP (1)
excerpt: 사전준비 작업 - 방화벽 포트 오픈 및 서버 생성
categories: Kubernetes
tag: [Kubernetes, DevOps, NCP, kubeadm, docker, calico]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---
이번 포스트에서는 kubeadm으로 클러스터를 직접 생성해보려고 합니다.

Naver Cloud Platform에 무료 크레딧이 남아 있어서 NCP에서 작업을 진행했습니다.
NCP에서 제공되는 Kubernetes Service도 있었지만 직접 서버를 생성해서 구축해 보고 싶었습니다 ^^*

# 1. 사전준비

## 1) 방화벽 포트 열기_<small>Master Node/Worker Node</small>


> **💡 Default ACG** 
> - 모든 들어오는 연결(inbound) 차단  
> - 모든 나가는 연결(outbound) 허용  
> - 원격 접속 기본 포트 (Linux - 22, Windows - 3389)에 대한 TCP 허용 _(Default ACG)_


- **공통**
    - TCP 22번
- **Master Node**
    
    
    | TCP | Inbound | 6443 | Kubernetes API server | All |
    | --- | --- | --- | --- | --- |
    | TCP | Inbound | 2379-2380 | etcd server client API | kube-apiserver, etcd |
    | TCP | Inbound | 10250 | kubelet API | Self, Control Plane |
    | TCP | Inbound | 10259 | kube-scheduler | Self |
    | TCP | Inbound | 10257 | kube-controller-manager | Self |
- **Worker Node**
    
    
    | TCP | Inbound | 10250 | kubelet API | Self, Control Plane |
    | --- | --- | --- | --- | --- |
    | TCP | Inbound | 30000-32767 | NodePort Services | All |

## 2) NCP에서 서버 생성_<small>Master Node/Worker Node</small>

### (1) 서버 생성

1. **서버 생성**
	- Services > Compute > Server > 서버 생성
    - <small>네트워크를 상세하게 신경 쓰지 않고 리전 간의 사설 통신만 하기 위해 VPC 보다 **Classic**으로 선택!</small>  
2. **서버 이미지 선택**
	- 부팅 디스크 크기 : `50GB` > 이미지타입 : `OS` > OS 이미지타입 : `Ubuntu` > 서버 타입 : `Standard` > 서버 이미지 : `ubuntu-18.04` 선택 > 다음
	- <small>Kubernetes 공식 문서에서 <u>서버 생성 조건</u> 확인</small>  
3. **서버 설정**
	- Zone 선택 : `KR-2` > 스토리지 종류 : `SSD` > 서버 세대 : `g1` > 서버 타입 : `Standard` / `vCPU 2개, 메모리 4GB, [SSD]디스크 50GB [g1]` > 요금제 선택 : `월요금제` > 서버 개수 : `1` > 서버 이름 : `master-node` > 반납 보호 : `해제` > 다음    

4. **인증키 설정**
	- `새로운 인증키 생성` > 인증키 이름 : `master-node-key` > `인증키 생성 및 저장` 클릭 > 다음  

5. **네트워크 접근 설정**
	- `신규 ACG 생성` > `+ACG 생성` > ACG 이름 : `master-node-acg` > 설정할 ACG 값 입력 > 생성  
    - 각각의 필요한 ACG 설정 값 + 원격 접속을 위한 22번 포트 허용  

6. **최종 확인**
    
![server](https://user-images.githubusercontent.com/100563973/168481135-d40e4625-42a9-4b65-bfd1-34f59ee81f8f.png)

### (2) 서버 접속 환경 설정

- **포트 포워딩 설정**
    - 포트 포워딩 설정 > 외부 포트의 값은 1024~65534 범위 내로 입력 > `+추가` > 적용
- **관리자 비밀번호 확인하기**
    - 서버 관리 및 설정 변경 > `관리자 비밀번호 확인` > 인증키 파일 첨부 > `비밀번호 확인` 클릭 > 서버 이름, 관리자 이름, 비밀번호 확인 <small>*(해당 비밀번호는 서버 접속 시 필요한 정보이므로 복사하여 별도로 저장 필요)*</small> > 확인

### (3) 서버 접속

1. 서버 접속용 공인 IP, 외부 포트 확인  
2. Putty 설치 및 실행  
3. 서버 접속용 공인 IP, 외부 포트 입력  
4. login as: `root` / password : 관리자 비밀번호 확인 단계에서 확인된 비밀번호 입력  
5. `passwd root`로 비밀번호 변경  

# 2. 설치 과정

## 1) 런타임 Docker 설치_<small>Master Node/Worker Node</small>

>💡 **To install Docker Engine**, you need the 64-bit version of one of these Ubuntu versions:
>- Ubuntu Impish 21.10
>- Ubuntu Hirsute 21.04
>- Ubuntu Focal 20.04 (LTS)
>- Ubuntu Bionic 18.04 (LTS)
>- ~~Ubuntu Linux 16.04 (LTS) 지원 중단~~

### (1) Set up the repository

1. apt package 업데이트 및 설치
    
    ```jsx
    root@master-node1:~# sudo apt-get update
    	Hit:1 http://kr.archive.ubuntu.com/ubuntu bionic InRelease
    	Hit:2 http://kr.archive.ubuntu.com/ubuntu bionic-updates InRelease
    	Hit:3 http://security.ubuntu.com/ubuntu bionic-security InRelease
    	Hit:4 http://kr.archive.ubuntu.com/ubuntu bionic-backports InRelease
    	Reading package lists... Done
    root@master-node1:~# apt-get install \
    >     ca-certificates \
    >     curl \
    >     gnupg \
    >     lsb-release
    ```
    
    
2. Docker의 공식 GPG 키 추가
    
    ```jsx
    root@master-node1:~#  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```
    
    
3. **stable** repository 설정 <small>_(repository 설치 설정 : stable/nightly/test)_</small>
    >- stable : 일반적으로 공급되는 최신 릴리스
	>- nightly : 다음 주요 릴리스에 대해 진행 중인 최신 빌드
	>- test : stable 전에 테스트 준비되어 있는 시험판

    ```jsx
    root@master-node1:~# echo \
    >   "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    >   $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    ```
    

### (2) Install Docker Engine

- apt package 업데이트 후, *최신 버전*의 Docker Engine, containerd, Docker Compose를 설치하거나 다음 단계로 이동하여 특정 버전을 설치
    
    ```jsx
    root@master-node1:~# sudo apt-get update
    root@master-node1:~# sudo apt-get install docker-ce docker-ce-cli containerd.io
    ```
    
    
    - *특정 버전*의 Docker Engine을 설치하려면 repository에 사용 가능한 버전을 확인한 다음, <VERSION_STRING> 대신 버전명 입력하여 설치 진행
        
        ```jsx
        root@master-node1:~# apt-cache madison docker-ce
        	 docker-ce | 5:20.10.14~3-0~ubuntu-bionic | https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages
        	 docker-ce | 5:19.03.15~3-0~ubuntu-bionic | https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages
        	 docker-ce | 5:18.09.9~3-0~ubuntu-bionic | https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages
        	 docker-ce | 18.06.3~ce~3-0~ubuntu | https://download.docker.com/linux/ubuntu bionic/stable amd64 Packages
        root@master-node1:~# sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io docker-compose-plugin
        ```
                

### (3) Configuring the container runtime cgroup driver

- Docker Engine에서 kubelet을 systemd 서비스로 관리하기 때문에 Kubernetes에서는 Docker 데몬의 드라이버를 systemd 드라이버로 권장합니다.
- 따라서 cgroup 드라이버는 systemd로 일치시켜야 하며 그렇지 않을 시 kubelte 프로세스는 실패합니다.  
<small>_daemon.json 파일 생성하지 않으면 `kubeadm init` 실행 시 kubelet 동작 오류 발생_</small>  

```jsx
root@master-node1:~# cat > /etc/docker/daemon.json <<EOF
	> {
	>   "exec-opts": ["native.cgroupdriver=systemd"],
	>   "log-driver": "json-file",
	>   "log-opts": {
	>     "max-size": "100m"
	>   },
	>   "storage-driver": "overlay2"
	> }
	> EOF
root@master-node1:~# mkdir -p /etc/systemd/system/docker.service.d
root@master-node1:~# systemctl daemon-reload
root@master-node1:~# systemctl restart docker
```

### (4) 설치 확인

1. `hello-world` 이미지를 실행하여 Docker 엔진이 올바르게 설치되었는지 확인
    
    ```jsx
    root@master-node1:~# sudo docker run hello-world
    	Unable to find image 'hello-world:latest' locally
    	latest: Pulling from library/hello-world
    	2db29710123e: Pull complete
    	Digest: sha256:10d7d58d5ebd2a652f4d93fdd86da8f265f5318c6a73cc5b6a9798ff6d2b2e67
    	Status: Downloaded newer image for hello-world:latest
    	
    	Hello from Docker!
    	This message shows that your installation appears to be working correctly.
    	
    	To generate this message, Docker took the following steps:
    	 1. The Docker client contacted the Docker daemon.
    	 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    	    (amd64)
    	 3. The Docker daemon created a new container from that image which runs the
    	    executable that produces the output you are currently reading.
    	 4. The Docker daemon streamed that output to the Docker client, which sent it
    	    to your terminal.
    	
    	To try something more ambitious, you can run an Ubuntu container with:
    	 $ docker run -it ubuntu bash
    	
    	Share images, automate workflows, and more with a free Docker ID:
    	 https://hub.docker.com/
    	
    	For more examples and ideas, visit:
    	 https://docs.docker.com/get-started/
    ```
    
    
2. `docker Info` 확인
        

### [참고1] Uninstall Docker Engine

1. Docker Engine, CLI, Containerd 및 Docker Compose 패키지 제거
    
    ```jsx
    sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-compose-plugin
    ```
    
2. 호스트의 이미지, 컨테이너, 볼륨 또는 사용자 지정 구성 파일은 자동으로 제거되지 않습니다.  

	모든 이미지, 컨테이너 및 볼륨을 삭제하려면:
    
    ```jsx
    sudo rm -rf /var/lib/docker
    sudo rm -rf /var/lb/containerd
    ```
    

### [참고2] Configure Docker to start on boot

- Debian/Ubuntu에서 부팅하면 기본적으로 Docker는 자동 시작됩니다.
- 하지만 다른 배포판에서 부팅하게 되면 Docker 및 Containerd는 아래 명령어를 사용해야 자동 시작 가능합니다.

```jsx
root@master-node1:~# sudo systemctl enable docker.service
	Synchronizing state of docker.service with SysV service script with /lib/systemd/systemd-sysv-install.
	Executing: /lib/systemd/systemd-sysv-install enable docker
root@master-node1:~# sudo systemctl enable containerd.service
```

## 2) kubeadm, kubelet 및 kubectl 설치_<small>Master Node/Worker Node</small>

> __💡 3가지 tool의 사용처__
> - kubeadm은 : Cluster 구성 및 업데이트　시 사용
> - kubelet : Cluster의 모든 Node에서 실행되는 Component로, pod나 container를 실행하는 등의 동작 수행
> - kubectl : Kubernetes Cluster에 명령을 내리기 위해 사용하는 CLI

### (1) 버전 호환

1. 지원되는 버전
    
    - Kubernetes 버전 표기 : **x.y.z**
		- **x** major 버전 / **y** minor 버전 / **z** patch 버전
    - Release (1.23, 1.22, 1.21) 에 대한 Release 분기를 유지
    - patch 지원
		- Kubernetes 1.19 이상 - 약 1년 간
		- Kubernetes 1.18 이하 - 약 9개월
2. kube-apiserver 1.23 기준
	- kubeadm은 Kubernetes 구성 요소와 __동일 버전__ 혹은 __한단계 최신 버전__ ⇒ `1.23`
	- kubelet은 kube-apiserver과 __동일 버전__ 혹은 __2단계 낮은 minor 버전__ 까지 허용 _(권장x)_ ⇒ `1.23/1.22`
	- kube-controller-manager, kube-scheduler, cloud-controller-manager는 __동일 버전__ 혹은 최대 __한 단계 낮은 minor 버전__ 까지 허용 ⇒ `1.23/1.22`
    - kubectl은 kube-apiserver의 __한 단계 minor 버전(이전 또는 최신)__ 허용 ⇒ `1.24/1.23/1.22`
	- kube-proxy는 __반드시 kubelet과 동일한 minor 버전__ ⇒ `1.23/1.22`  

	> 버전 호환성을 위해 __1.23__ 으로 통일

### (2) 설치

1. apt package 업데이트 후, kubernetes repository에 필요한 package 설치
	- HTTPS를 통해 repository를 사용할 수 있도록 아래 라이브러리를 설치해 줍니다.
    
    ```jsx
    root@master-node1:~# sudo apt-get update
    root@master-node1:~# sudo apt-get install -y apt-transport-https
    	
    ```
    
    
2. 구글 클라우드의 public signing key 다운로드
    
    ```jsx
    root@master-node1:~# sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    ```
    
    
3. Kubernetes repository 추가
    
    ```jsx
    root@master-node1:~# echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main
    ```
    
    
4. apt package 업데이트 후 kubelet, kubeadm, kubectl 설치 및 해당 버전 고정
    
    ```jsx
    root@master-node1:~# sudo apt-get update
    root@master-node1:~# sudo apt-get install -y kubelet kubeadm kubectl
    root@master-node1:~# sudo apt-mark hold kubelet kubeadm kubectl
    	kubelet set on hold.
    	kubeadm set on hold.
    	kubectl set on hold.
    ```
    

## 3) Swap 메모리 비활성화_<small>Master Node/Worker Node</small>

- kubelet은 Swap 메모리를 사용하도록 설정할 시 속도가 저하되거나 에러가 발생할 수 있습니다. 그래서 Master/Worker Node 모두 Swap 메모리를 사용하지 않도록 설정해야 합니다.
- 사용 중인 Swap 메모리 확인 : `top -d 1` > Swap 필터 적용 > 확인

```jsx
// 현재 스왑메모리 비활성화
root@worker-node1:~# swapoff -a
// 영구적으로 비활성화
root@worker-node1:~# sed -i '2s/^/#/' /etc/fstab
```


# 3. Master Node Settings

## 1) Control Components 설치

	- kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, pause, etcd, coredns
	- flags를 사용하여 repository나 설치 버전 변경이 가능합니다.
		- --image-repository string (Default: k8s.gcr.io)
		- --kubernetes-version string (Default: stable-1)

```jsx
root@master-node1:~# kubeadm config images pull
	[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.23.6
	[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.23.6
	[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.23.6
	[config/images] Pulled k8s.gcr.io/kube-proxy:v1.23.6
	[config/images] Pulled k8s.gcr.io/pause:3.6
	[config/images] Pulled k8s.gcr.io/etcd:3.5.1-0
	[config/images] Pulled k8s.gcr.io/coredns/coredns:v1.8.6
```


## 2) Kubeadm Init

```jsx
root@master-node1:~# kubeadm init
	[init] Using Kubernetes version: v1.23.6
	[preflight] Running pre-flight checks
	[preflight] Pulling images required for setting up a Kubernetes cluster
	[preflight] This might take a minute or two, depending on the speed of your internet connection
	[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
	[certs] Using certificateDir folder "/etc/kubernetes/pki"
	[certs] Generating "ca" certificate and key
	[certs] Generating "apiserver" certificate and key
	[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernet   es.default.svc.cluster.local master-node1] and IPs [10.96.0.1 10.41.2.156]
	[certs] Generating "apiserver-kubelet-client" certificate and key
	[certs] Generating "front-proxy-ca" certificate and key
	[certs] Generating "front-proxy-client" certificate and key
	[certs] Generating "etcd/ca" certificate and key
	[certs] Generating "etcd/server" certificate and key
	[certs] etcd/server serving cert is signed for DNS names [localhost master-node1] and IPs [10.41.2.156 127.0.0.1 ::1]
	[certs] Generating "etcd/peer" certificate and key
	[certs] etcd/peer serving cert is signed for DNS names [localhost master-node1] and IPs [10.41.2.156 127.0.0.1 ::1]
	[certs] Generating "etcd/healthcheck-client" certificate and key
	[certs] Generating "apiserver-etcd-client" certificate and key
	[certs] Generating "sa" key and public key
	[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
	[kubeconfig] Writing "admin.conf" kubeconfig file
	[kubeconfig] Writing "kubelet.conf" kubeconfig file
	[kubeconfig] Writing "controller-manager.conf" kubeconfig file
	[kubeconfig] Writing "scheduler.conf" kubeconfig file
	[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
	[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
	[kubelet-start] Starting the kubelet
	[control-plane] Using manifest folder "/etc/kubernetes/manifests"
	[control-plane] Creating static Pod manifest for "kube-apiserver"
	[control-plane] Creating static Pod manifest for "kube-controller-manager"
	[control-plane] Creating static Pod manifest for "kube-scheduler"
	[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
	[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kuberne   tes/manifests". This can take up to 4m0s
	[apiclient] All control plane components are healthy after 7.503627 seconds
	[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
	[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets    in the cluster
	NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap f   eature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this tr   ansition transparently.
	[upload-certs] Skipping phase. Please see --upload-certs
	[mark-control-plane] Marking the node master-node1 as control-plane by adding the labels: [node-role.kubernetes.io/ma   ster(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
	[mark-control-plane] Marking the node master-node1 as control-plane by adding the taints [node-role.kubernetes.io/mas   ter:NoSchedule]
	[bootstrap-token] Using token: 65z741.dd70q5p4d17ch3cv
	[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
	[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
	[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long te   rm certificate credentials
	[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bo   otstrap Token
	[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
	[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
	[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
	[addons] Applied essential addon: CoreDNS
	[addons] Applied essential addon: kube-proxy
	
	Your Kubernetes control-plane has initialized successfully!
	
	To start using your cluster, you need to run the following as a regular user:
	
	  mkdir -p $HOME/.kube
	  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	  sudo chown $(id -u):$(id -g) $HOME/.kube/config
	
	Alternatively, if you are the root user, you can run:
	
	  export KUBECONFIG=/etc/kubernetes/admin.conf
	
	You should now deploy a pod network to the cluster.
	Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
	  https://kubernetes.io/docs/concepts/cluster-administration/addons/
	
	Then you can join any number of worker nodes by running the following on each as root:
	
	kubeadm join 10.41.2.156:6443 --token 65z741.dd70q5p4d17ch3cv \
	        --discovery-token-ca-cert-hash sha256:a6949071dd2204e2ae577f890f5c58b330b0055ee14773f702647895be919c0a
```



### 시행착오

### 에러 로그

```jsx
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp 127.0.0.1:10248: connect: connection refused.

        Unfortunately, an error has occurred:
                timed out waiting for the condition

        This error is likely caused by:
                - The kubelet is not running
                - The kubelet is unhealthy due to a misconfiguration of the node in some way (required cgroups disabled)

        If you are on a systemd-powered system, you can try to troubleshoot the error with the following commands:
                - 'systemctl status kubelet'
                - 'journalctl -xeu kubelet'

        Additionally, a control plane component may have crashed or exited when started by the container runtime.
        To troubleshoot, list all containers using your preferred container runtimes CLI.

        Here is one example how you may list all Kubernetes containers running in docker:
                - 'docker ps -a | grep kube | grep -v pause'
                Once you have found the failing container, you can inspect its logs with:
                - 'docker logs CONTAINERID'

error execution phase wait-control-plane: couldn't initialize a Kubernetes cluster
To see the stack trace of this error execute with --v=5 or higher
```


### 원인 추적

```jsx
root@master-node1:~# systemctl status kubelet
	● kubelet.service - kubelet: The Kubernetes Node Agent
	   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enable
	  Drop-In: /etc/systemd/system/kubelet.service.d
	           └─10-kubeadm.conf
	   Active: activating (auto-restart) (Result: exit-code) since Tue 2022-05-03 00:31:01
	     Docs: https://kubernetes.io/docs/home/
	  Process: 8735 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_AR
	 Main PID: 8735 (code=exited, status=1/FAILURE)
root@master-node1:~# docker ps -a | grep kube | grep -v pause
root@master-node1:~# docker logs CONTAINERID
	Error: No such container: CONTAINERID
```


### 해결 방법

- Container Runtime인 Docker를 제대로 설치 안해줘서 발생한 에러
    - `daemon.json` 파일 생성 → `kubeadm reset` → `kubeadm init`
    
    ```jsx
    root@master-node1:~# cat > /etc/docker/daemon.json <<EOF
    	> {
    	>   "exec-opts": ["native.cgroupdriver=systemd"],
    	>   "log-driver": "json-file",
    	>   "log-opts": {
    	>     "max-size": "100m"
    	>   },
    	>   "storage-driver": "overlay2"
    	> }
    	> EOF
    root@master-node1:~# mkdir -p /etc/systemd/system/docker.service.d
    root@master-node1:~# systemctl daemon-reload
    root@master-node1:~# systemctl restart docker
    root@master-node1:~# kubeadm reset
    	[reset] Reading configuration from the cluster...
    	[reset] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-confi                      g -o yaml'
    	W0503 00:56:43.361756   26579 reset.go:101] [reset] Unable to fetch the kubeadm-config ConfigMa                      p from cluster: failed to get config map: configmaps "kubeadm-config" not found
    	[reset] WARNING: Changes made to this host by 'kubeadm init' or 'kubeadm join' will be reverted                      .
    	[reset] Are you sure you want to proceed? [y/N]: y
    	[preflight] Running pre-flight checks
    	W0503 00:57:41.933212   26579 removeetcdmember.go:80] [reset] No kubeadm config, using etcd pod spec to get data directory
    	[reset] Stopping the kubelet service
    	[reset] Unmounting mounted directories in "/var/lib/kubelet"
    	[reset] Deleting contents of config directories: [/etc/kubernetes/manifests /etc/kubernetes/pki]
    	[reset] Deleting files: [/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf]
    	[reset] Deleting contents of stateful directories: [/var/lib/etcd /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni]
    
    	The reset process does not clean CNI configuration. To do so, you must remove /etc/cni/net.d
    	
    	The reset process does not reset or clean up iptables rules or IPVS tables.
    	If you wish to reset iptables, you must do so manually by using the "iptables" command.
    	
    	If your cluster was setup to utilize IPVS, run ipvsadm --clear (or similar)
    	to reset your system's IPVS tables.
    	
    	The reset process does not clean your kubeconfig files and you must remove them manually.
    	Please, check the contents of the $HOME/.kube/config file.
    
    root@master-node1:~# kubeadm init
    ```
    

## 3) Kubectl 설정
- kubect으을 사용하기 위해 디렉토리 생성 및 소유권을 변경해 줍니다.

```jsx
root@master-node1:~# mkdir -p $HOME/.kube
root@master-node1:~# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
root@master-node1:~# sudo chown $(id -u):$(id -g) $HOME/.kube/config
```


# 4. 네트워크 플러그인 설치_<small>Master Node</small>

- pod 간의 통신을 위해 네트워크 플러그인 설치가 필요합니다.
- 네트워크 플러그인 종류
    - Weave  
        `kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"`
        
    - Flannel  
        `kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml`
        
    - Calico  
        `kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml`
        
- **Calico 설치**
    
    ```jsx
    root@master-node1:~# kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml**
    	configmap/calico-config created
    	....중략....
    	daemonset.apps/calico-node created
    	serviceaccount/calico-node created
    	deployment.apps/calico-kube-controllers created
    	serviceaccount/calico-kube-controllers created
    	Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
    	poddisruptionbudget.policy/calico-kube-controllers created
    ```
    

# 5. Worker Node Settings

## 1) Kubeadm Join
- kubeadm join 명령어를 통해 Worker Node를 Master Node에 연결해 줍니다.

```jsx
root@worker-node1:~# kubeadm join 10.41.2.156:6443 --token 0toaxd.7se6f4jxz6zlu1qr --discovery-token-ca-cert-hash sha256:a6949071dd2204e2ae577f890f5c58b330b0055ee14773f702647895be919c0a
	[preflight] Running pre-flight checks
	[preflight] Reading configuration from the cluster...
	[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
	W0503 01:50:32.857613   16706 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
	[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
	[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
	[kubelet-start] Starting the kubelet
	[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
	
	This node has joined the cluster:
	* Certificate signing request was sent to apiserver and a response was received.
	* The Kubelet was informed of the new secure connection details.
	
	Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```


### 시행착오

### 에러 로그

```jsx
root@worker-node1:~# kubeadm join 10.41.2.156:6443 --token 65z741.dd70q5p4d17ch3cv \
>         --discovery-token-ca-cert-hash sha256:a6949071dd2204e2ae577f890f5c58b330b0055ee14773f70264789
	[preflight] Running pre-flight checks
	error execution phase preflight: couldn't validate the identity of the API Server: invalid discovery token CA certificate hash: invalid hash "sha256:a6949071dd2204e2ae577f890f5c58b330b0055ee14773f70264789", expected a 32 byte SHA-256 hash, found 27 bytes
	To see the stack trace of this error execute with --v=5 or higher
```


### 원인 및 해결방법

- 토큰이 맞지 않아서 생긴 에러
- 토큰 재생성

```jsx
root@master-node1:~# kubeadm token list
	TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION                                                EXTRA GROUPS
	65z741.dd70q5p4d17ch3cv   23h         2022-05-03T15:59:09Z   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
root@master-node1:~# kubeadm token delete 65z741.dd70q5p4d17ch3cv
	bootstrap token "65z741" deleted
root@master-node1:~# kubeadm token list
root@master-node1:~# kubeadm token create --print-join-command
	kubeadm join 10.41.2.156:6443 --token 0toaxd.7se6f4jxz6zlu1qr --discovery-token-ca-cert-hash sha256:a6949071dd2204e2ae577f890f5c58b330b0055ee14773f702647895be919c0a
```


## 2) 각 노드 상태 확인

- Node끼리 잘 연결이 되었는지 상태를 확인할 수 있는 명령어는 여럿 있는데 그 중 저는 아래 세 가지로 확인해 보았습니다.

### (1) Kubectl get nodes

- Master Node에서 아래 명령어를 입력하면 Node끼리 잘 연결이 되었는지 확인이 가능합니다.

```jsx
root@master-node1:~# kubectl get node
	NAME           STATUS   ROLES                  AGE     VERSION
	master-node1   Ready    control-plane,master   56m     v1.23.6
	worker-node1   Ready    <none>                 5m30s   v1.23.6
```


### (2) Kubectl get cs

- component의 상태를 확인할 수 있는 명령어인데 1.19 버전부터는 사용하지 않는다고 합니다.

```jsx
root@master-node1:~# kubectl get cs
	Warning: v1 ComponentStatus is deprecated in v1.19+
	NAME                 STATUS    MESSAGE                         ERROR
	scheduler            Healthy   ok
	controller-manager   Healthy   ok
	etcd-0               Healthy   {"health":"true","reason":""}
```


### (3) API 구성 요소 확인

- kubernetes component는 kube-system라는 namespace에 설치되어 kube-system에서 확인 가능합니다.

```jsx
root@master-node1:~# kubectl get po -o custom-columns=POD:metadata.name,NODE:spec.nodeNam  e --sort-by spec.nodeName -n kube-system
POD                                       NODE
calico-kube-controllers-7c845d499-crnbg   master-node1
kube-controller-manager-master-node1      master-node1
calico-node-svp7t                         master-node1
coredns-64897985d-b6g7z                   master-node1
coredns-64897985d-w2rcv                   master-node1
etcd-master-node1                         master-node1
kube-apiserver-master-node1               master-node1
kube-proxy-86k76                          master-node1
kube-scheduler-master-node1               master-node1
calico-node-j8d59                         worker-node1
kube-proxy-j257x                          worker-node1
```

  
# 참고 사이트
---
- Docker 공식 문서
	- Docker storage driver : [https://docs.docker.com/storage/storagedriver/select-storage-driver/](https://docs.docker.com/storage/storagedriver/select-storage-driver/)
	- Install Docker Engine on Ubuntu : [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)
- Kubernetes 공식 문서
	- 버전 차이(skew) 정책 : [https://kubernetes.io/ko/releases/version-skew-policy/](https://kubernetes.io/ko/releases/version-skew-policy/)
	- Creating a cluster with kubeadm : [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#version-skew-policy)
- NCP 가이드
	- 리눅스 서버 생성 : [https://www.ncloud.com/guideCenter/guide/1](https://www.ncloud.com/guideCenter/guide/1)
