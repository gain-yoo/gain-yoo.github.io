---
layout: single
title: "[Database/Kubernetes/DOIK] Kafka & Strimzi Operator (2) - HTTP Bridge as a Sidecar "
excerpt: "메인 애플리케이션이 kafka 클라이언트 지원이 어려울 경우, HTTP로 Bridge Sidecar Container 활용하는 실습"
categories:
- Database
tag: [DOIK, Kubernetes, 쿠버네티스, DevOps, AWS, CRD, CR, Custom Resource, 커스텀 리소스, Helm, Kafka, Strimzi, HTTP Bridge, Sidecar]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---

# 🍭 HTTP Bridge를 Kubernetes Sidecar로 사용

## 1) Sidecar란?

![Untitled (2)](https://user-images.githubusercontent.com/100563973/175779063-d0bc9db7-6073-44be-8e2b-921c06ec3184.png)

- 한 Pod 안에 두 개 이상의 Container가 존재한다.
- 서비스를 micro 단위로 쪼개서 pod 하나에 애플리케이션 하나가 아닌, **container 하나에 애플리케이션 하나**인 구성이 가능하다.
- 따라서 **애플리케이션 변경 없이** 기능을 확장할 수 있다.
- 애플리케이션 하나에 장애가 나도 다른 애플리케이션에 지장은 없지만 pod를 재기동할 때는 모든 Container가 동시에 재기동된다.
- 외부 네트워크 없이 pod 내에서 내부적으로 **Container 간에 통신이 가능**하다.

## 2) Sidecar로 Strimzi HTTP Bridge

![Untitled (3)](https://user-images.githubusercontent.com/100563973/175779067-c66d83c4-7ba8-46c3-8ef8-4d745c2119e8.png)

*그림 출처 : [Using HTTP Bridge as a Kubernetes sidecar](https://strimzi.io/blog/2021/08/18/using-http-bridge-as-a-kubernetes-sidecar/)*

- 메인 애플리케이션이 kafka 클라이언트 지원이 어려울 경우, HTTP로 Bridge Sidecar Container 활용
- Pod 내 Container끼리 내부 통신하므로 Bridge의 HTTP 인터페이스 보안 걱정은 덜 수 있다.

## 3) Bridge Sidecar로 애플리케이션을 배포하기 전에 준비 실습

### (1) 포트 9093에서 TLS 클라이언트 인증 & 권한 부여 활성화

1. `kubectl edit`으로 클러스터 수정
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl edit -f kafka.yaml -n kafka
    	kafka.kafka.strimzi.io/my-cluster edited
    
    //아래와 같이 수정
    ...생략...
      kafka:
        replicas: 3
        listeners:
          - name: tls
            port: 9093
            type: internal
            **tls: true**
            **authentication:
              type: tls
        authorization:
          type: simple**
    ...생략...
    ```
    kafka 클러스터를 구성할 때 `tls`는 false로 설정해서 **true**로 다시 설정했다.

2. 정상 수정 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get kafkas -n kafka
    	NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   WARNINGS
    	my-cluster   3                        3                     True
    ```
    
3. kafka 클러스터 Listeners 정보 확인 : 9093 TLS
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get kafka -n kafka my-cluster -o jsonpath={.status} | jq -r ".listeners[1]"
    	{
    	  "addresses": [
    	    {
    	      "host": "my-cluster-kafka-bootstrap.kafka.svc",
    	      "port": 9093
    	    }
    	  ],
    	  "bootstrapServers": "my-cluster-kafka-bootstrap.kafka.svc:9093",
    	  "certificates": [
    	    "-----BEGIN CERTIFICATE-----\nMIIFLTC....중략...skCF0osWi92Q\n-----END CERTIFICATE-----\n"
    	  ],
    	  "name": "tls",
    	  "type": "tls"
    	}
    ```
    

### (2) TLS 클라이언트 인증을 위한 사용자 구성 & 권한 부여
클러스터가 실행되면 Bridge Sidecar에서 사용할 사용자가 필요하다.

1. bridge-user.yaml 내용 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# cat bridge-user.yaml
    ───────┬─────────────────────────────────────────────────────────────────────────
           │ File: /root/bridge-user.yaml
    ───────┼─────────────────────────────────────────────────────────────────────────
       1   │ apiVersion: kafka.strimzi.io/v1beta2
       2   │ kind: KafkaUser
       3   │ metadata:
       4   │   name: bridge
       5   │   labels:
       6   │     strimzi.io/cluster: my-cluster
       7   │ spec:
       8   │   authentication:
       9   │     type: tls
      10   │   authorization:
      11   │     type: simple
      12   │     acls:
      13   │       # Consume from topic my-topic using consumer group my-group
      14   │       - resource:
      15   │           type: topic
      16   │           name: my-topic
      17   │           patternType: literal
      18   │         operation: Read
      19   │         host: "*"
      20   │       - resource:
      21   │           type: group
      22   │           name: my-group
      23   │           patternType: literal
      24   │         operation: Read
      25   │         host: "*"
      26   │       # Producer messages to topic my-topic
      27   │       - resource:
      28   │           type: topic
      29   │           name: my-topic
      30   │           patternType: literal
      31   │         operation: Write
      32   │         host: "*"
    ───────┴─────────────────────────────────────────────────────────────────────────
    ```
    `authentication.type: tls`로 TLS 클라이언트 인증을 구성하고 `acls`로 topic에서 읽기/쓰기 권한을 구성한다.
    
2. KafkaUser 생성 및 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl apply -f bridge-user.yaml -n kafka
    	kafkauser.kafka.strimzi.io/bridge created
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get kafkauser -n kafka
    	NAME     CLUSTER      AUTHENTICATION   AUTHORIZATION   READY
    	bridge   my-cluster   tls              simple          True
    ```
    

### (3) 메시지를 보내고 받을 topic 만들기

1. bridge-topic.yaml 내용 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# cat bridge-topic.yaml
    ───────┬────────────────────────────────────────────
           │ File: /root/bridge-topic.yaml
    ───────┼────────────────────────────────────────────
       1   │ apiVersion: kafka.strimzi.io/v1beta2
       2   │ kind: KafkaTopic
       3   │ metadata:
       4   │   name: my-topic
       5   │   labels:
       6   │     strimzi.io/cluster: my-cluster
       7   │ spec:
       8   │   partitions: 1
       9   │   replicas: 1
    ───────┴────────────────────────────────────────────
    ```
    
2. KafkaTopic 생성 및 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl apply -f bridge-topic.yaml -n kafka
    	kafkatopic.kafka.strimzi.io/my-topic created
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get kafkatopic -n kafka my-topic
    	NAME       CLUSTER      PARTITIONS   REPLICATION FACTOR   READY
    	my-topic   my-cluster   1            1                    True
    ```
    

## 4) Bridge Sidecar 배포 실습

Cluster Operator를 사용하여 Strimzi HTTP Bridge를 배포하게 되면 사용자는 KafkaBridge Custom Resource만 생성하면 나머지는 Operator가 해 준다.

그러나!!!! Sidecar로 구성하게 된다면 사용자가 직접 아래와 같이 구성해야 한다.

> 1. bridge configuration을 설정한 ConfigMap 생성
> 2. Pod에 파일 마운트
> 3. bridge 시작 시 사용
> 

이 때, ConfigMap 구성은 **Apache Kafka configuration providers**를 사용한다.

### (1) Bridge **구성**

1. bridge-configmap.yaml 내용 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# cat bridge-configmap.yaml
    	───────┬──────────────────────────────────────────────────────────────────────────────────────
    	       │ File: /root/bridge-configmap.yaml
    	───────┼──────────────────────────────────────────────────────────────────────────────────────
    	   1   │ apiVersion: v1
    	   2   │ kind: ConfigMap
    	   3   │ metadata:
    	   4   │   name: bridge-configuration
    	   5   │ data:
    	   6   │   bridge.properties: |
    	   7   │     bridge.id=bridge-sidecar
    	   8   │
    	   9   │     # HTTP related settings
    	  10   │     http.enabled=true
    	  11   │     http.host=127.0.0.1
    	  12   │     http.port=8080
    	  13   │
    	  14   │     # Configuration Providers
    	  15   │     kafka.config.providers=env
    	  16   │     kafka.config.providers.env.class=io.strimzi.kafka.EnvVarConfigProvider
    	  17   │
    	  18   │     # General Kafka settings
    	  19   │     kafka.bootstrap.servers=${env:BOOTSTRAP_SERVERS}
    	  20   │     kafka.security.protocol=SSL
    	  21   │     kafka.ssl.keystore.type=PEM
    	  22   │     kafka.ssl.keystore.certificate.chain=${env:USER_CRT}
    	  23   │     kafka.ssl.keystore.key=${env:USER_KEY}
    	  24   │     kafka.ssl.truststore.type=PEM
    	  25   │     kafka.ssl.truststore.certificates=${env:CA_CRT}
    	  26   │     kafka.ssl.endpoint.identification.algorithm=HTTPS
    	  27   │
    	  28   │     # Kafka Producer options
    	  29   │     kafka.producer.acks=1
    	  30   │
    	  31   │     # Kafka Consumer options
    	  32   │     kafka.consumer.auto.offset.reset=earliest
    	───────┴──────────────────────────────────────────────────────────────────────────────────────
    ```

    - bridge.id : bridge 인스턴스 항목 지정
	- HTTP related settings : 같은 pod 내에서만 통신하기 위해 로컬 주소인 `127.0.0.1`과 pod 내 통신 기본 포트인 `8080`으로 설정한다.
	- Apache Kafka APIs (Consumer API/Producer API/Admin API) : configuration providers, bootstrap servers, authentication 설정
		- 환경변수 설정으로 `Strimzi EnvVar Configuration Provider`를 initialize한다.
		- 커스터마이징을 하고 싶다면 Apache Kafka의 FileConfigProvider/DirectoryConfigProvider 또는 Container 이미지에 포함되어 있는 Strimzi Kubernetes Configuration Provider를 사용하면 된다.

2. Config 생성 및 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl apply -f bridge-configmap.yaml -n kafka
    	configmap/bridge-configuration created
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get configmaps -n kafka bridge-configuration
    	NAME                   DATA   AGE
    	bridge-configuration   1      42s
    ```
    

### (2) Sidecar 배포

1. bridge-pod-sidecar.yaml 내용 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# cat bridge-pod-sidecar.yaml
    	───────┬───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    	       │ File: /root/bridge-pod-sidecar.yaml
    	───────┼───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    	   1   │ apiVersion: v1
    	   2   │ kind: Pod
    	   3   │ metadata:
    	   4   │   name: bridge-sidecar
    	   5   │ spec:
    	   6   │   containers:
    	   7   │     - name: main
    	   8   │       image: centos:7
    	   9   │       command: ["sh", "-c", "sleep 3600"]
    	  10   │     - name: bridge
    	  11   │       image: quay.io/strimzi/kafka-bridge:0.21.5
    	  12   │       command: ["/opt/strimzi/bin/kafka_bridge_run.sh", "--config-file", "/etc/strimzi-bridge/bridge.properties"]
    	  13   │       env:
    	  14   │         - name: BOOTSTRAP_SERVERS
    	  15   │           value: my-cluster-kafka-bootstrap:9093
    	  16   │         - name: USER_CRT
    	  17   │           valueFrom:
    	  18   │             secretKeyRef:
    	  19   │               name: bridge
    	  20   │               key: user.crt
    	  21   │         - name: USER_KEY
    	  22   │           valueFrom:
    	  23   │             secretKeyRef:
    	  24   │               name: bridge
    	  25   │               key: user.key
    	  26   │         - name: CA_CRT
    	  27   │           valueFrom:
    	  28   │             secretKeyRef:
    	  29   │               name: my-cluster-cluster-ca-cert
    	  30   │               key: ca.crt
    	  31   │       volumeMounts:
    	  32   │         - name: bridge-configuration
    	  33   │           mountPath: /etc/strimzi-bridge
    	  34   │   volumes:
    	  35   │     - name: bridge-configuration
    	  36   │       configMap:
    	  37   │         name: bridge-configuration
    	  38   │   restartPolicy: Never
    	───────┴───────────────────────────────────────────────────────────────────────────────────────────────────────────────────
    ```
    - main container : centos7으로 기동
	- bridge container : ConfigMap 마운트(bridge.properties), cert는 환경변수로 지정, BOOTSTRAP_SERVERS 지정
		- 궁금한 kafka_bridge_run.sh 파일
			
			```java
			[strimzi@bridge-sidecar strimzi]$ cat /opt/strimzi/bin/kafka_bridge_run.sh
				#!/bin/sh
				set -x
				
				# Find my path to use when calling scripts
				MYPATH="$(dirname "$0")"
				
				# Configure logging
				if [ -z "$KAFKA_BRIDGE_LOG4J_OPTS" ]
				then
					KAFKA_BRIDGE_LOG4J_OPTS="-Dlog4j2.configurationFile=file:${MYPATH}/../config/log4j2.properties"
				fi
				
				# Make sure that we use /dev/urandom
				JAVA_OPTS="${JAVA_OPTS} -Dvertx.cacheDirBase=/tmp/vertx-cache -Djava.security.egd=file:/dev/./urandom"
				
				exec java $JAVA_OPTS $KAFKA_BRIDGE_LOG4J_OPTS -classpath "${MYPATH}/../libs/*" io.strimzi.kafka.bridge.Application "$@"
			```

2. Pod 생성 및 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl apply -f bridge-pod-sidecar.yaml -n kafka
    	pod/bridge-sidecar created
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get pod -n kafka bridge-sidecar
    	NAME             READY   STATUS    RESTARTS   AGE
    	bridge-sidecar   2/2     Running   0          64s
    ```
    
3. Secret 확인
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl get secret -n kafka bridge -o json | jq
    	{
    	  "apiVersion": "v1",
    	  "data": {
    	    "ca.crt": "LS0tLS1...중략...FLS0tLS0K",
    	    "user.crt": "LS0tLS1Ca...중략...VZLS0tLS0K",
    	    "user.p12": "MIIKhgI+L1...중략...1yAgIIAA==",
    	    "user.password": "패스워드"
    	  },
    	  "kind": "Secret",
    	  "metadata": {
    	    "creationTimestamp": "2022-06-25T04:00:24Z",
    	    "labels": {
    	      "app.kubernetes.io/instance": "bridge",
    	      "app.kubernetes.io/managed-by": "strimzi-user-operator",
    	      "app.kubernetes.io/name": "strimzi-user-operator",
    	      "app.kubernetes.io/part-of": "strimzi-bridge",
    	      "strimzi.io/cluster": "my-cluster",
    	      "strimzi.io/kind": "KafkaUser"
    	    },
    	    "name": "bridge",
    	    "namespace": "kafka",
    	    "ownerReferences": [
    	      {
    	        "apiVersion": "kafka.strimzi.io/v1beta2",
    	        "blockOwnerDeletion": false,
    	        "controller": false,
    	        "kind": "KafkaUser",
    	        "name": "bridge",
    	        "uid": "70522019-f5c2-4843-8e6c-219e2f8cdfa9"
    	      }
    	    ],
    	    "resourceVersion": "10873",
    	    "uid": "6baec932-fef9-493d-8ca5-4d328cf3af6d"
    	  },
    	  "type": "Opaque"
    	}
    ```
    
### (3) Sidecar 사용

1. sidecar의 main container로 bash 쉘 접속
    
    ```java
    (🚴|DOIK-Lab:default) root@k8s-m:~# kubectl exec -ti -n kafka bridge-sidecar -c main -- bash
    ```
    
2. POST 요청으로 메시지 보내고 topic 리스트 조회
    
    ```java
    [root@bridge-sidecar /]# curl -X POST http://localhost:8080/topics/my-topic \
    	> -H 'Content-Type: application/vnd.kafka.json.v2+json' \
    	> -d '{ "records": [ { "value": "Hello World!" } ] }' ; echo
    		{"offsets":[{"partition":0,"offset":0}]}
    [root@bridge-sidecar /]# curl -X GET http://localhost:8080/topics ; echo
    	["my-topic"]
    ```
    
3. consumer 그룹 생성
    
    ```java
    [root@bridge-sidecar /]# curl -X POST http://localhost:8080/consumers/my-group \
    > -H 'Content-Type: application/vnd.kafka.v2+json' \
    > -d '{
    >            "name": "my-consumer",
    >            "auto.offset.reset": "earliest",
    >            "format": "json",
    >            "enable.auto.commit": true,
    >            "fetch.min.bytes": 512,
    >            "consumer.request.timeout.ms": 30000
    >          }' ; echo
    	{"instance_id":"my-consumer","base_uri":"http://localhost:8080/consumers/my-group/instances/my-consumer"}
    ```
    
4. topic 정보 확인
    
    ```java
    [root@bridge-sidecar /]# curl -X GET http://localhost:8080/topics/my-topic ; echo
    	{"error_code":500,"message":"Topic authorization failed."}
    -----------------------------------------------------------------------------------
    [2022-06-25 13:47:48,524] INFO  <getTopic    :85> [oop-thread-1] [1814703591] GET_TOPIC Request: from 127.0.0.1:40374, method = GET, path = /topics/my-topic
    [2022-06-25 13:47:48,525] INFO  <ientEndpoint:96> [oop-thread-1] Describe topics [my-topic]
    [2022-06-25 13:47:48,525] INFO  <ientEndpoint:104> [oop-thread-1] Describe configs [ConfigResource{name=my-topic,type=TOPIC,isDefault=false}]
    [2022-06-25 13:47:48,526] INFO  <getTopic    :85> [oop-thread-1] [1814703591] GET_TOPIC Response:  statusCode = 200, message = OK
    ```
    
5. subscribe to my-topic
    
    ```java
    [root@bridge-sidecar /]# curl -v -X POST http://localhost:8080/consumers/my-group/instances/my-consumer/subscription \
    > -H 'Content-Type: application/vnd.kafka.v2+json' \
    > -d '{
    >            "topics": [
    >              "my-topic"
    >            ]
    >          }'
    	* About to connect() to localhost port 8080 (#0)
    	*   Trying ::1...
    	* Connection refused
    	*   Trying 127.0.0.1...
    	* Connected to localhost (127.0.0.1) port 8080 (#0)
    	> POST /consumers/my-group/instances/my-consumer/subscription HTTP/1.1
    	> User-Agent: curl/7.29.0
    	> Host: localhost:8080
    	> Accept: */*
    	> Content-Type: application/vnd.kafka.v2+json
    	> Content-Length: 72
    	>
    	* upload completely sent off: 72 out of 72 bytes
    	< HTTP/1.1 204 No Content
    	<
    	* Connection #0 to host localhost left intact
    -----------------------------------------------------------------------------------
    [2022-06-25 13:44:57,188] INFO  <subscribe   :85> [oop-thread-1] [1000138638] SUBSCRIBE Request: from 127.0.0.1:39972, method = POST, path = /consumers/my-group/instances/my-consumer/subscription
    [2022-06-25 13:44:57,189] INFO  <idgeEndpoint:199> [oop-thread-1] Subscribe to topics [SinkTopicSubscription(topic=my-topic,partition=null,offset=null), SinkTopicSubscription(topic=my-topic,partition=null,offset=null)]
    [2022-06-25 13:44:57,189] INFO  <subscribe   :85> [oop-thread-1] [1000138638] SUBSCRIBE Response:  statusCode = 200, message = OK
    [2022-06-25 13:44:57,190] INFO  <afkaConsumer:965> [mer-thread-0] [Consumer clientId=my-consumer, groupId=my-group] Subscribed to topic(s): my-topic
    ```

# 📚 참고 자료

- 🚀**가시다님 노션**🚀
- [Strimzi Kafka Bridge Documentation (0.21.5)](https://strimzi.io/docs/bridge/latest/)
- [Using HTTP Bridge as a Kubernetes sidecar](https://strimzi.io/blog/2021/08/18/using-http-bridge-as-a-kubernetes-sidecar/)
- [Using Kubernetes Configuration Provider to load data from Secrets and Config Maps](https://strimzi.io/blog/2021/07/22/using-kubernetes-config-provider-to-load-data-from-secrets-and-config-maps/)
- [https://github.com/strimzi/kafka-env-var-config-provider](https://github.com/strimzi/kafka-env-var-config-provider)
- [https://github.com/strimzi/kafka-kubernetes-config-provider](https://github.com/strimzi/kafka-kubernetes-config-provider)
- [Configuring Strimzi](https://strimzi.io/docs/operators/latest/full/configuring.html#type-ExternalConfiguration-reference)
