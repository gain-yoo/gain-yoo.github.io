---
layout: single
title: "[Kubernetes] kubeadm init 에러"
excerpt: "kubelet 프로세스 실패"
categories: "Trouble Shooting"
tag: [Kubernetes, DevOps, NCP, kubeadm, docker, Trouble Shooting, kubelet, systemd]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---


## kubelet 프로세스 실패
저번 포스팅에서 kubeadm으로 직접 클러스터를 생성하면서 시행착오를 겪었습니다.  
  
에러가 발생한 부분은 `kubeadm init` 이었고
원인은 Container Runtime 환경을 Docker로 설정하면서 놓친 부분에 있었습니다.  

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
kubelet이 정상 작동하고 있지 않네요...  
  
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


### 에러 해결

- Container Runtime인 Docker를 제대로 설치하지 않아서 발생한 에러였습니다.
	- Docker Engine에서 kubelet을 systemd 서비스로 관리하기 때문에 Kubernetes에서는 Docker 데몬의 드라이버를 systemd 드라이버로 권장합니다.
	- 따라서 cgroup 드라이버는 systemd로 일치시켜야 하며 그렇지 않을 시 kubelet 프로세스는 실패하게 됩니다.
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
