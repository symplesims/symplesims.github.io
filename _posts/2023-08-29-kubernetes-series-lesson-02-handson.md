---
layout: post
title: "kubernetes hands on basic"

date:  2023-08-29 20:00:00 +0900
categories:
   - DevOps
   - Kubernetes
   - HandsOn
---

Kubernetes 클러스터가 어떻게 운영되는지 kubectl 명령어를 통해 직접 확인하고 애플리케이션을 배포 및 관리하는 연습을 해 봅니다. 

그 전에 왜 Kubernetes 인지 그리고 컨테이너에 대한 특징을 알아보도록 하겠습니다.

<br> 

## Why Kubernetes?

Kubernetes 는 애플리케이션 서비스 배포 및 운영 방식이 과거의 모놀리스 또는 Virtual Machine 기반과 비교해서 서비스 론칭 속도와 운영의 세련됨이 비교할 수 없을 정도로 뛰어납니다. 

<br>

### Virtual Machine(hypervisor) vs Container

IaaS 근본인 가상화 기술과 Kubernetes 의 근본인 Container 의 차이를 살펴봅시다.  


![img_14.png](/assets/images/23q3/img_14.png)

| Div.           | Virtual Machine                      | Container                          |
|----------------|--------------------------------------|------------------------------------|
| Weight         | Heavy                                | Light                              |
| Performance    | Limited                              | Native                             |
| Virtualization | Hardware virtualization              | OS virtualization (shared kernel)  |
| Start-Up       | in minutes                           | in milliseconds                    |
| Memory         | Need allocated                       | less memory space                  |
| Isolation      | OS level Fully isolated  more secure | Process level isolated less secure |

Kubernetes는 컨테이너 기술을 기반으로 하는 모든 종류의 애플리케이션을 관리하기 위한 포괄적인 솔루션을 제공하는 컨테이너 오케스트레이션 플랫폼입니다.  
애플리케이션을 자동으로 배포, 스케일링, 관리하고 무중단 서비스를 제공하는 독립적인 플랫폼 임에도 Cloud, On-Premise 심지어 Bare-Metal 에도 동일하게 Kubernetes 서비스 스택을 구성할 수 있습니다.  

- 참고로, Monolith: 방식은 대게 단일 서버로 운영되며 하드웨어 리소스, 확장, 고가용성에 대응하기 어렵습니다. 플리케이션 서버와 인프라 관리는 사람에 의해 수동으로 이루어지므로 인프라의 부하와 설정상의 오류로 장애가 발생할 수 있습니다.


<br>

## Step 1 - [Kubernetes object 이해](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/)

Kubernetes 오브젝트는 Kubernetes 리소스 디스크립터(Yaml)와 같은 Spec을 API(kubectl)를 통해 생성 / 수정 / 삭제 등 관리되는 객체 입니다.

생성된 객체는 크게 `Spec` 과 `Status` 로 구분 됩니다.

```
# Spec 명세 정보  
apiVersion: v1
kind: Pod
metadata:
spec:

# Status 상태 현황 정보  
status:
  conditions:
  containerStatuses:
  hostIP: 192.168.58.3
  phase: Running
  podIP: 10.244.1.3
  startTime: "2023-08-31T10:23:00Z"
```

Spec 은 객체 상태를 원하는 상태로 만들어야 하는 지시자 이며, Status 는 만들어진 객체의 현재 상태를 나타냅니다. 이것은 etcd 를 통해 관리 됩니다.

### KAMS

Kubernetes 오브젝트 Spec(명세)을 정의 하려면 핵심 속성인 `KAMS` 로 기억하면 좋습니다. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - image: nginxZ
    name: nginx
```

| Attribute   | Description                                                                     |
|--------------|---------------------------------------------------------------------------------|
| apiVersion   | 오브젝트를 생성하기 위해 사용하는 쿠버네티스 API 및 버전을 정의 합니다.                         |
| kind         | 오브젝트 유형을 정의 합니다.                                                          |
| metadata     | 이름 및 네임스페이스로 오브젝트의 Identity 를 부여합니다. 이 이름을 통해 오브젝트를 식별하고 참조 합니다.                |
| spec         | 원하는 오브젝트 상태를 위해 spec 을 기술 합니다. 컨테이너 이미지 및 이름, 참조 변수, 볼륨 등 많은 정보를 기술 할 수 있습니다.   |



<br>


## Step 2 - Kubernetes Context를 통한 클러스터의 연결 방법 

## Kubernetes Context
현재 운영중인 Kubernetes 클러스터와의 세션(연결 정보) 정보를 Context 환경 정보로 관리하고 있습니다.

아래 명령으로 Context 정보를 확인할 수 있으며 기본적으로 `$HOME/.kube/config` 파일로 관리 됩니다.

```
kubectl config get-contexts         # kubectl 설정에서 사용 가능한 컨텍스트 목록을 조회합니다.
kubectl config use-context basic    # 연결을 활성화 할 cluster 를 지정합니다. 
kubectl config view                 # Kubernetes 설정 정보를 조회 합니다.  
```

- Context 는 연결 가능한 Kubernetes 클러스터 정보를 담고 있습니다.

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /Users/symplesims/.minikube/ca.crt
    server: https://127.0.0.1:32784
  name: basic
contexts:
- context:
    cluster: basic
    namespace: default
    user: basic
  name: basic
current-context: basic
users:
- name: basic
  user:
    client-certificate: /Users/symplesims/.minikube/profiles/basic/client.crt
    client-key: /Users/symplesims/.minikube/profiles/basic/client.key
```


| Attribute       | Description                                                                |
|-----------------|----------------------------------------------------------------------------|
| clusters        | 연결 가능한 Kuberntes 클러스터 및 접속 정보를 정의 합니다.                                     |
| users           | Kuberntes 클러스터에 연결할 사용자(Client) 정보를 관리 합니다.                                |
| contexts        | 어떤 사용자로 어떤 Kuberntes 클러스터에 연결할 것인지 context 정보를 정의 합니다.                     |
| current-context | 현재 연결중인 context 입니다. kubectl과 같은 API 로 명령어를 실행하면 현재 세션으로 연결하여 명령어를 실행 합니다. |


<br>


## kubectl (Kubernetes API)

kubectl 클라이언트를 통해 kubernetes 클러스터 정보를 확인 하고 오브젝트 Lifecycle 을 관리합니다. 

[Kubectl autocomplete](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-autocomplete) 설정으로 명령어 자동 완성 및 축약 기능을 설정할 수 있습니다. 



### Kubernetes 운영을 위한 필수 명령어 모음

#### Cluster 관리

```
kubectl cluster-info                    # 클러스터의 정보와 상태를 확인합니다.
kubectl get nodes                       # 클러스터 내의 노드들의 상태를 확인합니다.
kubectl describe node <node-name>       # 특정 노드의 자세한 정보를 조회합니다.
kubectl get namespaces                  # 네임스페이스 목록을 조회합니다.
kubectl get events                      # 클러스터 이벤트를 확인합니다.
```

#### Pod 및 리소스 관리

```
kubectl get pods                        # 모든 Pod의 목록을 조회합니다.
kubectl describe pod <pod-name>         # 특정 Pod의 자세한 정보를 조회합니다.
kubectl logs <pod-name>                 # Pod의 로그를 조회합니다.
kubectl exec -it <pod-name> -- /bin/sh  # Pod 내부로 들어가 터미널로 작업할 수 있습니다.
kubectl delete pod <pod-name>           # Pod를 삭제합니다.
```

#### Deployment 관리

```
kubectl get deployments                                 # 모든 배포 목록을 조회합니다.
kubectl describe deployment <deployment-name>           # 배포 상세 정보를 조회합니다.
kubectl rollout status deployment/<deployment-name>     # 배포 롤아웃 상태를 확인합니다.
kubectl rollout history deployment/<deployment-name>    # 배포 롤아웃 히스토리를 조회합니다.
```

#### Service 관리
```
kubectl get services                    # 모든 서비스 목록을 조회합니다.
kubectl describe service <service-name> # 서비스의 자세한 정보를 조회합니다.

# 배포를 서비스로 노출합니다.
kubectl expose deployment <deployment-name> --type=NodePort --name=<service-name> 
```

#### Config 설정

```
kubectl config get-contexts                 # kubectl 설정에서 사용 가능한 컨텍스트 목록을 조회합니다.
kubectl config use-context <context-name>   # 활성화할 컨텍스트를 설정합니다.
```

#### 모니터링 및 디버깅

```
kubectl top pods                        # Pod의 리소스 사용량을 모니터링합니다.
kubectl describe pod <pod-name>         # Pod 문제를 진단하고 디버깅합니다.
```

#### Rolling Updates 및 롤백

```
kubectl rollout                         # 배포의 롤링 업데이트와 롤백을 관리합니다.
```

#### 포트 포워딩

```
# 로컬 포트를 통해 Pod 내부로 포트 포워딩합니다.
kubectl port-forward <pod-name> <local-port>:<remote-port>
```

#### 트러블 슈팅 명령 모음 

아래는 애플리케이션이 정상적으로 동작하지 않는 경우 문제 원인 분석을 위해 상용되는 유용한 명령어 입니다. 

```
# 특정 Pod의 상세 정보와 이벤트를 확인합니다. 이벤트에 오류 메시지가 표시될 수 있습니다.
kubectl describe pod <pod-name>

# 문제가 있는 Pod의 로그를 확인하여 어떤 오류 또는 이슈가 있는지 파악합니다.
kubectl logs -f <pod-name>

# 문제가 있는 Pod의 특정 컨테이너로 직접 들어가 터미널로 확인할 수 있습니다. (/bin/bash)
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> <container-name> -- /bin/sh # Pod 안에 여러 컨테이너가 있는 경우 

# 클러스터 이벤트 목록을 확인하여 클러스터 내에서 발생한 모든 이벤트를 조회합니다. grep 으로 문제가 되는 객체를 필터링 할 수 있습니다.
kubectl get events

# Pod의 리소스 사용량을 모니터링하여 CPU 및 메모리를 많이 사용하는 Pods 를 확인 합니다.
kubectl top pods

# 서비스 설정을 확인하여 외부로 노출되는 서비스의 포트와 IP를 확인합니다.
kubectl describe service <service-name> 

# 컨테이너가 ConfigMap 정보를 참조한다면 관련 정보가 정확하게 반영되는지 확인할 수 있습니다.  
kubectl describe configmap <configmap-name>

# 애플리케이션 배포의 롤아웃 상태를 확인하여 업데이트가 완료되었는지 확인합니다.
kubectl rollout status deployment/<deployment-name>

# 애플리케이션 배포 롤아웃 히스토리를 조회하여 롤백 옵션을 검토합니다.
kubectl rollout history deployment/<deployment-name>: 
```

<br>
<br>

## 애플리케이션 배포

얼마나 빠르게 애플리케이션이 배포될 수 있는지 경험해 봅니다. 

### Nginx 배포

- kubectl run 명령어로 배포

```shell
kubectl run nginx-first --image nginx:latest --port=80
```

- kubectl 명령어로 Spec 확인

```shell
kubectl get po nginx-first -o yaml
```


### Pod 배포 

- helloworld 애플리케이션 Pod 를 배포합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello
spec:
  containers:
    - name: hello
      image: symplesims/hello-python:latest
```

```
kubectl apply -f https://raw.githubusercontent.com/simplydemo/hello-python-flask/main/cicd/k8s/hello-po.yaml
```

- helloworld 애플리케이션 Pod 를 조회 합니다.

```
kubectl get po helloflask
kubectl get po helloflask -o wide
kubectl get po helloflask --show-labels
kubectl get po helloflask --show-labels -o wide
kubectl get po helloflask -o yaml
kubectl get pods -l type=demo
```

- 서비스 체크 

port-forward 를 통해 host Local 포트를 remote container 포트로 바인딩 할 수 있습니다. 
```
kubectl port-forward helloflask 8850:8050

 curl -i http://localhost:8850/health
```

<br>

- helloworld 애플리케이션 Pod 상세 정보 및 최근 이벤트 내역을 조회 합니다.

```
kubectl describe po helloflask
``` 

### ReplicaSet 배포  

Pods 의 기본 확장 메커니즘(Sacling)을 제공하는 하위 수준 추상화입니다. Pod 의 이상 동작에 대해 Self-Healing 처리도 담당 합니다. 하지만 새로운 애플리케이션에 대한 업데이트를 지원하지 않습니다. 

![img_17.png](/assets/images/23q3/img_17.png)

```
kubectl apply -f https://raw.githubusercontent.com/simplydemo/hello-python-flask/main/cicd/k8s/hello-rs.yaml
```

<br>

### Deployment 배포

애플리케이션의 배포와 롤백 그리고 탄력적인 확장을 컨트롤 합니다. Pods 상태와 확장은 ReplicaSet 을 활용 합니다. 

![img_18.png](/assets/images/23q3/img_18.png)


```
kubectl apply -f https://raw.githubusercontent.com/simplydemo/hello-python-flask/main/cicd/k8s/hello-deploy.yaml
```

<br>

### Deployment Controller 

아래 그림은 Kubernetes 배포 스케줄 컨트롤러에 대한 High-Level 추상화로 각각의 컨트롤러는 저 마다의 방식으로 Pods를 실행하고 상태를 관리합니다.  
Kubernetes 는 SRP(Single Responsibility Principle) 원칙을 따릅니다. 

![img_16.png](/assets/images/23q3/img_16.png)

- DaemonSet: 최초 노드가 실행될 때 자동으로 Pods 가 실행됩니다. 주로 로그 수집, 모니터링, 보안 에이전트 구성 등의 작업에 사용합니다. 
- StatefulSet: SQL Database, Kafka Stream, Message Broker 와 같이 pod 가 영속적인 볼륨 데이터를 관리하고 클러스터 서비스를 운영하는 경우 사용합니다.   

<br>


### Deployment Controller 배포 전략
Kubernetes Deployment 컨트롤러는 애플리케이션을 배포하기 위한 훌륭한 전략을 가지고 있습니다. 

<br>

#### RollingUpdate 전략 

롤링 업데이트는 현재 사용중인 Pod 를 교체하기 전에 새로운 Pod가 준비되었는지 확인하며 정해진 Replication 규칙으로 진행 합니다.    
롤링 업데이트는 `kubectl set image` 명령을 사용하여 트리거됩니다. 문제가 있는 경우 업데이트를 중지 하고 롤백할 수있습니다.  

주요 옵션은 다음과 같습니다.  
- MaxSurge:  롤아웃 중에 신규로 생성할 Pod 수를 지정합니다. Pod 갯수를 지정하거나 전체 Pod 의 백분율로 지정할 수 있습니다. (기본값은 25%) 입니다.
- MaxUnavailable: 롤아웃 중에 한번에 축소할 최대 Pod 수를 지정 합니다.

```
cat <<EOF | > nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 4
      maxUnavailable: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine3.17 # alpine3.17 | alpine3.18 
EOF

```

- nginx 애플리케이션 배포 

```shell
# 애플리케이션 배포
kubectl apply -f nginx-deploy.yaml

# 배포 상택 확인 
kubectl rollout status deployment.apps/nginx --watch

# 배포 이력 확인 
kubectl rollout history deployment.apps/nginx

# Pod 확인 
kubectl get po -l="app=nginx"
```

<br>

#### RollingUpdate Rollout 동작 확인

현재 애플리케이션 이미지가 `nginx:alpine3.17` 인데 `nginx:alpine3.18` 로 새롭게 업데이트해 보겠습니다.  

`kubectl set image deployment` 명령을 통해 새로운 이미지로 업데이트 할 수 있습니다.  

```shell
# 새로운 이미지로 교체 
kubectl set image deployment.apps/nginx nginx=nginx:alpine3.18

# 실시간 Rollout 상택 확인 
kubectl rollout status deployment.apps/nginx --watch

# 배포 이력 확인 
kubectl rollout history deployment.apps/nginx

# 변경 내역 코멘트 추가 
kubectl annotate deployment.apps/nginx kubernetes.io/change-cause="nginx 애플리케이션 버전 업 from alpine3.17 to alpine3.18" --overwrite=true
```

#### 이전 버전으로 Rollback 하기 
```
# 배포 이력 확인 
kubectl rollout history deployment.apps/nginx

# revision 1 버전으로 롤백 
kubectl rollout undo --to-revision=1 deployment.apps/nginx

# 실시간 rollout 상태 확인 
kubectl rollout status deployment.apps/nginx --watch

________________________________________
Waiting for deployment "nginx" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 6 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 7 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 7 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 7 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 7 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 8 out of 10 new replicas have been updated...
Waiting for deployment "nginx" rollout to finish: 4 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 3 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 3 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 2 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx" rollout to finish: 1 old replicas are pending termination...
deployment "nginx" successfully rolled out

# 현재 버전 코멘트 
kubectl annotate deployment.apps/nginx kubernetes.io/change-cause="nginx 애플리케이션 롤백 to alpine3.17" --overwrite=true
```


<br>

#### Recreate 전략 

현재 Replica 셋의 모든 Pods 를 한번에 교체 합니다.

```
cat <<EOF | > nginx-deploy.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine3.17 # alpine3.17 | alpine3.18
EOF

```

#### Recreate Rollout 동작 확인

```
# 새로운 이미지로 교체 
kubectl set image deployment.apps/nginx nginx=nginx:alpine3.18

# 실시간 Rollout 상택 확인 
kubectl rollout status deployment.apps/nginx --watch

________________________________________
Waiting for deployment "nginx" rollout to finish: 0 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 1 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 2 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 3 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 5 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 6 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 7 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 8 of 10 updated replicas are available...
Waiting for deployment "nginx" rollout to finish: 9 of 10 updated replicas are available...
deployment "nginx" successfully rolled out
```

- 이미지는 가능한 안정적이고 가벼운 것을 선택 합니다. Start-Up / Memory 효율이 좋은 이미지를 선택 합니다.    
- 이미지 버전은 `latest` 으로 하는 것 보다 정확한 버전을 명시하는 것이 좋습니다.   


<br>

### Ramped Slow Rollout 전략
느리지만 가장 안전하게 Pod 를 교체 하는 배포 전략 입니다.  항상 10개의 Pods가 운영되는것을 보장 하면서 한번에 하나씩 교체하는 전략 입니다.

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
#... more

```

<br>

### Blue-Green 전략

Kubernetes Service / Ingress 객체가 가리키는 타겟 객체(Service, Deployment)을 Blue 또는 Green 으로 한번에 전환할 수 있습니다.

대규모의 워크로드를 한번에 전환할 수 있으며, 특히 기존 버전(Blue)과 교체(Green)할 버전을 Rollout 시점에 유지하고 있으므로 문제가 발생하는 경우 즉시 이전 버전으로 롤백이 가능 합니다.

전환 방법은 Blue / Green 에 해당하는 객체의 Selector 를 통해 이루어 입니다.

- minikube 를 사용한다면 [Tunnel](https://minikube.sigs.k8s.io/docs/handbook/accessing/#example-of-loadbalancer) 세션을 열어서 LoadBalancer 타입을 지원합니다.

```
minikube tunnel
``` 

- `nginx` 를 사용한 Blue / Green 애플리케이션 스택을 구성하여 배포 합니다. 

```
cat <<EOF | > nginx-deploy-bg.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
  labels:
    app: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
      version: v1.0
  template:
    metadata:
      labels:
        app: nginx-app
        version: v1.0
    spec:
      initContainers:
      - name: install
        image: busybox
        command:
        - sh
        - -c
        - echo "<!DOCTYPE html><html><head></head><body><h1>Nginx App-V1</h1></body></html>" > /htdocs/index.html
        volumeMounts:
        - name: htdocs
          mountPath: "/htdocs"
      containers:
      - name: nginx
        image: nginx:alpine3.17 
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: htdocs        
      dnsPolicy: Default
      volumes:
      - name: htdocs
        emptyDir: {}

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
  labels:
    app: nginx-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-app
      version: v2.0
  template:
    metadata:
      labels:
        app: nginx-app
        version: v2.0
    spec:
      initContainers:
      - name: install
        image: busybox
        command:
        - sh
        - -c
        - echo "<!DOCTYPE html><html><head></head><body><h1>Nginx App-V2</h1></body></html>" > /htdocs/index.html
        volumeMounts:
        - name: htdocs
          mountPath: "/htdocs"
      containers:
      - name: nginx
        image: nginx:alpine3.17 
        ports:
        - containerPort: 80
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: htdocs        
      dnsPolicy: Default
      volumes:
      - name: htdocs
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: nginx-app-svc
  labels:
    app: nginx-app
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 8880
    targetPort: 80
  selector:
    app: nginx-app
    version: v1.0

EOF

```


#### Blue/Green 배포 전환 

현재 서비스 중인 상태를 확인 하고 Service 의 Selector 지시자를 통해 Blue/Green 타겟 런탐임을 선택할 수 있습니다. 

- 생성된 kubernetes 객체를 조회 하고 v1(Blue) 를 v2(Green) 으로 타겟을 변경해 보도록 하겠습니다.

```
# STEP 1 - 연관된 객체를 조회   
kubectl get all -o wide -n default

kubectl get all -l "app=nginx-app" -n default
```

- `v1` 버전 에서 `v2` 버전 으로의 전환은 현재 서비스가 가리키는 Selector(선택자)의 타겟을 확인하고 변경할 수 있습니다.   

```
# STEP 2 - 현재 서비스 expose(Endpoint) 및 대상 Selector 확인합니다.
kubectl get svc nginx-app-svc -o wide

________________________________________
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE     SELECTOR
nginx-app-svc   LoadBalancer   10.101.245.95   127.0.0.1     8880:31385/TCP   3m57s   app=nginx-app,version=v1.0
________________________________________

# `nginx-app-svc` 서비스의 Selector 선택자 속성 중 version 을 v2.0 으로 변경합니다.  
kubectl patch service/nginx-app-svc -p '{"spec":{"selector":{"version":"v2.0"}}}'
```

- `v2` 버전 에서 모든 서비스가 정상적으로 동작됨을 확인 했다면 `v1` 버전을 삭제합니다. 
```
kubectl delete deployment.apps/nginx-v1
```

<br>

### ReadinessProbe 설정 

신규로 구동된 컨테이너가 정상적으로 Start 되어서 요청 트래픽을 수신할 수 있는 상태가 되었는지를 체크 합니다.   
대게 /health 경로로 HttpStatus 200 코드를 확인 합니다.   

```
spec:
  containers:
  - name: my-app-container-name
    image: my-app-image
    readinessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      successThreshold: 1
      failureThreshold: 3
```

<br>

### LivenessProbe 설정

현재 구동중인 컨테이너가 정상적으로 동작하는지를 체크 합니다. 예를들어 listen 포트나 health 체크는 정상이지마, 실제 비즈니스 처리에서 Hang 걸려서 응답이 오지 않는 경우 등을 점검할 수 있습니다.  
대부분의 경우에서 생략할 수 있으며 필요하다고 판단되는 애플리케이션에 대하여 구성할 수 있습니다.  

```
spec:
  containers:
  - name: my-app-container-name
    image: my-app-image
    livenessProbe:
      httpGet:
        path: /health
        port: 80
      initialDelaySeconds: 180
      periodSeconds: 60
      timeoutSeconds: 60
      failureThreshold: 3
```

<br>

## Practice review 

- Pod, ReplicaSet, Deployment 를 활용 하여 Pod 를 배포 하고 상태를 조회해 봅니다.
- Pod 의 실시간 로그를 확인해 봅니다.
- Pod 의 특정 컨테이너 안으로 터미널을 통해 진입해 봅니다. 
- 클러스터의 전반적인 이벤트를 조회해 봅니다.  

<br>


## Kubectl cheatsheet

[kubectl cheatsheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/) 에 나열된 명령어로 연습해 봅니다.

- Kubernetes Object 가 어느 노드에 있는지 확인 하려면
- Kubernetes Object 자동화 운영 관리를 위한 메타 정보를 구성 하고 관련 정보를 조회 하려면  
