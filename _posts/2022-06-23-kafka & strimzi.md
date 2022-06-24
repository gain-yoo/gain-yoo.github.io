---
layout: single
title: "[Database/Kubernetes/DOIK] Kafka & Strimzi Operator - HTTP Bridge as a Sidecar (1)"
excerpt: "Kafka & Strimzi Operator 설명 및 설치_작성중"
categories:
- Database
tag: [DOIK, Kubernetes, 쿠버네티스, DevOps, AWS, CRD, CR, Custom Resource, 커스텀 리소스, Helm, Kafka, Strimzi, HTTP Bridge, Sidecar]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

# 1. Kafka & Strimzi Operator

## 1) Kafka란?

- 비동기식, 스트리밍 플랫폼, 데이터허브 느낌….

무슨 말인지???? 일단 아래부터 이해해 보자!

<aside>
❓ **메시지** 브로커 (RabbitMQ) vs **이벤트** 브로커 (Kafka) 차이?

</aside>

- **메시지** 브로커 : 메시지 전송 → 처리 → 삭제
- **이벤트** 브로커 : 메시지 브로커와 유사하지만, 삭제는 없이! 데이터 베이스에 저장하듯이 이벤트 브로커의 큐에 저장!
    - **이벤트** 하나만 보관하여 인덱스를 통해 개별 액세스 관리
    - 필요한 시간 동안에는 이벤트 **보존** 가능
    - 장점👍
        1. 한번 일어난 이벤트 데이터를 브로커에 저장함으로 **단일 진실 공급원**(Single Source of Truth, SSOT)으로 사용 가능  
        <i>(* SSOT : 집약 데이터의 축척은 한곳에서 담당하고 데이터의 활용은 참조 형식으로 다수 사용하는 형태)</i>
        2. 장애 발생 시 **장애 발생 시점부터** 다시 처리 가능
        3. 많은 양의 **실시간 스트림 데이터**를 효과적으로 처리 가능

*참고 영상 : [카프카, 레빗엠큐, 레디스 큐의 큰 차이점! 이벤트 브로커와 메시지 브로커에 대해 알아봅시다.](https://www.youtube.com/watch?v=H_DaPyUOeTo)*

<aside>
❓ **스트림** 데이터? 는 **기존** 데이터와 무엇이 다를까요?

</aside>

- **기존** 데이터 : 데이터 수집 → 처리 및 구조화 → 구축
    
    📌 keypoint : 시간의 경과에 따라 **데이터 일괄 처리**
    
- **스트림** 데이터 : 실시간 이동 데이터의 이용/저장/보강/분석
    
    📌 keypoint : **저장(Storage) & 처리(Processing)**

*참고 링크 : [What is Streaming Data? How it Works, Examples, and Use Cases | KR](https://www.confluent.io/ko-kr/learn/data-streaming/)*

<aside>
❓ **이벤트** 처리 vs **이벤트 스트림** 처리?

</aside>

- **이벤트** 처리 : **시간별로** 정렬된 **개별 이벤트**를 한 번에 하나씩 확인
- **이벤트 스트림** 처리 : **대량** 이벤트를 **실시간으로** 함께 처리

*참고 링크 : [이벤트 스트림 처리란 무엇입니까?](https://www.tibco.com/ko/reference-center/what-is-event-stream-processing)*

## 2) Strimzi Operator란?
- Strimzi 는 Kubernetes 환경에서 Kafka 운영 관리에 도움을 주는 Operator 다.

## 3) 브리지 사이드카로 애플리케이션을 배포하기 전에 준비

### (1) Strimzi Cluster Operator 설치

1. namespace 생성
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl create namespace kafka
    	namespace/kafka created
    ```
    
2. repo 추가
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# helm repo add strimzi https://strimzi.io/charts/
    	"strimzi" has been added to your repositories
    (🚴|DOIK-Lab:default) root@k8s-m:~# helm show values strimzi/strimzi-kafka-operator
    ```
    
3. Control Plane에 Operator Pod 설치
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# printf 'tolerations: [{key: node-role.kubernetes.io/master, operator: Exists, effect: NoSchedule}]\n' | \
    helm install kafka-operator strimzi/strimzi-kafka-operator --version 0.29.0 --namespace kafka \
      --set nodeSelector."kubernetes\.io/hostname"=k8s-m --values /dev/stdin
    ```
     
4. deployment, pod 리소스 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get deploy,pod -n kafka
    	NAME                                       READY   to UP-TO-DATE   AVAILABLE   AGE
    	deployment.apps/strimzi-cluster-operator   1/1     1            1           3m54s
    	
    	NAME                                            READY   STATUS    RESTARTS   AGE
    	pod/strimzi-cluster-operator-555b78d767-tzft6   1/1     Running   0          3m54s
    ```
    
5. crd 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get crd
    	NAME                                  CREATED AT
    	kafkabridges.kafka.strimzi.io         2022-06-23T14:16:37Z
    	kafkaconnectors.kafka.strimzi.io      2022-06-23T14:16:37Z
    	kafkaconnects.kafka.strimzi.io        2022-06-23T14:16:37Z
    	kafkamirrormaker2s.kafka.strimzi.io   2022-06-23T14:16:37Z
    	kafkamirrormakers.kafka.strimzi.io    2022-06-23T14:16:37Z
    	kafkarebalances.kafka.strimzi.io      2022-06-23T14:16:37Z
    	kafkas.kafka.strimzi.io               2022-06-23T14:16:36Z
    	kafkatopics.kafka.strimzi.io          2022-06-23T14:16:37Z
    	kafkausers.kafka.strimzi.io           2022-06-23T14:16:37Z
    	strimzipodsets.core.strimzi.io        2022-06-23T14:16:37Z
    ```
    
6. 삭제
    
    ```java
    helm uninstall kafka-operator -n kafka && kubectl delete ns kafka
    ```
    

### (2) 포트 9093에서 TLS 클라이언트 인증이 활성화되고 권한 부여도 활성화된 Kafka 클러스터를 배포

1. kafka.yaml 확인 (3.1.2 버전 설치)
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# cat kafka.yaml
    
    	───────┬───────────────────────────────────────────────────────────────────
    	       │ File: kafka.yaml
    	───────┼───────────────────────────────────────────────────────────────────
    	   1   │ apiVersion: kafka.strimzi.io/v1beta2
    	   2   │ kind: Kafka
    	   3   │ metadata:
    	   4   │   name: my-cluster
    	   5   │ spec:
    	   6   │   kafka:
    	   7   │     #version: 3.1.1
    	   8   │     replicas: 3
    	   9   │     listeners:
    	  10   │       - name: plain
    	  11   │         port: 9092
    	  12   │         type: internal
    	  13   │         tls: false
    	  14   │       - name: tls
    	  15   │         port: 9093
    	  16   │         type: internal
    	  17   │         tls: false
    	  18   │       - name: external
    	  19   │         port: 9094
    	  20   │         type: nodeport
    	  21   │         tls: false
    	  22   │     storage:
    	  23   │       type: jbod
    	  24   │       volumes:
    	  25   │       - id: 0
    	  26   │         type: persistent-claim
    	  27   │         size: 10Gi
    	  28   │         deleteClaim: true
    	  29   │     config:
    	  30   │       offsets.topic.replication.factor: 3
    	  31   │       transaction.state.log.replication.factor: 3
    	  32   │       transaction.state.log.min.isr: 2
    	  33   │       default.replication.factor: 3
    	  34   │       min.insync.replicas: 2
    	  35   │       #inter.broker.protocol.version: "3.1.1"
    	  36   │     template:
    	  37   │       pod:
    	  38   │         affinity:
    	  39   │           podAntiAffinity:
    	  40   │             requiredDuringSchedulingIgnoredDuringExecution:
    	  41   │               - labelSelector:
    	  42   │                   matchExpressions:
    	  43   │                     - key: app.kubernetes.io/name
    	  44   │                       operator: In
    	  45   │                       values:
    	  46   │                         - kafka
    	  47   │                 topologyKey: "kubernetes.io/hostname"
    	  48   │   zookeeper:
    	  49   │     replicas: 3
    	  50   │     storage:
    	  51   │       type: persistent-claim
    	  52   │       size: 10Gi
    	  53   │       deleteClaim: true
    	  54   │     template:
    	  55   │       pod:
    	  56   │         affinity:
    	  57   │           podAntiAffinity:
    	  58   │             requiredDuringSchedulingIgnoredDuringExecution:
    	  59   │               - labelSelector:
    	  60   │                   matchExpressions:
    	  61   │                     - key: app.kubernetes.io/name
    	  62   │                       operator: In
    	  63   │                       values:
    	  64   │                         - zookeeper
    	  65   │                 topologyKey: "kubernetes.io/hostname"
    	  66   │   entityOperator:
    	  67   │     topicOperator: {}
    	  68   │     userOperator: {}
    	───────┴───────────────────────────────────────────────────────────────────
    ```
    
    > ”배포 시 **requiredDuringSchedulingIgnoredDuringExecution 지원** , ~~preferredDuringSchedulingIgnoredDuringExecution **미지원**~~...(상당한 삽질...)”
	  
	💡 가시다님이 위와 같이 말씀하신게 궁금해서 이 옵션 값에 대해 찾아 보았는데 [같은 이슈를 가진 케이스가 있었다.](https://github.com/strimzi/strimzi-kafka-operator/issues/2280)  
	  
	좀 더 찾아 보니 아래 두 가지 글을 발견했다. 아래와 같은 이유로 `requiredDuringSchedulingIgnoredDuringExecution`를 사용하는 게 아닐까 조심스레 추측해 본다.  
	  
	> 2.7.1.1. Use pod anti-affinity to avoid critical applications sharing nodes
	>  
	> **Use pod anti-affinity to ensure that critical applications are never scheduled on the same disk.** When running a Kafka cluster, it is recommended to use pod **anti-affinity** to ensure that the Kafka brokers do not share nodes with other workloads, such as databases.  
	  
	> However, the `preferredDuringSchedulingIgnoredDuringExecution` rule does **not guarantee that the brokers will be spread.** Depending on your exact OpenShift and Kafka configurations, you should add additional affinity rules or configure topologySpreadConstraints for both ZooKeeper and Kafka to make sure the nodes are properly distributed accross as many racks as possible  
	  
	[링크 참고 1](https://access.redhat.com/documentation/en-us/red_hat_amq_streams/2.1/html/configuring_amq_streams_on_openshift/assembly-deployment-configuration-str#con-scheduling-to-specific-nodes-str)    [링크 참고 2](https://access.redhat.com/documentation/en-us/red_hat_amq_streams/2.1/html/configuring_amq_streams_on_openshift/api_reference-str#property-listener-config-preferredNodePortAddressType-reference)  
	  
    - podAffinity & pod**Anti**Affinity
		- node에서 이미 실행 중인 pod 레이블을 기반으로 `podAffinity`는 <u>동일한 label을 가진 pod가 동일 영역에 스케줄링되게</u> 해 주는 설정 값이고 반대로 `podAntiAffinity`는 HA 구성할 때와 같이 <u>동일한 label을 pod가 서로 다른 영역에 스케줄링되게</u> 해 주는 설정 값이다.  
        *(영역 : node, rack, cloud provider zone or region과 같은 topology domain)*  
		*(pod : LabelSelector에 영향받음)*
    - **required**DuringSchedulingIgnoredDuringExecution & **preferred**DuringSchedulingIgnoredDuringExecution
		- required(hard affinity) : 반드시 조건에 맞아야 해당 영역에만 배포됨
		- preferred(soft affinity) : 되도록 조건에 맞는다면 해당 영역에 배포됨 (우선시하되 필수는 아니고 weight 옵션을 통해 우선순위 설정 가능)
        - 즉, 위 매니페스트 파일에 의하면
        **`kafka` 또는 `zookeeper`는 *app.kubernetes.io/name=kafka* 또는 *app.kubernetes.io/name=zookeeper*인 조건 하나만 충족한다면 동일한 label의 pod끼리는 반드시 다른 node에 스케줄되어야 한다는 것을 의미한다.** ⇒ <u>고가용성</u>✨
	  
    *링크 참고 : [노드에 파드 할당하기](https://kubernetes.io/ko/docs/concepts/scheduling-eviction/assign-pod-node/#%ED%8C%8C%EB%93%9C%EA%B0%84-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0%EC%99%80-%EC%95%88%ED%8B%B0-%EC%96%B4%ED%94%BC%EB%8B%88%ED%8B%B0)*    
    
2. 클러스터 배포
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl apply -f kafka.yaml -n kafka
    	kafka.kafka.strimzi.io/my-cluster created
    ```
    
3. 배포된 클러스터 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get kafka -n kafka
    	NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   WARNINGS
    	my-cluster   3                        3                     True
    ```
    
4. statefulset으로 설치된 kafka, zookeeper 리소스 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get sts -n kafka -owide
    	NAME                   READY   AGE   CONTAINERS   IMAGES
    	my-cluster-kafka       3/3     11m   kafka        quay.io/strimzi/kafka:0.29.0-kafka-3.2.0
    	my-cluster-zookeeper   3/3     12m   zookeeper    quay.io/strimzi/kafka:0.29.0-kafka-3.2.0
    ```
    
5. kafkatopics crd 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get kafkatopics -n kafka
    	NAME                                                                                               CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
    	consumer-offsets---84e7a678d08f4bd226872e5cdd4eb527fadc1c6a                                        my-cluster   50           3                    True
    	strimzi-store-topic---effb8e3e057afce1ecf67c3f5d8e4e3ff177fc55                                     my-cluster   1            3                    True
    	strimzi-topic-operator-kstreams-topic-store-changelog---b75e702040b99be8a9263134de3507fc0cc4017b   my-cluster   1            3                    True
    ```
    
6. service와 configmap 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~#  kubectl get svc,configmap -n kafka
    	NAME                                          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                               AGE
    	service/my-cluster-kafka-0                    NodePort    10.200.1.247   <none>        9094:31250/TCP                        5h38m
    	service/my-cluster-kafka-1                    NodePort    10.200.1.11    <none>        9094:32516/TCP                        5h38m
    	service/my-cluster-kafka-2                    NodePort    10.200.1.12    <none>        9094:30658/TCP                        5h38m
    	service/my-cluster-kafka-bootstrap            ClusterIP   10.200.1.133   <none>        9091/TCP,9092/TCP,9093/TCP            5h38m
    	service/my-cluster-kafka-brokers              ClusterIP   None           <none>        9090/TCP,9091/TCP,9092/TCP,9093/TCP   5h38m
    	service/my-cluster-kafka-external-bootstrap   NodePort    10.200.1.115   <none>        9094:31094/TCP                        5h38m
    	service/my-cluster-zookeeper-client           ClusterIP   10.200.1.35    <none>        2181/TCP                              5h39m
    	service/my-cluster-zookeeper-nodes            ClusterIP   None           <none>        2181/TCP,2888/TCP,3888/TCP            5h39m
    	
    	NAME                                                DATA   AGE
    	configmap/kube-root-ca.crt                          1      6h27m
    	configmap/my-cluster-entity-topic-operator-config   1      5h37m
    	configmap/my-cluster-entity-user-operator-config    1      5h37m
    	configmap/my-cluster-kafka-config                   5      5h38m
    	configmap/my-cluster-zookeeper-config               2      5h39m
    	configmap/strimzi-cluster-operator                  1      5h52m
    ```
    