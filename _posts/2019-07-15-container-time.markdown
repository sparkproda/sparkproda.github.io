---
layout: post
title:  "kubernetes container time zone 변경  구성하기"
date:   2019-07-15
categories: kubernetes container timezone 컨테이너 타임존
---


컨테이너의 시간을 변경하기 위해서는 hostpath로 volume mount해서 변경 할 수 있습니다.
### 1. timezone 변경


~~~
$ kubectl edit deploy nginx
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  ...
  template:
    ...
    spec:
      containers:
      ...
        volumeMounts:
        - mountPath: /etc/localtime
          name: timezone-config
      volumes:
      - hostPath:
          path: /usr/share/zoneinfo/Asia/Seoul
        name: timezone-config

~~~
