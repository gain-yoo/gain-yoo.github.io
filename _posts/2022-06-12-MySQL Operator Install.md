---
layout: single
title: "[Database/Kubernetes/DOIK] Custom Resource Definition Install Error"
excerpt: "Kubernetes에 MySQL Operator"
categories:
- Database
tag: [Kubernetes, 쿠버네티스, DevOps, AWS, CRD, CR, Custom Resource, 커스텀 리소스, MySQL Operator, MySQL InnoDB Cluster, Helm, 프로메테우스, Prometheus Operator, kube-prometheus]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

최근에 HPA 설정에 관해 작업하고 있었고, cpu 및 메모리 리소스 대신 다른 자원을 metric으로 수집하지 못할까 궁금했다. 그러던 중에 **Promethues**를 **Custom Resource**로 생성할 수 있다는 사실을 알고 흥미를 갖는 중이었다.  
  
마침 타이밍이 딱 좋게 스터디에서도 Custom Resource가 언급되었고 바로 <u>Database와 Prometheus를 결합</u>한 실습을 해야겠다 생각했다.  
  
그래서 이번 포스팅에서 **<u>MySQL Operator, InnoDB Cluster, Prometheus Operator</u>**를 설치부터 해 볼 것이다.  
*(사실 Prometheus Operator를 설치할 때 살짝 헤매기도 했지만 욕심내니까 진도가 너무 더뎠다 😢)*  

Prometheus Operator 설치에는 세 가지 방법이 있는데
# Prerequisites

먼저 AWS에서 가시다님이 제공해 주신 CloudFormation 템플릿으로 바닐라 쿠버네티스 클러스터를 생성해서 진행했다.

이전 포스팅 참고 : https://gain-yoo.github.io/database/DOIK-1%EC%B0%A8%EC%8B%9C-(1)/  

*이전 포스팅과 스펙은 약간 다르다*
  
> 본 실습에서 사용한 spec :
>
> OS : Ubuntu 22.04 LTS
> Kubernetes : v1.23.7
> Master 1개: AWS t3.large (2cpu, ram4G)
> Node 3개: AWS t3.medium (2cpu, ram8G)
> 



## MySQL Operator 설치 with Helm

1. repo 추가 및 설치
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm repo add mysql-operator https://mysql.github.io/mysql-operator/
    	"mysql-operator" has been added to your repositories
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm repo update
    	Hang tight while we grab the latest from your chart repositories...
    	...Successfully got an update from the "mysql-operator" chart repository
    	Update Complete. ⎈Happy Helming!⎈
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm show values mysql-operator/mysql-operator
    	image:
    	  registry:
    	  repository: mysql       
    	  name: mysql-operator    
    	  pullPolicy: IfNotPresent
    	  # Overrides the image tag whose default is the chart appVersion.
    	  tag: ""
    	  pullSecrets:
    	    enabled: false
    	    secretName:
    	
    	envs:
    	    imagesPullPolicy: IfNotPresent
    	    imagesDefaultRegistry:
    	    imagesDefaultRepository:
    
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm install mysql-operator mysql-operator/mysql-operator --namespace mysql-operator --create-namespace --version 2.0.4
    	NAME: mysql-operator
    	LAST DEPLOYED: Mon Jun  6 21:12:43 2022
    	NAMESPACE: mysql-operator
    	STATUS: deployed
    	REVISION: 1
    	TEST SUITE: None
    	NOTES:
    	Create an InnoDB Cluster by executing:
    	1. When using a source distribution / git clone: `helm install [cluster-name] -n [ns-name] ~/helm/mysql-innodbcluster`
    	2. When using Helm repos :  `helm install [cluster-name] -n [ns-name] mysql-innodbcluster`
    ```
    
2. 설치 확인
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl get deploy,pod -n mysql-operator
    	NAME                             READY   UP-TO-DATE   AVAILABLE   AGE 
    	deployment.apps/mysql-operator   1/1     1            1           15s 
    	
    	NAME                                  READY   STATUS    RESTARTS   AGE
    	pod/mysql-operator-7f7f5f795c-lgsw9   1/1     Running   0          15s
    ```
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~#  kubectl get crd | grep -v calico
    	NAME                              CREATED AT
    	clusterkopfpeerings.zalando.org   2022-06-06T12:12:40Z
    	innodbclusters.mysql.oracle.com   2022-06-06T12:12:40Z
    	kopfpeerings.zalando.org          2022-06-06T12:12:40Z
    	mysqlbackups.mysql.oracle.com     2022-06-06T12:12:40Z
    ```
    
3. 삭제
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm uninstall mysql-operator -n mysql-operator && kubectl delete ns mysql-operator
    	release "mysql-operator" uninstalled
    	namespace "mysql-operator" deleted
    ```
    

### **MySQL InnoDB Cluster** 설치 with Helm

1. 설치
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm install mycluster mysql-operator/mysql-innodbcluster --set credentials.root.password='sakila' --set tls.useSelfSigned=true --namespace mysql-cluster --create-namespace --version 2.0.4
    	NAME: mycluster
    	LAST DEPLOYED: Mon Jun  6 21:16:49 2022
    	NAMESPACE: mysql-cluster
    	STATUS: deployed
    	REVISION: 1
    	TEST SUITE: None
    ```
    
2. 설치 확인
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm get values mycluster -n mysql-cluster
    
    	USER-SUPPLIED VALUES:
    	credentials:
    	  root:
    	    password: sakila
    	tls:
    	  useSelfSigned: true
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# kubectl get innodbcluster,sts,pod,pv,pvc,svc,pdb,all -n mysql-cluster
    	NAME                                       STATUS   ONLINE   INSTANCES   ROUTERS   AGE
    	innodbcluster.mysql.oracle.com/mycluster   ONLINE   3        3           1         23m
    	
    	NAME                         READY   AGE
    	statefulset.apps/mycluster   3/3     23m
    	
    	NAME                                    READY   STATUS    RESTARTS   AGE
    	pod/mycluster-0                         2/2     Running   0          23m
    	pod/mycluster-1                         2/2     Running   0          23m
    	pod/mycluster-2                         2/2     Running   0          23m
    	pod/mycluster-router-5f9758dc8f-q7hgp   1/1     Running   0          21m
    
    	NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS   REASON   AGE
    	persistentvolume/pvc-568da398-6de0-49c8-bc83-c50b177fe18e   2Gi        RWO            Delete           Bound    mysql-cluster/datadir-mycluster-0   local-path              32m
    	persistentvolume/pvc-70e8e355-06d3-4e7c-8b3b-268765dd3d53   2Gi        RWO            Delete           Bound    mysql-cluster/datadir-mycluster-2   local-path              32m
    	persistentvolume/pvc-bc16b438-6fc5-4b0a-a5b0-784153f12ad8   2Gi        RWO            Delete           Bound    mysql-cluster/datadir-mycluster-1   local-path              32m
    
    	NAME                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    	persistentvolumeclaim/datadir-mycluster-0   Bound    pvc-568da398-6de0-49c8-bc83-c50b177fe18e   2Gi        RWO            local-path     23m
    	persistentvolumeclaim/datadir-mycluster-1   Bound    pvc-bc16b438-6fc5-4b0a-a5b0-784153f12ad8   2Gi        RWO            local-path     23m
    	persistentvolumeclaim/datadir-mycluster-2   Bound    pvc-70e8e355-06d3-4e7c-8b3b-268765dd3d53   2Gi        RWO            local-path     23m
    	
    	NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                                                  AGE
    	service/mycluster             ClusterIP   10.200.1.168   <none>        3306/TCP,33060/TCP,6446/TCP,6448/TCP,6447/TCP,6449/TCP   23m
    	service/mycluster-instances   ClusterIP   None           <none>        3306/TCP,33060/TCP,33061/TCP                             23m
    	
    	NAME                                       MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
    	poddisruptionbudget.policy/mycluster-pdb   N/A             1                 1                     23m
    	
    	NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
    	deployment.apps/mycluster-router   1/1     1            1           23m
    	
    	NAME                                          DESIRED   CURRENT   READY   AGE
    	replicaset.apps/mycluster-router-5f9758dc8f   1         1         1       23m
    ```
    
3. 이벤트 확인
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# kubectl describe innodbcluster -n mysql-cluster | grep Events: -A30
    	Events:
    	  Type    Reason            Age   From      Message
    	  ----    ------            ----  ----      -------
    	  Normal  StatusChange      25m   operator  Cluster status changed to INITIALIZING. 0 member(s) ONLINE
    	  Normal  Join              24m   operator  Joining mycluster-2 to cluster
    	  Normal  ResourcesCreated  25m   operator  Dependency resources created, switching status to PENDING
    	  Normal  StatusChange      24m   operator  Cluster status changed to ONLINE. 1 member(s) ONLINE
    	  Normal  Join              24m   operator  Joining mycluster-1 to cluster
    	  Normal  Logging           25m   kopf      Initializing InnoDB Cluster name=mycluster namespace=mysql-cluster
    	  Normal  Logging           25m   kopf      Router instances:          1
    	  Normal  Logging           25m   kopf      Backup profiles:           0
    	  Normal  Logging           25m   kopf      Base ServerId:             1000
    	  Normal  Logging           25m   kopf      ImageRepository:           mysql
    	  Normal  Logging           25m   kopf      Backup schedules:          0
    	  Normal  Logging           25m   kopf      Router Image:              mysql/mysql-router:8.0.29 / IfNotPresent
    	  Normal  Logging           25m   kopf      Server.TLS.useSelfSigned:  True
    	  Normal  Logging           25m   kopf      InnoDB Cluster mysql-cluster/mycluster Edition(Edition.community) Edition
    	  Normal  Logging           25m   kopf      Server Image:     mysql/mysql-server:8.0.29 / IfNotPresent
    	  Normal  Logging           25m   kopf      Sidecar Image:    mysql/mysql-operator:8.0.29-2.0.4 / IfNotPresent
    	  Normal  Logging           25m   kopf      ImagePullPolicy:  ImagePullPolicy.IfNotPresent     
    	  Normal  Logging           25m   kopf      Handler 'on_innodbcluster_field_instances/spec.instances' succeeded.
    	  Normal  Logging           25m   kopf      Handler 'on_innodbcluster_field_version/spec.version' succeeded.
    	  Normal  Logging           25m   kopf      Creation is processed: 6 succeeded; 0 failed.      
    	  Normal  Logging           25m   kopf      on_innodbcluster_field_tls_use_self_signed
    	  Normal  Logging           25m   kopf      Handler 'on_innodbcluster_create' succeeded.       
    	  Normal  Logging           25m   kopf      Handler 'on_innodbcluster_field_image_pull_policy/spec.imagePullPolicy' succeeded.
    	  Normal  Logging           25m   kopf      Handler 'on_innodbcluster_field_router_instances/spec.router.instances' succeeded.
    	  Normal  Logging           25m   kopf      Handler 'on_innodbcluster_field_tls_use_self_signed/spec.tlsUseSelfSigned' succeeded.
    ```
    
4. 초기 설정 확인
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# kubectl describe cm -n mysql-cluster mycluster-initconf
    //....생략....
    	01-group_replication.cnf:
    	----
    	# GR and replication related options
    	# Do not edit.
    	[mysqld]
    	log_bin=mycluster
    	enforce_gtid_consistency=ON
    	gtid_mode=ON   # 그룹 복제 모드 사용을 위해서 GTID 활성화
    	relay_log_info_repository=TABLE  # 복제 메타데이터는 데이터 일관성을 위해 **릴레이로그**를 파일이 아닌 **테이블**에 저장
    	skip_slave_start=1
    //....생략....
    ```
    
5. probe 확인
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# kubectl describe pod -n mysql-cluster mycluster-0 | egrep 'Liveness|Readiness:|Startup'
        Liveness:       exec [/livenessprobe.sh] delay=15s timeout=1s period=15s #success=1 #failure=10
        Readiness:      exec [/readinessprobe.sh] delay=10s timeout=1s period=5s #success=1 #failure=10000
        Startup:        exec [/livenessprobe.sh 8] delay=5s timeout=1s period=3s #success=1 #failure=10000
    ```
    
6. 삭제
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm uninstall mycluster -n mysql-cluster && kubectl delete ns mysql-cluster
    	release "mycluster" uninstalled
    	namespace "mysql-cluster" deleted
    ```
    

## Prometheus Operator 생성

운영자는 다음 [사용자 지정 리소스 정의(CRD)](https://github.com/coreos/prometheus-operator/blob/master/README.md#customresourcedefinitions) 에서 작동합니다 .

- **원하는 Prometheus**
    
    배포를 정의하는 Prometheus. 운영자는 리소스 정의와 일치하는 배포가 실행되고 있는지 항상 확인합니다.
    
- **ServiceMonitor**
    
    , 서비스 그룹을 모니터링하는 방법을 선언적으로 지정합니다. 운영자는 정의를 기반으로 Prometheus 스크랩 구성을 자동으로 생성합니다.
    
- **Prometheus**
    
    경고 및 기록 규칙을 포함하는 Prometheus 인스턴스에서 로드할 수 있는 원하는 Prometheus 규칙 파일을 정의하는 PrometheusRule .
    
- **원하는 Alertmanager**
    
    배포를 정의하는 Alertmanager. 운영자는 리
    

### Install Prometheus Operator

```java
(🍉 |DOIK-Lab:default) root@k8s-m:~# git clone https://github.com/prometheus-operator/kube-prometheus.git
	Cloning into 'kube-prometheus'...
	remote: Enumerating objects: 16271, done.
	remote: Counting objects: 100% (192/192), done.
	remote: Compressing objects: 100% (101/101), done.
	remote: Total 16271 (delta 126), reused 128 (delta 84), pack-reused 16079
	Receiving objects: 100% (16271/16271), 8.17 MiB | 11.57 MiB/s, done.
	Resolving deltas: 100% (10456/10456), done.
(🍉 |DOIK-Lab:default) root@k8s-m:~# git clone https://github.com/mateobur/prometheus-monitoring-guide.git
	Cloning into 'prometheus-monitoring-guide'...
	remote: Enumerating objects: 58, done.
	remote: Total 58 (delta 0), reused 0 (delta 0), pack-reused 58
		Receiving objects: 100% (58/58), 11.29cd po KiB | 1.88 MiB/s, done.
	Resolving deltas: 100% (18/18), done.
(🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl create namespace monitoring
	namespace/monitoring created
```

```java
(🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
	customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
	customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
	customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
	customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
	customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com created
	customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com created
	customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
	clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator created
	clusterrole.rbac.authorization.k8s.io/prometheus-operator created
	deployment.apps/prometheus-operator created
	serviceaccount/prometheus-operator created
	service/prometheus-operator created
	The CustomResourceDefinition "prometheuses.monitoring.coreos.com" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
//(🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl apply -f kube-prometheus/manifests/
(🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl replace -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml --force
	customresourcedefinition.apiextensions.k8s.io "alertmanagerconfigs.monitoring.coreos.com" deleted
	customresourcedefinition.apiextensions.k8s.io "alertmanagers.monitoring.coreos.com" deleted
	customresourcedefinition.apiextensions.k8s.io "podmonitors.monitoring.coreos.com" deleted
	customresourcedefinition.apiextensions.k8s.io "probes.monitoring.coreos.com" deleted
	customresourcedefinition.apiextensions.k8s.io "prometheusrules.monitoring.coreos.com" deleted
	customresourcedefinition.apiextensions.k8s.io "servicemonitors.monitoring.coreos.com" deleted
	customresourcedefinition.apiextensions.k8s.io "thanosrulers.monitoring.coreos.com" deleted
	clusterrolebinding.rbac.authorization.k8s.io "prometheus-operator" deleted
	clusterrole.rbac.authorization.k8s.io "prometheus-operator" deleted
	deployment.apps "prometheus-operator" deleted
	serviceaccount "prometheus-operator" deleted
	service "prometheus-operator" deleted
	customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com replaced
	customresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com replaced
	clusterrolebinding.rbac.authorization.k8s.io/prometheus-operator replaced
	clusterrole.rbac.authorization.k8s.io/prometheus-operator replaced
	deployment.apps/prometheus-operator replaced
	serviceaccount/prometheus-operator replaced
	service/prometheus-operator replaced
```

[[Kubernetes] Custom Resource Definition Install Error](https://gain-yoo.github.io/trouble%20shooting/custom-resource-error/)

리소스 확인

```java
(🍉 |DOIK-Lab:default) root@k8s-m:~# k get all -n monitoring 
	NAME                                      READY   STATUS    RESTARTS        AGE
	pod/blackbox-exporter-746c64fd88-t7mwl    3/3     Running   0               54m
	pod/grafana-55f8bc6d97-5s5b8              1/1     Running   0               54m
	pod/kube-state-metrics-6c8846558c-dr5jd   3/3     Running   0               54m
	pod/node-exporter-77xm4                   2/2     Running   0               54m
	pod/node-exporter-lgdqz                   2/2     Running   0               54m
	pod/node-exporter-mrwz8                   2/2     Running   0               54m
	pod/node-exporter-tnmxw                   2/2     Running   0               54m
	pod/prometheus-adapter-6455646bdc-9nwzj   1/1     Running   0               54m
	pod/prometheus-adapter-6455646bdc-mz9z9   1/1     Running   0               54m
	pod/prometheus-operator-f59c8b954-2fzhv   2/2     Running   5 (4m39s ago)   54m
	
	NAME                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
	service/alertmanager-main     ClusterIP   10.200.1.20    <none>        9093/TCP,8080/TCP    55m
	service/blackbox-exporter     ClusterIP   10.200.1.83    <none>        9115/TCP,19115/TCP   54m
	service/grafana               ClusterIP   10.200.1.128   <none>        3000/TCP             54m
	service/kube-state-metrics    ClusterIP   None           <none>        8443/TCP,9443/TCP    54m
	service/node-exporter         ClusterIP   None           <none>        9100/TCP             54m
	service/prometheus-adapter    ClusterIP   10.200.1.197   <none>        443/TCP              54m
	service/prometheus-k8s        ClusterIP   10.200.1.145   <none>        9090/TCP,8080/TCP    54m
	service/prometheus-operator   ClusterIP   None           <none>        8443/TCP             54m
	
	NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
	daemonset.apps/node-exporter   4         4         4       4            4           kubernetes.io/os=linux   54m
	
	NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
	deployment.apps/blackbox-exporter     1/1     1            1           54m
	deployment.apps/grafana               1/1     1            1           54m
	deployment.apps/kube-state-metrics    1/1     1            1           54m
	deployment.apps/prometheus-adapter    2/2     2            2           54m
	deployment.apps/prometheus-operator   1/1     1            1           54m
	
	NAME                                            DESIRED   CURRENT   READY   AGE
	replicaset.apps/blackbox-exporter-746c64fd88    1         1         1       54m
	replicaset.apps/grafana-55f8bc6d97              1         1         1       54m
	replicaset.apps/kube-state-metrics-6c8846558c   1         1         1       54m
	replicaset.apps/prometheus-adapter-6455646bdc   2         2         2       54m
	replicaset.apps/prometheus-operator-f59c8b954   1         1         1       54m
```

### Install ****kube-prometheus****

1. Clone kube-prometheus
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~# git clone https://github.com/coreos/kube-prometheus.git
    	Cloning into 'kube-prometheus'...
    	remote: Enumerating objects: 16271, done.
    	remote: Counting objects: 100% (192/192), done.
    	remote: Compressing objects: 100% (104/104), done.
    	remote: Total 16271 (delta 123), reused 128 (delta 81), pack-reused 16079
    	Receiving objects: 100% (16271/16271), 8.17 MiB | 11.65 MiB/s, done.
    	Resolving deltas: 100% (10463/10463), done.
    ```
    
2. crd 생성
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~# cd kube-prometheus/
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k create -f manifests/setup/
    	customresourcedefinition.apiextensions.k8s.io/alertmanagerconfigs.monitoring.coreos.com created
    	customresourcedefinition.apiextensions.k8s.io/alertmanagers.monitoring.coreos.com created
    	customresourcedefinition.apiextensions.k8s.io/podmonitors.monitoring.coreos.com created
    	customresourcedefinition.apiextensions.k8s.io/probes.monitoring.coreos.com created
    	customresourcedefinition.apiextensions.k8s.io/prometheuses.monitoring.coreos.com created
    	customresourcedefinition.apiextensions.k8s.io/prometheusrules.monitoring.coreos.com createdcustomresourcedefinition.apiextensions.k8s.io/servicemonitors.monitoring.coreos.com createdcustomresourcedefinition.apiextensions.k8s.io/thanosrulers.monitoring.coreos.com created
    	namespace/monitoring created
    ```
    
3. 리소스 생성
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
    	No resources found
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# kubectl create -f manifests/
    	//...생략...
    ```
    
4. 리소스 확인
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get all -n monitoring 
    	NAME                                      READY   STATUS    RESTARTS   AGE
    	pod/alertmanager-main-0                   2/2     Running   0          4m49s
    	pod/alertmanager-main-1                   2/2     Running   0          4m49s
    	pod/alertmanager-main-2                   2/2     Running   0          4m49s
    	pod/blackbox-exporter-746c64fd88-j6sft    3/3     Running   0          4m53s
    	pod/grafana-55f8bc6d97-fmc5v              1/1     Running   0          4m52s
    	pod/kube-state-metrics-6c8846558c-2v6zt   3/3     Running   0          4m52s
    	pod/node-exporter-65666                   2/2     Running   0          4m51s
    	pod/node-exporter-dwmwn                   2/2     Running   0          4m51s
    	pod/node-exporter-t7vbx                   2/2     Running   0          4m51s
    	pod/node-exporter-wjkjz                   2/2     Running   0          4m51s
    	pod/prometheus-adapter-6455646bdc-mqxmc   1/1     Running   0          4m51s
    	pod/prometheus-adapter-6455646bdc-x9lf9   1/1     Running   0          4m51s
    	pod/prometheus-k8s-0                      2/2     Running   0          4m48s
    	pod/prometheus-k8s-1                      2/2     Running   0          4m48s
    	pod/prometheus-operator-f59c8b954-p7zjk   2/2     Running   0          4m51s
    	
    	NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
    	           AGE
    	service/alertmanager-main       ClusterIP   10.200.1.39    <none>        9093/TCP,8080/TCP 
    	           4m53s
    	service/alertmanager-operated   ClusterIP   None           <none>        9093/TCP,9094/TCP,9094/UDP   4m49s
    	service/blackbox-exporter       ClusterIP   10.200.1.23    <none>        9115/TCP,19115/TCP           4m53s
    	service/grafana                 ClusterIP   10.200.1.52    <none>        3000/TCP
    	           4m52s
    	service/kube-state-metrics      ClusterIP   None           <none>        8443/TCP,9443/TCP 
    	           4m52s
    	service/node-exporter           ClusterIP   None           <none>        9100/TCP
    	           4m51s
    	service/prometheus-adapter      ClusterIP   10.200.1.118   <none>        443/TCP
    	           4m51s
    	service/prometheus-k8s          ClusterIP   10.200.1.151   <none>        9090/TCP,8080/TCP 
    	           4m51s
    	service/prometheus-operated     ClusterIP   None           <none>        9090/TCP
    	           4m48s
    	service/prometheus-operator     ClusterIP   None           <none>        8443/TCP
    	           4m51s
    	
    	NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
    	daemonset.apps/node-exporter   4         4         4       4            4           kubernetes.io/os=linux   4m52s
    
    	NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
    	deployment.apps/blackbox-exporter     1/1     1            1           4m53s
    	deployment.apps/grafana               1/1     1            1           4m52s
    	deployment.apps/kube-state-metrics    1/1     1            1           4m52s
    	deployment.apps/prometheus-adapter    2/2     2            2           4m51s
    	deployment.apps/prometheus-operator   1/1     1            1           4m51s
    	
    	NAME                                            DESIRED   CURRENT   READY   AGE
    	replicaset.apps/blackbox-exporter-746c64fd88    1         1         1       4m53s
    	replicaset.apps/grafana-55f8bc6d97              1         1         1       4m52s
    	replicaset.apps/kube-state-metrics-6c8846558c   1         1         1       4m52s
    	replicaset.apps/prometheus-adapter-6455646bdc   2         2         2       4m51s
    	replicaset.apps/prometheus-operator-f59c8b954   1         1         1       4m51s
    	
    	NAME                                 READY   AGE
    	statefulset.apps/alertmanager-main   3/3     4m49s
    	statefulset.apps/prometheus-k8s      2/2     4m48s
    ```
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get crd -n monitoring
    	NAME                                        CREATED AT
    	alertmanagerconfigs.monitoring.coreos.com   2022-06-07T12:57:32Z
    	alertmanagers.monitoring.coreos.com         2022-06-07T12:57:32Z
    	innodbclusters.mysql.oracle.com             2022-06-07T11:28:01Z
    	mysqlbackups.mysql.oracle.com               2022-06-07T11:28:01Z
    	podmonitors.monitoring.coreos.com           2022-06-07T12:57:32Z
    	probes.monitoring.coreos.com                2022-06-07T12:57:32Z
    	prometheuses.monitoring.coreos.com          2022-06-07T12:57:33Z
    	prometheusrules.monitoring.coreos.com       2022-06-07T12:57:33Z
    	servicemonitors.monitoring.coreos.com       2022-06-07T12:57:33Z
    	thanosrulers.monitoring.coreos.com          2022-06-07T12:57:33Z
    ```
    
5. crd 통해 확인
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get prometheus -n monitoring 
    	NAME   VERSION   REPLICAS   AGE
    	k8s    2.36.0    2          39m
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get alertmanager -n monitoring 
    	NAME   VERSION   REPLICAS   AGE
    	main   0.24.0    3          39m
    ```
    
6. 외부접속을 위해 svc 수정
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get svc -n monitoring grafana       
    	NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
    	grafana   ClusterIP   10.200.1.52   <none>        3000/TCP   41m
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k edit svc/grafana -n monitoring      
    	service/grafana edited
    //TYPE=NodePort만 지정해 주면, nodeport 번호 자동 지정
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get svc -n monitoring grafana       
    	NAME      TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
    	grafana   NodePort   10.200.1.52   <none>        3000:30191/TCP   43m
    ```
    
    ![Untitled](Prometheus%20Custom%20Resource%20%E1%84%89%E1%85%A2%E1%86%BC%E1%84%89%E1%85%A5%E1%86%BC%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5%204e24cbd58f5a442ca897c4dccc71d10b/Untitled.png)
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get svc -n monitoring prometheus-k8s
    	
    	NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
    	prometheus-k8s   ClusterIP   10.200.1.151   <none>        9090/TCP,8080/TCP   62m
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k edit svc/prometheus-k8s -n monitoring
    	service/prometheus-k8s edited
    //TYPE=NodePort만 지정해 주면, nodeport 번호 자동 지정
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get svc -n monitoring prometheus-k8s
    	
    	NAME             TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
    	prometheus-k8s   NodePort   10.200.1.151   <none>        9090:32704/TCP,8080:32519/TCP   63m
    ```
    
    ![Untitled](Prometheus%20Custom%20Resource%20%E1%84%89%E1%85%A2%E1%86%BC%E1%84%89%E1%85%A5%E1%86%BC%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5%204e24cbd58f5a442ca897c4dccc71d10b/Untitled%201.png)
    

### Create Deployment, service, Service-Monitor, and HPA itself for testing

```java
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# vi servicemonitor.yaml
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k create -f servicemonitor.yaml -n mysql-cluster
	servicemonitor.monitoring.coreos.com/mycluster-router created
```

## Prometheus로 수집한 리소스를 기준으로 HPA 설정

## MySQL에 과부하줘서 3번에 설정한 HPA로 자동 scaling

# 참고문헌

[MySQL :: MySQL Operator for Kubernetes Manual :: 3.1 Deploy using Helm](https://dev.mysql.com/doc/mysql-operator/en/mysql-operator-innodbcluster-simple-helm.html)

[core_kubernetes/chapters/16 at master · bjpublic/core_kubernetes](https://github.com/bjpublic/core_kubernetes/tree/master/chapters/16)