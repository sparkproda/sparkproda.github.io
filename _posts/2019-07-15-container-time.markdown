---
layout: post
title:  "kubernetes container time zone 변경  구성하기"
date:   2019-07-15
categories: kubernetes, container, time
---



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
