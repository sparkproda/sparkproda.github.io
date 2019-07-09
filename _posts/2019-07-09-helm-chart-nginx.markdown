---
layout: post
title:  "kubernetes helm chart로 nginx 구성하기"
date:   2019-07-09
categories: kubernetes
---
# kubernetes helm chart로 nginx 구성하기

### 1. Configmap 작성

~~~
"my-nginx-configmap.yaml"


kind: ConfigMap
apiVersion: v1
metadata:
  name: my-nginx-configmap
data:
  nginx.conf: |-
    events {
      worker_connections  1024;
    }

    http {
      server {
          listen 80;
          root /usr/share/nginx/html/;
          index index.html index.htm;
          location = / {
          }
      }
    include /etc/nginx/conf.d/*.conf;
    }
~~~

### 2. helm repo 추가하기

~~~~
helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/

helm repo update

helm search ibm-charts/ibm-nginx-dev
~~~~

### 3. NFS dynamic PVC구성

~~~

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 2Gi

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-data
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 2Gi

~~~


### 4. helm value 설정하기

~~~


nginxdevelvalues.yaml

replicaCount: 2

service:
  port: 80
  externalPort: 3080
configMapName: "my-nginx-configmap"
confdPVC:
  enabled: true
  existingClaimName: "pvc-nfs"
htmlPVC:
  enabled: true
  existingClaimName: "pvc-nfs-data"  



~~~

### 5. nginx 설치

~~~
helm install  --tls -f nginxdevelvalues.yaml  --name my-web ibm-charts/ibm-nginx-dev
~~~

### 6. ingress 설치

~~~


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: 'nginx'
    nginx.org/ssl-services: 'hello-world-svc'
    ingress.kubernetes.io/ssl-redirect: 'false'
    nginx.ingress.kubernetes.io/connection-proxy-header: "keep-alive"
spec:

  rules:
  - host: abc.sample.com
    http:
      paths:
      - path: /
        backend:
          serviceName: my-web-ibm-nginx-dev
          servicePort: 3080
~~~
