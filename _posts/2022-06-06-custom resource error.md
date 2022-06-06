---
layout: single
title: "[Kubernetes] Custom Resource Definition Install Error"
excerpt: "Kubernetes에 Custom Resource 생성하다가 CRD의 annotation이 너무 길어서 CRD가 생성되지 않는 에러 해결"
categories:
- Trouble Shooting
tag: [Kubernetes, 쿠버네티스, DevOps, AWS, CRD, CR, Custom Resource, 커스텀 리소스, 프로메테우스, Prometheus Operator]
toc: true
author_profile: false
sidebar:
    nav: "docs"
---
  
이 포스팅은 kubernetes에 Prometheus Operator 설치하려고 하다가 생긴 트러블 슈팅에 관한 내용이다!  
  
## 1. Custom Resource 생성하기 전에 Custom Resource Definition 설치 필요🌟

```java
(🍉 |DOIK-Lab:default) root@k8s-m:~# k apply -f kube-prometheus/manifests/
	//...중략...
	resource mapping not found for name: "prometheus-adapter" namespace: "monitoring" from "kube-prometheus/manifests/prometheusAdapter-serviceMonitor.yaml": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
	ensure CRDs are installed first
	resource mapping not found for name: "prometheus-operator-rules" namespace: "monitoring" from "kube-prometheus/manifests/prometheusOperator-prometheusRule.yaml": no matches for kind "PrometheusRule" in version "monitoring.coreos.com/v1"
	ensure CRDs are installed first
	resource mapping not found for name: "prometheus-operator" namespace: "monitoring" from "kube-prometheus/manifests/prometheusOperator-serviceMonitor.yaml": no matches for kind "ServiceMonitor" in version "monitoring.coreos.com/v1"
	ensure CRDs are installed first
```

Custom Resource로 **Prometheus Operator**를 생성하기 위해 작업 진행하는데 에러 발생!

구글링해 보니 아래 링크와 같이 CRD를 먼저 설치해야 한다고 한다. 개념이 부족하니 사소한 트러블슈팅까지 하게 된다….(반성😥)  
  
*참고 링크 ) [ServiceMonitor not found in monitoring.coreos.com/v1](https://stackoverflow.com/questions/51095556/servicemonitor-not-found-in-monitoring-coreos-com-v1)*  
  
> **Custom Resource Definition**으로 객체들의 <u>Spec을 정의</u>하고 **Custom Resource**는 그 정의한 값으로 객체들의 <u>실제 상태 데이터를 조합하고 관리</u>한다. 


```java
(🍉 |DOIK-Lab:default) root@k8s-m:~/kube-prometheus/manifests# kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
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
```
  
CRD는 하나빼고 다 생성완료!  
  
하지만 아직도 사소한 에러 발생…
끝나지 않은 트러블슈팅이다 ㅎㅎ

## 2. Custom Resource Definition is invalid?!!

```java
(🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml
	The CustomResourceDefinition "prometheuses.monitoring.coreos.com" is invalid: metadata.annotations: Too long: must have at most 262144 bytes
```

이 에러 메세지를 구글링해 보니 다른 사람들도 흔히 겪는 이슈였다

이럴 땐 `apply` 대신 `replace`로 입력해 줘야 하는데 이때, `--force`를 줘서 강제로 만들어 주자!

하지만 CRD를 삭제하고 다시 생성하는 거라,  

**CRD를 삭제하면 모든 CR 인스턴스도 삭제되므로** <u>클러스터에 치명적일 수 있다는 주의 사항이 있다.</u>

*참고 링크 ) [https://github.com/argoproj/argo-cd/issues/820](https://github.com/argoproj/argo-cd/issues/820)*


```java
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

일단 성공ㅎㅎ

```java
(🍉 |DOIK-Lab:default) root@k8s-m:~# kubectl apply -f kube-prometheus/manifests/
	//...중략...
	Warning: resource clusterrolebindings/prometheus-operator is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
```

annotation이 없다는 경고 사항이 뜬다..!  
추후 문제가 생긴다면 다시 트러블슈팅해야 겠다~