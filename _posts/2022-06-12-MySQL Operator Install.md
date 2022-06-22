---
layout: single
title: "[Database/Kubernetes/DOIK] Prometheus로 수집한 metric으로 MySQL 데이터베이스의 HPA 조절하기 (1)"
excerpt: "Kubernetes에 MySQL Operator/InnoDB Cluster/Prometheus Operator 설치하기_작성중"
categories:
- Database
tag: [Kubernetes, 쿠버네티스, DevOps, AWS, CRD, CR, Custom Resource, 커스텀 리소스, MySQL Operator, MySQL InnoDB Cluster, Helm, 프로메테우스, Prometheus Operator, kube-prometheus]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---
  
 아직 작성중입니다...🥺  
  
# 0. 욕심으로 시작한 실습 도전기!😅

최근에 HPA 설정에 관해 작업하고 있었고, cpu나 memory 리소스 대신 다른 자원을 metric으로 수집하지 못할까 궁금했다. 그러던 중에 **Promethues**를 **Custom Resource**로 생성할 수 있다는 사실을 알고 흥미를 갖고 있었다.  
  
마침 타이밍이 딱 좋게 스터디에서도 Custom Resource가 언급되었고 바로 <u>Database와 Prometheus를 결합</u>한 실습을 해야겠다 생각했다.  
  
이번 포스팅에서 `MySQL Operator`, `InnoDB Cluster`, `Prometheus Operator`를 설치부터 해 볼 것이다.  
*(사실 Prometheus Operator를 설치할 때 살짝 헤매기도 했지만 욕심내니까 실습 진도가 너무 더뎠다 😢)*  

# 1. Prerequisites

먼저 가시다님이 제공해 주신 CloudFormation 템플릿으로 AWS에서 바닐라 쿠버네티스 클러스터를 생성해서 진행했다. **(항상 좋은 자료 감사합니다😆)**

이전 포스팅 참고 : [[Database/DOIK] AWS EC2에 Vanilla Kubernetes 실습 환경 배포](https://gain-yoo.github.io/database/DOIK-1%EC%B0%A8%EC%8B%9C-(1)/)
  
> 본 실습에서 사용한 spec :
>
> **OS** - Ubuntu 22.04 LTS  
> **Kubernetes** - v1.23.7  
> **Master 1개** - AWS t3.large (2cpu, ram4G)  
> **Node 3개** - AWS t3.medium (2cpu, ram8G)  
> **MySQL Operaotr/InnoDB Cluster** - v2.0.4  
> **Prometheus Operator** - main *(release-0.11와 유사)*
>

*위 포스팅과 스펙은 약간 다르다*

## 1) 실습하기 전에 MySQL & InnoDB에 대해 알고 가자 ☝
### MySQL Operator

![Medium](https://user-images.githubusercontent.com/100563973/173386387-e7568bb2-0eb0-415e-872e-63fa4f2042b9.jpg)  
*그림 출처: [https://blogs.oracle.com/mysql/post/mysql-operator-for-kubernetes-reaches-general-availability](https://blogs.oracle.com/mysql/post/mysql-operator-for-kubernetes-reaches-general-availability)*
  
- `MySQL Operator for Kubernetes` : MySQL InnoDB 클러스터 **관리나 자동화** 측면에서 편리하다.
- `MySQL InnoDB Cluster` : InnoDB Cluster는 3개 이상의 MySQL Server 인스턴스로 구성되며 HA 기능을 제공한다.

## 2) 실습하기 전에 Prometheus Operator도 알고 가자 ✌
Kubernetes에서 Prometheus를 사용하는 방법은 두 가지가 있다.
- `Prometheus` : Prometheus Server를 직접 생성하는 방법
- `Prometheus Operator` : Prometheus Operator를 이용하여 Prometheus Server를 생성하는 방법인데 **관리나 자동화** 측면에서 더 편리하다.
	- <u>생성/삭제</u> : Prometheus Operator를 사용하여 Prometheus 인스턴스를 쉽게 실행하고 삭제할 수 있다.
	- <u>단순 구성</u> : **Kubernetes의 리소스**를 이용하여 Prometheus 설정 구성이 가능하다.
	- <u>Label을 통한 대상 서비스</u> : **Label Query**를 기반으로 모니터링 대상 구성을 자동으로 생성할 수 있다.
  
위와 같이 Prometheus Operator을 사용하는 이유를 알 수 있었다.

![prometheus_operator_servicemonitor](https://user-images.githubusercontent.com/100563973/173386686-fa3be8bd-4ae4-44c6-9393-e47f70fc1693.png)
*그림 출처 : [https://sysdig.com/blog/kubernetes-monitoring-prometheus-operator-part3/](https://sysdig.com/blog/kubernetes-monitoring-prometheus-operator-part3/)*
  
위 그림처럼 `Prometheus` 인스턴스와 `ServiceMonitor`은 동일한 네임스페이스에 설치되어 있어야 한다.

### (1) 설치 버전
Prometheus Operator 버전 `0.39.0` 이상은 Kubernetes 버전 `1.16.0` 이상이어야 한다.  
Prometheus Operator를 처음 사용하는 경우 **최신 버전**을 사용하는 것이 좋으며, 이전 버전인 경우 <u>Kubernetes를 먼저 업그레이드하고</u> Prometheus Operator를 업그레이드할 것을 권장한다.

### (2) 설치 방법
Prometheus Operator 설치에는 아래 세 가지 방법이 있다.

1. **Prometheus Operator**
	- <u>Prometheus</u>, <u>Alertmanager</u> 및 관련 모니터링 구성 요소를 포함하여 Custom Resource로 배포한다.
	-  현재는 `prometheus-operator/prometheus-operator`라는 repository를 사용 중이지만 과거에는 `coreos/prometheus-operator`라는 repository를 사용하였다.
2. **kube-prometheus**
	- <u>Prometheus</u>, <u>Prometheus Operator</u>, <u>Alertmanager</u>, <u>node_exporter</u>, <u>Prometheus Adapter</u>, <u>kube-state-metrics</u>, <u>Grafana</u>를 포함해 설치한다.
3. **community helm chart**
	- <u>kube-prometheus와 비슷한 기능</u>을 제공하며 Prometheus 커뮤니티에서 관리하고 있다.
	- 현재는 `prometheus-community/kube-prometheus-stack helm`라는 repository를 사용 중이지만 과거에는 `stable/prometheus-operator`라는 repository를 사용하였다.
  
<br>

나는 1, 2번 둘다 설치해 봤는데 1번은 [약간의 트러블슈팅](https://gain-yoo.github.io/trouble%20shooting/custom-resource-error/)이 필요하여 따로 블로깅해 두었다.  
그리고 나같은 경우에는 구글링의 도움을 받으며 설치하다 보니 각각 사이트마다 설치하는 repository 명이 달라 좀 더 헤맨 감이 있었다.  
<u>설치 repository가 이전되면서 명칭이 달라진 거</u>였던 사소한 원인 ㅠㅠ...

# 2. MySQL Operator & InnoDB Cluster & Prometheus Operator
## 1) MySQL Operator 설치 with Helm

1. repo 추가 및 확인
    
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
    ```
2. 네임스페이스를 생성한 곳에 MySQL Operator를 설치
	```java
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
    
3. 설치 확인
    
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
    
4. 삭제
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm uninstall mysql-operator -n mysql-operator && kubectl delete ns mysql-operator
    	release "mysql-operator" uninstalled
    	namespace "mysql-operator" deleted
    ```
    

## 2) **MySQL InnoDB Cluster** 설치 with Helm

1. `tls.useSelfSigned` 사용, root 패스워드 지정, 네임스페이스를 생성한 곳에 MySQL InnoDB를 설치
    
    ```java
    (🍉 |kubernetes-admin@kubernetes:default) root@k8s-m:~# helm install mycluster mysql-operator/mysql-innodbcluster --set credentials.root.password='사용자 설정' --set tls.useSelfSigned=true --namespace mysql-cluster --create-namespace --version 2.0.4
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
    	    password: 
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
    

## 3) Prometheus Operator 생성

### (1) Install Prometheus Operator

prometheus operator를 설치하다가 오류가 나서 아래 링크와 같이 트러블슈팅을 진행했다.  

[[Kubernetes] Custom Resource Definition Install Error](https://gain-yoo.github.io/trouble%20shooting/custom-resource-error/)
  
prometheus operator로 설치하면 기본 namespace가 `default`로 잡혀 있다.  
<u>namespace를 변경해 주려 했지만</u> 이미 매니페스트에 정의가 되어 있으면 **namespace override가 안되는 듯하다** (아직까지 방법 못찾음. edit으로 수동 변경해 주면 될지도 모르겠지만….)  
암튼 그래서 패스하고 kube-prometheus로 namespace는 `monitoring`에서 설치해 줄 것이다.

### (2) Install kube-prometheus

1. Clone kube-prometheus
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~# git clone https://github.com/prometheus-operator/kube-prometheus.git
    ```
    
2. setup으로 crd와 namespace monitoring을 생성한다.
    
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
    
3. 그외 리소스도 생성해 주는데 여기서는 apply 대신 create로 리소스를 생성해 줬더니 오류나지 않았다
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# kubectl create -f manifests/
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
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get crd
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
    
5. crd 리소스 확인
    
    ```java
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get prometheus -n monitoring 
    	NAME   VERSION   REPLICAS   AGE
    	k8s    2.36.0    2          39m
    (🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k get alertmanager -n monitoring 
    	NAME   VERSION   REPLICAS   AGE
    	main   0.24.0    3          39m
    ```
    
# 3. HPA & Metric Server

## 1) 외부접속을 위해 svc 수정하고 `promethues`와 `grafana` 접속 확인
    
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

![Untitled](https://user-images.githubusercontent.com/100563973/173387955-ee43260a-7f0f-4786-8ce2-b5a35346439b.png)

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

![Untitled (1)](https://user-images.githubusercontent.com/100563973/173388065-4d6303cf-6e56-4aae-8b3d-c9507d0e2973.png)

## 2) ServiceMonitor & HPA 생성하려고 했지만....!

```java
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# vi autoscaling.yaml
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# cat autoscaling.yaml
	───────┬────────────────────────────────────────────────────
	       │ File: autoscaling.yaml
	───────┼────────────────────────────────────────────────────
	   1   │ apiVersion: monitoring.coreos.com/v1
	   2   │ kind: ServiceMonitor
	   3   │ metadata:
	   4   │   name: autoscaling-sm
	   5   │   namespace: monitoring
	   6   │   labels:
	   7   │     mysql.oracle.com/cluster: mycluster
	   8   │     tier: mysql
	   9   │ spec:
	  10   │   jobLabel: autoscalingmetrics
	  11   │   selector:
	  12   │     matchLabels:
	  13   │       mysql.oracle.com/cluster: mycluster
	  14   │       tier: mysql
	  15   │   namespaceSelector:
	  16   │     matchNames:
	  17   │     - mysql-cluster
	  18   │   endpoints:
	  19   │   - port: mysql
	  20   │     interval: 10s
	  21   │     path: /metrics
	  22   │ ---
	  23   │ apiVersion: autoscaling/v2beta1
	  24   │ kind: HorizontalPodAutoscaler
	  25   │ metadata:
	  26   │   name: autoscaling-app-hpa
	  27   │   namespace: mysql-cluster
	  28   │ spec:
	  29   │   scaleTargetRef:
	  30   │     apiVersion: apps/v1
	  31   │     kind: StatefulSet
	  32   │     name: mycluster
	  33   │   minReplicas: 3
	  34   │   maxReplicas: 10
	  35   │   metrics:
	  36   │   - type: Object
	  37   │     object:
	  38   │       target:
	  39   │         kind: Service
	  40   │         name: mycluster-instances
	  41   │       metricName: http_requests
	  42   │       targetValue: 5
	───────┴────────────────────────────────────────────────────
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# k create -f autoscaling.yaml
	servicemonitor.monitoring.coreos.com/autoscaling-sm created
	Warning: autoscaling/v2beta1 HorizontalPodAutoscaler is deprecated in v1.22+, unavailable in v1.25+; use autoscaling/v2 HorizontalPodAutoscaler
	horizontalpodautoscaler.autoscaling/autoscaling-app-hpa created

```
http_requests를 metric으로 잡고 mycluster를 hpa로 위와 같이 설정해 줬다. ServiceMonitor에서는 `labels`,`selector`,`namespaceSelector`,`enpoints.port`를 변경해 줬고 HPA에서는 `spec.kind`,`metadata.namespace`,`Replicas`,`spec.metrics 하위 값`을 변경해 주었다.  
  
```java
(🚴|DOIK-Lab:default) root@k8s-m:~# k describe hpa -n mysql-cluster autoscaling-app-hpa
	Warning: autoscaling/v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, unavailable in v1.26+; use autoscaling/v2 HorizontalPodAutoscaler
	Name:                                                             autoscaling-app-hpa
	Namespace:                                                        mysql-cluster
	Labels:                                                           <none>
	Annotations:                                                      <none>
	CreationTimestamp:                                                Wed, 22 Jun 2022 00:49:27 +0900
	Reference:                                                        StatefulSet/mycluster
	Metrics:                                                          ( current / target )
	  "http_requests" on Service/mycluster-instances (target value):  <unknown> / 5
	Min replicas:                                                     3
	Max replicas:                                                     10
	StatefulSet pods:                                                 3 current / 0 desired
	Conditions:
	  Type           Status  Reason                 Message
	  ----           ------  ------                 -------
	  AbleToScale    True    SucceededGetScale      the HPA controller was able to get the target's current scale
	  ScalingActive  False   FailedGetObjectMetric  the HPA was unable to compute the replica count: unable to get metric http_requests: Service on mysql-cluster mycluster-instances/unable to fetch metrics from custom metrics API: no custom metrics API (custom.metrics.k8s.io) registered
	Events:
	  Type     Reason                 Age                  From                       Message
	  ----     ------                 ----                 ----                       -------
	  Warning  FailedGetScale         41m (x121 over 71m)  horizontal-pod-autoscaler  statefulsets.apps "autoscaling-deploy" not found
	  Warning  FailedGetObjectMetric  74s (x145 over 37m)  horizontal-pod-autoscaler  unable to get metric http_requests: Service on mysql-cluster mycluster-instances/unable to fetch metrics from custom metrics API: no custom metrics API (custom.metrics.k8s.io) registered
```
그러나 http_requests이라는 metric을 찾을 수 없다는 에러 메시지를 뱉었다........  
  
![prometheus,hpa](https://user-images.githubusercontent.com/100563973/175080314-9f262c79-9efe-4e5a-ad76-6f74a7774b60.PNG)

이쯤에서 흐름도를 설명해 보자면 그림과 같다.  
  
위에서 설치한 Prometheus Adapter는 HPA를 사용하기에 가장 필요한 리소스다.  
Prometheus Adapter는 Prometheus Operator에 쿼리를 날린 후 Custom Metric 데이터를 가져와 API 서버에 제공하는 역할을 담당한다.  
나는 우선 MySQL 서버에 쿼리를 날릴 것이다. 이 때 서버가 받는 requests가 기준치를 넘어갈 때 HPA로 MySQL 서버를 autoscaling해 줄 것이다. 근데 그러려면 **https_request 리소스가 Custom metric으로 등록되어야 한다.**  
  
이 것이 바로 지금 내가 흐름도를 설명하는 이유이다..😂

### +) Custom metric 등록하기★★

```java
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq
	Error from server (NotFound): the server could not find the requested resource
```

이 명령어로 결과가 위와 같이 에러 메세지가 나온다면 metric 관련된 API가 생성되지 않은 것이다.  
  
metric API 종류는 아래와 같이 세 가지가 있다. *(내가 필요한건 custom metric)*
- `metrics` : CPU와 메모리 같은 기본 메트릭만 지원  
- `custom.metrics` : 기본 메트릭을 Kubernetes 오브젝트(http_requests, pod 개수 등)로 커스텀해서 사용
- `external.metrics` : Kubernetes 오브젝트가 아닌 외부 메트릭 수집

prometheus adapter를 생성할 때, `kube-prometheus/example.jsonnet` 파일에서 주석 해제하여 custom metric을 addon할 수 있다. 하지만 이 것도…에러가…….ㅠㅠ  
  
*링크 참고 : [kube-prometheus/example.jsonnet at main · prometheus-operator/kube-prometheus](https://github.com/prometheus-operator/kube-prometheus/blob/main/example.jsonnet#L8)*
  
```java
(🚴|DOIK-Lab:default) root@k8s-m:~/kube-prometheus# ./build.sh
	+ set -o pipefail
	++ pwd
	+ PATH=/root/kube-prometheus/tmp/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
	+ rm -rf manifests
	+ mkdir -p manifests/setup
	+ jsonnet -J vendor -m manifests example.jsonnet
	+ xargs '-I{}' sh -c 'cat {} | gojsontoyaml > {}.yaml' -- '{}'
	RUNTIME ERROR: couldn't open import "kube-prometheus/main.libsonnet": no match locally or in the Jsonnet library paths.
	        example.jsonnet:2:4-43  thunk <kp>
	        example.jsonnet:22:114-116      thunk <o>
	        std.jsonnet:1293:24
	        std.jsonnet:1293:5-33   function <anonymous>
	        example.jsonnet:22:97-136
	        example.jsonnet:22:15-137       thunk <a>
	        example.jsonnet:(20:1)-(23:2)   function <anonymous>
	        example.jsonnet:(20:1)-(23:2)
```

jsonnet-bundler를 설치하고 `build.sh`를 실행하라고 했지만..아직 헤매는 중이다 ㅠ  
*jsonnet-bundler 설치 링크 : [https://github.com/jsonnet-bundler/jsonnet-bundler](https://github.com/jsonnet-bundler/jsonnet-bundler)*

# 4. 참고 링크

- [MySQL :: MySQL Operator for Kubernetes Manual :: 3.1 Deploy using Helm](https://dev.mysql.com/doc/mysql-operator/en/mysql-operator-innodbcluster-simple-helm.html)  
- [core_kubernetes/chapters/16 at master · bjpublic/core_kubernetes](https://github.com/bjpublic/core_kubernetes/tree/master/chapters/16)  
- [Horizontal Pod Autoscaler 연습](https://kubernetes-docsy-staging.netlify.app/ko/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EC%98%A4%EB%B8%8C%EC%A0%9D%ED%8A%B8%EC%99%80-%EA%B4%80%EB%A0%A8%EC%9D%B4-%EC%97%86%EB%8A%94-%EB%A9%94%ED%8A%B8%EB%A6%AD%EC%9D%84-%EA%B8%B0%EC%B4%88%EB%A1%9C%ED%95%9C-%EC%98%A4%ED%86%A0%EC%8A%A4%EC%BC%80%EC%9D%BC%EB%A7%81)  
- [EKS AutoScaling 하기 Part 1 - Horizontal Pod Autoscaler With Custom Metrics](https://medium.com/@tkdgy0801/eks-autoscaling-%ED%95%98%EA%B8%B0-part-1-horizontal-pod-autoscaler-with-custom-metrics-2274566463f9)  
- [Hpa not fetching existing custom metric?](https://stackoverflow.com/questions/58151513/hpa-not-fetching-existing-custom-metric)  
- [GitHub - prometheus-operator/kube-prometheus: Use Prometheus to monitor Kubernetes and applications running on Kubernetes](https://github.com/prometheus-operator/kube-prometheus#prerequisites)  
