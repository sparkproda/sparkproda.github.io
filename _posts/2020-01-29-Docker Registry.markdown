---
layout: post
title:  "도커로 Docker Registry 구성하기"
date:   2020-01-29
categories: Docker local registry docker login https local registry htpasswd
---

도커레지스트 docker 이미지 기반으로 도커 레지스트리를 구성합니다.
사용자 인증서를 사전에 생성해야 하며, 이를 통해서 https 통신을  할 수 있게 구성할 수 있습니다. 또한 self-signed 인증서 같은 경우 rootCA를 Trust CA로 등록 하고 docker를 재기동 해야 합니다.

# 01. docker login을 위한 패스워드 파일 생성

사용자명을 ose4로 생성합니다.

~~~
$ htpasswd -Bc htpasswd ose4

# 파일 내용을 보면 아래와 같습니다.

ose4:$apr1$h1Tl.Fu*****u$pFWzOZCYV0b13L1
~~~

# 02. docker 이미지를 저장할 디렉토리를 생성합니다.

~~~

$ mkdir -p /data

~~~

# 03. docker registry 이미지를 사용하여 실행

인증서 경로, 패스워드 경로, 사용할 포트를 잘 확인하셔서 구성하시면 됩니다.

~~~
$ sudo docker run --restart=always --name mirror-registry -p 5001:5000 -v /data:/data -v /auth:/auth  -e "REGISTRY_AUTH=htpasswd"  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm"  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd  -e REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/data -v /etc/pki/tls/private/:/certs -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/docker-cert.pem  -e REGISTRY_HTTP_TLS_KEY=/certs/docker-key.pem -d docker.io/library/registry:2

~~~

# 04. docker login 확인하기

아래 명령어를 통해서 repository가 조회 되는지 확인해 봅니다.

~~~
$ curl -u <user_name>:<password> -k https://<local_registry_host_name>:<local_registry_host_port>/v2/_catalog

{"repositories":[]}

~~~
