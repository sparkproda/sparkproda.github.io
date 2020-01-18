---
layout: post
title:  "pod status updated "
date:   2019-09-02
categories: kubernetes pod update time
---

Pod 정보 업데이트 시간을 단축 하기 위한 옵션 설정값이 kubernetes 1.13에서 아래 부분으로 변경되었습니다.

기존 pod-eviction-timeout 옵션에서 아래 부분으로 변경되었습니다.

- default-not-ready-toleration-seconds
- default-unreachable-toleration-seconds

# Kubernetes node 상태 업데이트 단축방안

~~~
services:
  kube-controller:
    extra_args:
      node-monitor-period: Xs
      node-monitor-grace-period: Xs
  kubelet:
      node-status-update-frequency: Xs
  kube-api:
    extra_args:
      default-not-ready-toleration-seconds: X
      default-unreachable-toleration-seconds: X
~~~

무엇보다 적용하기 위해서는 PoD를 재시작하여 Pod tolerations 정보가 업데이트 되어 있어야 합니다.    

~~~
tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 30
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 30
~~~    
