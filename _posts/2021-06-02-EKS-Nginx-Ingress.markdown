---
layout: post
title:  "Using Application Load Balancer, Network Load Balancer with the Nginx ingress Controller on EKS"
date:   2021-06-02
categories:  EKS, Nginx Ingress, ALB, NLB
---

NGINX Ingress Controller를 사용해도 부하분산 및 이중화를 위해서 외부에서 오는 트래픽을 적절히 분배해 줄 외부 로드밸런서는 필요합니다
여기에서는 외부 로드발런서 ALB, NLB 구성하여 EKS 내부의 Nginx ingress와 연동하는 부분을 설명 드리겠습니다.


![스크린샷 2021-06-02 오후 1.11.55](/assets/Using-ALB-NLB-with-NginxingressController-EKS.png)

## 1. Nginx ingress controller 구성
먼저, Nginx ingress Controller를 Helm으로 설치 진행하도록 합니다.

~~~
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
~~~

~~~
 helm install sparkproda-nginx \
   --set-string controller.service.externalTrafficPolicy=Local \
   --set-string controller.service.type=NodePort \
   --set controller.publishService.enabled=true \
   --set serviceAccount.create=true \
   --set rbac.create=true \
   --set-string controller.config.server-tokens=false \
   --set-string controller.config.use-proxy-protocol=false \
   --set-string controller.config.compute-full-forwarded-for=true \
   --set-string controller.config.use-forwarded-headers=true \
   --set controller.metrics.enabled=true \
   --set controller.autoscaling.maxReplicas=3 \
   --set controller.autoscaling.minReplicas=3 \
   --set controller.autoscaling.enabled=true \
   --namespace kube-system ingress-nginx/ingress-nginx
~~~

## 2. Using Application Load Balancer, Network Load Balancer with the Nginx ingress Controller on EKS

### 2-1. Using Application LoadBalancer with the Nginx ingress Controller on EKS

#### 1) ALB 생성
Ingress 생성을 통해서 Applicaiton Load Balancer를 생성하며, 생성시  Nginx controller 서비스를 명명합니다. 그리고 healthcheck-path를 지정해야 합니다.

~~~
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: kube-system
  name: sparkproda-alb-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance
    alb.ingress.kubernetes.io/tags: Environment=dev,Team=test
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    #alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    #alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    #alb.ingress.kubernetes.io/success-codes: '200'
    #alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    #alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
spec:
  rules:
    - http:
        paths:
          - backend:
              serviceName: sparkproda-nginx-ingress-nginx-controller
              servicePort: 80
~~~
서비스명을 확인합니다.

~~~
$ kubectl get svc -n kube-system
NAME                                               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
sparkproda-nginx-ingress-nginx-controller             NodePort    10.100.95.32     <none>        80:30764/TCP,443:31474/TCP   15d
sparkproda-nginx-ingress-nginx-controller-admission   ClusterIP   10.100.188.58    <none>        443/TCP                      15d
sparkproda-nginx-ingress-nginx-controller-metrics     ClusterIP   10.100.236.138   <none>        10254/TCP                    15d
aws-load-balancer-webhook-service                  ClusterIP   10.100.196.61    <none>        443/TCP                      17d

~~~

로드발런서 이름을 확인합니다.
~~~
$ kubectl get ingress -n kube-system
NAME                     CLASS    HOSTS   ADDRESS                                                                       PORTS   AGE
sparkproda-alb-ingress   <none>   *       k8s-kubesyst-sparkproda-5ff44abcd-12345667.ap-northeast-2.elb.amazonaws.com   80      11d
~~~


#### Health check 상태확인: nginx controller pod가 1개일 때

node 2개중 1개가 unhealthy 상태입니다.
~~~
$ kubectl get pod -n kube-system
NAME                                                      READY   STATUS    RESTARTS   AGE
sparkproda-nginx-ingress-nginx-controller-5d667bfccb-x2kdk   1/1     Running   0          14d


~~~

![스크린샷 2021-06-02 오전 9.31.52](/assets/alb_healthcheck_node1.png)

#### Health check 상태확인: nginx controller pod가 3개일 때

~~~
$ kubectl get pod -n kube-system
NAME                                                      READY   STATUS    RESTARTS   AGE
sparkproda-nginx-ingress-nginx-controller-5d667bfccb-cnzbp   1/1     Running   0          3m27s
sparkproda-nginx-ingress-nginx-controller-5d667bfccb-k9gf4   1/1     Running   0          2m42s
sparkproda-nginx-ingress-nginx-controller-5d667bfccb-wjrzp   1/1     Running   0          68m
~~~

![스크린샷 2021-06-02 오전 10.56.02](/assets/alb_healthcheck_node3.png)

#### 2) ALB를 Nginx ingress와 연결

Nginx ingress 설정 시 ALB를 명명합니다.

~~~
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  namespace: sparkproda
  name: hello-kubernetes
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: k8s-kubesyst-sparkproda-5ff44abcd-12345667.ap-northeast-2.elb.amazonaws.com
    http:
      paths:
      - backend:
          serviceName: hello-kubernetes
          # connect to service using http port
          servicePort: 80
        # default path put here explicitly for illustrative purposes
        path: /  
 ~~~       



~~~
MacBook-Pro-3:TechPortal sparkproda$ kubectl get ingress -n sparkproda
NAME                CLASS    HOSTS                                                                                ADDRESS          PORTS   AGE
hello-kubernetes    <none>   k8s-kubesyst-sparkproda-5ff44abcd-12345667.ap-northeast-2.elb.amazonaws.com         10.100.250.223   80      12d
~~~

### 2-2. Using Network LoadBalancer with the Nginx ingress Controller on EKS

#### 1) NLB 생성
NLB를 생성합니다.

~~~
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: nlb-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    #service.beta.kubernetes.io/aws-load-balancer-eip-allocations: eipalloc-abcd,eipalloc-efgh,eipalloc-ijkl
spec:
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
  ports:
  - name: http
    port: 80
    protocol: TCP
~~~    

~~~
MacBook-Pro-3:eks_enviroment sparkproda$ kubectl get svc -n kube-system
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP                                                                          PORT(S)                      AGE
sparkproda-nginx-ingress-nginx-controller             NodePort       10.100.95.32     <none>                                                                               80:30764/TCP,443:31474/TCP   16d
sparkproda-nginx-ingress-nginx-controller-admission   ClusterIP      10.100.188.58    <none>                                                                               443/TCP                      16d
sparkproda-nginx-ingress-nginx-controller-metrics     ClusterIP      10.100.236.138   <none>                                                                               10254/TCP                    16d
aws-load-balancer-webhook-service                  ClusterIP      10.100.196.61    <none>                                                                               443/TCP                      17d
nlb-service                                        LoadBalancer   10.100.234.79    a65962f71234454538e6a6d056bb7ed-f23e218dfdsdfdsfdsfeee407.elb.ap-northeast-2.amazonaws.com   80:30979/TCP                 2m33s


~~~
![스크린샷 2021-06-02 오전 11.28.12](/assets/nlb-healthcheck.png)

#### 2) NLB를 Nginx ingress와 연결

~~~
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  namespace: sparkproda
  name: hello-kubernetes
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: a65962f71234454538e6a6d056bb7ed-f23e218dfdsdfdsfdsfeee407.elb.ap-northeast-2.amazonaws.com
    http:
      paths:
      - backend:
          serviceName: hello-kubernetes
          # connect to service using http port
          servicePort: 80
        # default path put here explicitly for illustrative purposes
        path: /  
 ~~~       
