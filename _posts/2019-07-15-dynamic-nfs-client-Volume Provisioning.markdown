---
layout: post
title:  "kubernetes NFS Dynamic Volume Provisioning"
date:   2019-07-15
categories: nfs volume nfsclient helm kubernetes
---


NFS client를 구성할 때 helm를 통해서 설치 하거나, 별도 provision을 통해서 설치를 할수 있습니다. 여기에서는 helm를 이용한 구성방안을 알려 드립니다.

# NFS Dynamic Volume Provisioning

## 선행 사항
* nfs server
* nfs shared volume 구성
* nfs package install on host

![flow](https://blogfiles.pstatic.net/MjAxODA5MTZfMjAz/MDAxNTM3MDczODcxODE1.91LWpZNeX46GzbjaawLGJzy3Jt50tACXg0FZmKjV6lkg.bnkBBPuBcc_myJbP7_aHFKsoFChX6Q3EpfuVkQoP4tgg.PNG.alice_k106/%EA%B7%B8%EB%A6%BC1.png?type=w2)
[참조 사이트] https://blog.naver.com/alice_k106/221360005336



## 1. helm를 통한 nfs client pod 생성

nfs client pod 및 storage class가 생성됩니다.

~~~
helm repo update

helm install -n kube-system \
  --set nfs.server=*.*.*.*  \
  --set nfs.path=[디렉토리]  \
    stable/nfs-client-provisioner --tls


kubectl get pods | grep nfs-client
worn-donkey-nfs-client-provisioner-57cb6bddb6-xv4sp               1/1     Running     0          45s

kubectl get sc | grep nfs-client
nfs-client                 cluster.local/worn-donkey-nfs-client-provisioner   1m
~~~


### 참고 사이트

[charts/stable/nfs-client-provisioner at master · helm/charts · GitHub](https://github.com/helm/charts/tree/master/stable/nfs-client-provisioner)

[openshift dynamic NFS persistent volume using NFS-client-provisioner](https://medium.com/faun/openshift-dynamic-nfs-persistent-volume-using-nfs-client-provisioner-fcbb8c9344e)

[NFS Dynamic Volume Provisioning in IBM Cloud Private](https://medium.com/@zhimin.wen/nfs-dynamic-volume-provisioning-in-ibm-cloud-private-ca44bc514cdb)

[Dynamically provision GKE storage from Cloud Filestore using the NFS-Client provisioner Google Cloud Platform Community](https://cloud.google.com/community/tutorials/gke-filestore-dynamic-provisioning)

[kubernetes dynamic provisioning ](https://kubernetes.io/blog/2016/10/dynamic-provisioning-and-storage-in-kubernetes/)
