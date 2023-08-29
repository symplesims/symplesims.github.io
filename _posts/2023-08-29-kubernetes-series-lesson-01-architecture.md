---
layout: post
title: "Understanding kubernetes and
 
architectural overview"
date:  2023-08-29 13:00:00 +0900
categories:
  - AWS
  - DevOps
  - Kubernetes
  - EKS

---

Kubernetes는 컨테이너화된 애플리케이션을 배포, 관리, 확장 및 운영 하기 위한 오픈소스 플랫폼입니다. 
Kubernetes 플랫폼은 애플리케이션간의 유기적인 통합과 일관된 관리 프로세스로 워크로드 크기에 상관없이 서비스 가치를 IT 로 실현해 낼 수 있는 훌륭한 도구 입니다. 

<br>

## Kubernetes UseCase 기반 특징  

1. 웹 애플리케이션 배포:  
   Kubernetes를 사용하여 회사 내부에서 사용되는 웹 애플리케이션을 안정적으로 배포하고 관리할 수 있습니다.  
   웹 애플리케이션의 컨테이너화된 버전을 배포하고 로드 밸런싱, 자동 확장 및 롤링 업데이트를 설정합니다. 
   이렇게 하면 애플리케이션의 가용성이 높아지고 개발자와 운영팀이 함께 협력하여 애플리케이션을 관리할 수 있습니다.

2. 마이크로서비스 아키텍처:  
   Kubernetes를 사용하여 마이크로 서비스 아키텍처 기반으로 워크로드르 운영 관리 할 수 있습니다.    
   각 마이크로서비스를 별도의 컨테이너로 배포하고, 서비스 디스커버리와 로드 밸런싱을 통해 서로 통신하도록 설정합니다. 각 마이크로서비스는 개별적으로 관리되므로 개발, 배포 및 유지보수가 용이해집니다.

3. DevOps 및 CI/CD 환경:  
   Kubernetes를 사용하여 개발, 테스트, 스테이징 및 프로덕션을 위한 각 환경을 노드 그룹으로 나누어 구성할 수 있습니다.  
   GitOps와 CI/CD 도구를 위한 독립적인 Toolchain 서비스를 구성하고, Toolchain 서비스는 코드 변경 사항을 자동으로 클러스터에 배포하며, 롤링 업데이트와 롤백 기능으로 안정적인 애플리케이션 배포를 하게됩니다.

4. 스케일 기능:  
   Kubernetes의 자동 스케일링 기능을 활용하여 트래픽의 증가에 대응 하는 애플리케이션 인스턴스를 자동으로 확장합니다.   
   가상화 컴퓨팅 인스턴스인 워커 노드 또한 자동으로 확장 / 축소 할 수 있습니다. 이를 통해 서비스의 가용성을 높이고 사용자의 요구에 빠르게 대응할 수 있습니다.

<br>

## Kubernetes 아키텍처 

![img_6.png](/assets/images/23q3/img_6.png)

Kubernetes 클러스터는 [컨트롤 플레인 (Control Plane)](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)과 [데이터 플레인 (Data Plane)](https://kubernetes.io/docs/concepts/overview/components/#node-components) 컴포넌트로 구성 되며, 이 두 컴포넌트는 Kubernetes 서로 상호작용하여 클러스터의 안정성과 가용성을 확보합니다.   
`컨트롤 플레인 (Control Plane)`은 클러스터의 구성 및 관리를 담당하고, `데이터 플레인 (Data Plane)`은 실제 애플리케이션을 실행하고 관리합니다. 


예를 들어 애플리케이션 Pod 를 배포하고 서비스를 Endpoint 로 expose(노출) 하는 등의 리소스(Pod, Service, ...) 상태를 조정 하는 등의 역할은 `컨트롤 플레인 (Control Plane)`이 담당 하고, 실제 애플리케이션 Pod 가 인스턴스로 올라와서 서비스 기능 자체로 동작하도록 기반을 제공하는 것은 `데이터 플레인 (Data Plane)`이 담당하게 됩니다.  

<br>

### [컨트롤 플레인 (Control Plane)](https://kubernetes.io/docs/concepts/overview/components/#control-plane-components)

- `컨트롤 플레인 (Control Plane)`은 클러스터 상태를 중앙 관리 및 제어를 통해 지속적으로 조정해 주는 컨트롤 센터 역할을 담당 합니다.   
클라이언트(사용자)가 API 및 kubectl 을 통해 Kubernetes 리소스를 생성, 수정, 삭제, 자동화 요청을 하게 되며 `컨트롤 플레인 (Control Plane)` 은 이 명령들을 처리하게 됩니다.    

#### API 서버 (API Server)  
- 클러스터의 컨트롤 플레인의 중심이 되는 구성 요소로, 클러스터 관리 작업을 수행하고 API 요청을 처리합니다.

#### 스케줄러 (Scheduler)
- 파드를 노드에 할당하는 역할을 담당합니다. 사용자가 생성한 파드를 최적의 노드에 할당하여 실행하도록 관리합니다.

#### 컨트롤러 매니저 (Controller Manager)
- 클러스터의 상태를 지속적으로 확인하고 필요에 따라 상태를 조정합니다. 레플리케이션 컨트롤러, 디플로이먼트 컨트롤러 등의 컨트롤러가 포함됩니다.

#### Etcd (Cluster Storage)
- 클러스터의 모든 구성 정보를 저장하는 분산 데이터 저장소로, 클러스터의 상태 및 설정 정보를 관리합니다.  

<br>

### [데이터 플레인 (Data Plane)](https://kubernetes.io/docs/concepts/overview/components/#node-components)

`데이터 플레인 (Data Plane)`은 모든 Node 에서 실제 컨테이너화된 애플리케이션 컨테이너가 실행중인 Pod 를 유지하고 Kubernetes 런타임 환경을 제공 합니다.    

#### kubelet 
- 각 노드에서 실행되는 에이전트로, Node 에서 PodSpec에 설명된 컨테이너가 Pod 로서 실행 중이고 정상적으로 동작하는지 확인합니다. 


#### kube-proxy:
- kube-proxy는 노드의 네트워크 규칙을 유지하며 클러스터 내부 또는 외부의 네트워크 세션을 Pod와 연결되도록 합니다.  
kube-proxy는 각 노드에서 실행되는 네트워크 프록시로 NAT 및 네트워크 연결을 담당 합니다.  
 

#### Container runtime:
- 컨테이너 런타임 (Container Runtime)은 Kubernetes 환경 내에서 컨테이너의 생성에서 제거까지 수명 주기를 관리하는 실행 주체입니다. Docker, Containerd, CRI-O 및 기타 Kubernetes CRI(컨테이너 런타임 인터페이스) 구현체가 컨테이너 런타임을 실행합니다.

<br>

### Pods
- 컨테이너화된 애플리케이션을 배포하는 가장 작은 단위 입니다. 여기에는 애플리케이션과 애플리케이션이 사용하는 공유 리소스(Volume 등)를 함께 구성하여 배포할 수 있습니다.
참고로, Pod 는 고유의 private IP 주소를 가집니다. Pod 를 구성하는 애플리케이션 서비스인 container 는 Pod 내에서 localhost 로 서로 통신 합니다.   

<br>


## Kubernetes 클라우드 아키텍처 

![img_5.png](/assets/images/23q3/img_5.png)

Kubernetes 의 기본 아키텍처를 Cloud Native한 환경에서 운영되도록 Cloud 컴포넌트를 추가하여 확장한 아키텍처 입니다. 

#### Cloud Controller Manager
- Cloud 환경에서 Node 의 자동화된 확장 정책, DNS 라우팅 정책, CSP 특화된 Ingress 컨트롤러 등을 관리 합니다.  

#### Cloud Provider API
- Public 및 Private Cloud Service Provider 를 위한 API 를 제공 합니다.

#### Cloud DNS
- Cloud 에서 제공되는 public / private 도메인 네임 해석을 Pods 및 Services 에 라우팅 되도록 지원 합니다. 

<br>

## Kubernetes 를 이용하는 최대 강점

#### 자동화된 컨테이너 관리
- Kubernetes는 컨테이너화된 애플리케이션의 배포, 확장, 관리, 복구 등을 자동화하며, 개발자와 운영팀의 작업 부담을 줄여줍니다.

#### 가용성 및 확장성
- Kubernetes는 다양한 클러스터 환경에서 애플리케이션의 가용성을 보장하고 필요에 따라 자동으로 스케일링할 수 있습니다.

#### 선언적 구성 관리
- YAML 또는 JSON 형식의 리소스 정의 파일을 사용하여 애플리케이션의 상태를 선언적으로 관리하므로, 원하는 상태에 맞게 변경할 수 있습니다.

#### 포터블한 환경
- Kubernetes는 여러 환경에서 일관된 방식으로 애플리케이션을 배포하고 관리할 수 있습니다. 로컬 개발 환경부터 클라우드까지 효율적으로 작업할 수 있습니다.

#### 설계상의 확장성
- Kubernetes는 모듈화된 아키텍처로 설계되어 확장성이 우수합니다. 필요에 따라 노드, 클러스터, 레플리카 등을 확장할 수 있습니다.

#### 자동 복구 및 롤아웃
- 애플리케이션의 장애 상황에서 자동으로 복구하고, 롤아웃 전략을 사용하여 버전 업그레이드를 안정적으로 진행할 수 있습니다.

#### 많은 커뮤니티 및 에코시스템
- Kubernetes는 큰 개발 및 사용자 커뮤니티를 가지고 있어 다양한 리소스, 도구, 플러그인, 서비스 등을 활용할 수 있는 풍부한 에코시스템을 제공합니다.

#### 클라우드 네이티브 지원
- Kubernetes는 클라우드 네이티브 워크로드와 통합하기에 이상적인 도구로서, 다양한 클라우드 프로바이더와 연동하여 사용할 수 있습니다.


<br>

## helloworld 애플리케이션 서비스 배포 

Kubernetes 환경에서 애플리케이션을 얼마나 빨리 서비스를 올리고 그 방법의 일관됨과 단순함을  
helloworld 애플리케이션 배포를 통해 `선언적 구성 배포` 와 `자동화된 컨테이너 관리`와 `가용성 및 확장성`을 바로 확인할 수 있습니다.   


- [helloworld-deploy.yaml](/assets/images/23q3/helloworld-deploy.yaml)
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
spec:
  replicas: 2
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: nginx:alpine
          ports:
            - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
   name: helloworld-svc
spec:
   selector:
      app: helloworld
   ports:
      - protocol: TCP
        port: 80
        targetPort: 80
   type: LoadBalancer
```

- 적용 
```
kubectl apply -f helloworld.yaml
```

- 확인 
```
kubectl -n default get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/helloworld-9bf945f5f-d96vq   1/1     Running   0          121m
pod/helloworld-9bf945f5f-sjmzh   1/1     Running   0          121m

NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/helloworld-svc   LoadBalancer   10.111.225.215   127.0.0.1     80:31038/TCP   121m
service/kubernetes       ClusterIP      10.96.0.1        <none>        443/TCP        10h

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helloworld   2/2     2            2           121m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/helloworld-9bf945f5f   2         2         2       121m
```

<br>

## Kubernetes 가 애플리케이션을 배포하는 흐름 

![img_7.jpg](/assets/images/23q3/img_7.jpg)

위 다이어그램은 Kubernetes 클러스터의 컴포넌트가 유기적인 상호작용으로 애플리케이션을 어떻게 배포하고 상태를 현행화하는지를 보여주고 있습니다.  

<br>

## Minikube

Kubernetes 리소스를 배포 및 테스트하는 목적으로 **`Minikube`** 를 통해 로컬 클러스터를 구성할 수 있습니다.   

[**Minikube**](https://minikube.sigs.k8s.io/docs/start/)는 로컬 개발 및 테스트를 위한 Kubernetes 클러스터를 간단하게 구성하고 관리하는 도구로, 로컬 개발 및 디버깅, 샘플 및 데모 실행, 기술 검토 및 교육, CI/CD 파이프라인 테스트 용도로 활용할 수 있습니다.

#### minikube 설치

[**Minikube Start**](https://minikube.sigs.k8s.io/docs/start/) 를 통해 OS 및 CPU 아키텍처에 해당하는 번들을 다운로드하고 설치 할 수 있습니다.

#### minikube 설정
다음은 minikube 리소스를 정의하고 실행 에뮬레이터 드라이버를 설정 합니다.
```
minikube config set driver docker
minikube config set cpus 2
minikube config set memory 5919MB
minikube config view
```

#### minikube 실행
```
minikube start
```

minikube 에서 LoadBalancer 타입을 지원하기 위해 [Tunnel](https://minikube.sigs.k8s.io/docs/handbook/accessing/#example-of-loadbalancer) 을 생성합니다. 
```
minikube tunnel
```

#### minikube 다중 노드 실행
```
minikube start --nodes 3 -p mininode
```
 
