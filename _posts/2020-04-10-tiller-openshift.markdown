---
layout: post
title:  "Helm tiller install  On openshift  "
date:   2020-04-10
categories:  helm tiller openshift install
---
Helm tiller를 오픈쉬프트4에 설치합니다.


# Helm tiller install On openshift

Helm tiller를 오픈쉬프트4에 설치합니다.


## 1. Helm을 다운로드 받습니다.
~~~
mkdir -p /tiller

$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.9.0-linux-amd64.tar.gz | tar xz

cp linux-amd64/helm /usr/bin/

~~~

## 2. tiller를 설치합니다.

~~~
$ oc new-project tiller

$ export TILLER_NAMESPACE=tiller

$ wget -q https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml

$ oc process -p TILLER_NAMESPACE=$TILLER_NAMESPACE \
    -p HELM_VERSION=v2.9.1 -f tiller-template.yaml > tiller.yaml

$ oc create -f tiller.yaml

$ oc policy add-role-to-user edit \
"system:serviceaccount:${TILLER_NAMESPACE}:tiller"


오류발생시
이부분을 해야 tls 오류 해결됨
$ export TILLER_NAMESPACE=tiller

$ oc policy add-role-to-user edit \
"system:serviceaccount:${TILLER_NAMESPACE}:tiller"

~~~

## 3. Helm repo update

~~~
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=tiller:tiller

helm init --client-only
helm repo update
~~~


## 4. Helm install test

~~~
oc new-project newapp

oc adm policy add-scc-to-user anyuid system:serviceaccount:newapp:redis-master-redis-ha -n newapp

helm install --set persistentVolume.storageClass=managed-nfs-storage --name redis-master stable/redis-ha -n newapp

oc adm policy add-scc-to-user anyuid system:serviceaccount:newapp:redis-master-redis-ha -n newapp

~~~
