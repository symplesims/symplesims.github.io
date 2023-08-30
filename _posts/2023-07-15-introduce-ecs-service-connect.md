---
layout: post
title: "Introduce ECS Service Connect."
date:  2023-07-15 18:00:00 +0900
categories:
- AWS
- DevOps
- ECS
---

Service Connect 는 ECS 클러스터 내부의 분산된 서비스 간의 연결을 손쉽게 구축하고 운영할 수 있는 새로운 기능 입니다.  
애플리케이션 코드를 직접 변경하지 않으면서 서비스 간 통신을 위한 회복 탄력성(resilience)있는 네트워크 계층을 추가하여 트래픽의 헬스 체크와 요청 실패시 자동 재시도와 같은 기능을 쉽게 설정할 수 있으며 트래픽의 텔레메트리 데이터에 대한 인사이트도 Amazon CloudWatch 와 ECS 콘솔을 통해 얻을 수 있습니다.

<br>

## Business Challenge
최근 회사에서 운영중인 서비스 스택을 Bahrain 리전에서 UAE 리전 으로 마이그레이션 해야 하는 작업이 있었습니다.   
법인이 있는 국가에 속한 리전에서 안전하게 고객 데이터를 관리하고자 하는 요구사항이 있었기에 클라우드 환경이었지만 마이그레이션을 강행하였습니다.  

마이그레이션 이전을 하기전에 UAE 리전에 관한 제약 사항들을 조사하였고 주요 항목은 다음과 같습니다.    
- AWS 에서 현재까지 UAE 리전은 사용가능한 리전이 아니므로, 리전 목록에서 활성화를 해줘야 합니다.   
- Ahtena 를 정식으로 지원하지 않습니다.
- ECS Service Discovery 를 위한 CloudMap 의 Namespace 를 지원하지 않습니다.
- 그 밖에 [Service Connect 고려 사항](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html#service-connect-considerations) 을 주의깊에 살펴보아야 했습니다.


다행히 마이그레이션 기간 중에 [2023년 7월](https://aws.amazon.com/ko/about-aws/whats-new/2023/06/amazon-athena-aws-middle-east-uae-region/) 부터 Athena 서비스가 UAE 리전에서도 정식 지원이 되었습니다.  

문제는 다수의 ECS Fargate 서비스는 내부적으로 CloudMap 의 Namespace 를 사용하도록 구성되었는데 이 부분은 여전히 지원 되지 않았기 때문에 가장 적합한 대체 서비스를 찾아야 했습니다. 

조사를 통해 [2022년 AWS ReInvent](https://youtu.be/1_YUmq3MpYQ?t=945) 에서 소개된 ECS Service Connect 를 사용하여 서비스간 인터페이스를 구성하고, CloudMap 을 이용한 HPPT_NAMESPACE 방식의 Service Discovery 를 사용하는 방법을 확인할 수 있었고 이를 통해 UAE 리전에 서비스를 구성할 수 있었습니다.


<br>


### AWS ECS Service Connect 의 개요

AWS [ECS (Amazon Elastic Container Service) Service Connect](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html) 는 
Amazon ECS 서비스와 다른 AWS ECS 서비스와의 연결성을 개선하는 기능입니다. 
이 기능은 컨테이너화된 애플리케이션과 다른 AWS 리소스 간의 통신을 보다 쉽게 설정하고 관리할 수 있도록 지원합니다.


![img.png](/assets/images/23q3/img.png)


<br>

### ECS Service Connect의 주요 기능 및 장점

ECS Service Connect는 다음과 같은 주요 기능을 제공합니다:


| 기능                | 설명                                                                                                                                                                             |
|:------------------|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 서비스 간 통신 간소화    | ECS Service Connect를 사용하면 서비스 간 통신을 간소화할 수 있습니다. 기존에는 서비스 간에 통신하기 위해 수동으로 VPC 피어링, 로드 밸런서, NAT 게이트웨이 등을 설정해야 했지만, ECS Service Connect를 사용하면 보다 쉽고 간편하게 서비스 간에 통신을 구축할 수 있습니다.  |
| 서비스 간 통신 보안 강화 | ECS Service Connect는 AWS PrivateLink를 기반으로 하여 서비스 간의 통신을 보안적으로 격리할 수 있습니다. 통신은 AWS 내부 네트워크를 통해 이루어지므로 인터넷을 통과하지 않고 내부적으로만 통신하게 됩니다.                                            |
| 서비스 간 통신의 단순화  | ECS Service Connect를 사용하면 VPC 간 피어링 설정과 같은 복잡한 네트워킹 구성을 피할 수 있습니다. 이를 통해 개발자는 서비스 간 통신에 집중하고 간단한 설정으로 서비스 간 통신을 구축할 수 있습니다.                                                    |
| 뛰어난 네트워크 성능    | ECS Service Connect는 AWS PrivateLink를 사용하여 높은 네트워크 성능을 제공합니다. 이는 인터넷 대역폭에 의존하지 않고 안정적이고 빠른 서비스 간 통신을 가능하게 합니다.                                                                 |


ECS Service Connect는 컨테이너화된 애플리케이션의 네트워크 통신을 단순화하고 보안을 강화하여 AWS 리소스 간의 통신을 편리하게 관리할 수 있도록 지원합니다. 이를 통해 애플리케이션 배포 및 관리를 향상시킬 수 있습니다.

<br>

## Service Connect 프로비저닝 흐름

![img_1.png](/assets/images/23q3/img_1.png)

1. 콘솔에서 Service Connect 설정을 업데이트하면, Amazon ECS Agent를 통해 Service Connect Agent 컨테이너 설정이 주입됩니다.
2. Amazon ECS Agent는 Service Connect Agent를 생성하고, 애플리케이션 컨테이너를 배포하기 위한 명령을 내립니다.
3. ECS 노드 인스턴스에는 먼저 Service Connect Agent 가 생성되며, 추가로 필요한 컨테이너(e.g., pause 컨테이너) 등이 배포됩니다. 그리고 Service Connect 관련 환경 설정이 완료되면 애플리케이션 컨테이너가 생성됩니다.
4. 이때 Amazon ECS는 AWS Cloud Map의 HTTP_NAMESPACE 에서 미리 등록된 서비스들의 디스커버리 정보를 가져옵니다.
5. 가져온 최신 서비스 디스커버리 정보는 내부 통신을 위해 Service Connect Agent에 설정됩니다.
6. cats-api가 정상적으로 배포되면, cats-api의 Service Connect 관련 정보도 AWS Cloud Map 에 업데이트됩니다.

<br>

## Service Connect Request 흐름

![img_2.png](/assets/images/23q3/img_2.png)

위 그림은 Client - ELB - Frontend - Backkend API 서비스간 Service Connect 를 통해 Service-Mesh 를 구성하고, 그 내부에서 이루어지는 요청 흐름을 보여줍니다. 

1. 인터넷 사용자 또는 외부 클라이언트는 도메인을 통해 서비스를 액세스 하며, 도메인에 대한 타겟인 ELB 로 전달 됩니다.
2. ELB 의 listener 포트 또는 path 경로를 통해 ECS Frontend 서비스로 전달됩니다. 
3. ECS Frontend 서비스 진입에서 Service Connect Agent 는 요청 정보를 식별하고, ECS 컨테이너 내부에서 관리하는 서비스 디스커버리 정보인 Service Name 를 통해 요청을 전달합니다.
4. Frontend Service Connect Agent 는 ECS Frontend 서비스가 요청하는 Backend API 를 식별하고 Service Name 으로 ECS Backend 서비스를 연결합니다. 
5. Backend 서비스 진입에서 Service Connect Agent 는 요청 정보를 식별하고, ECS 컨테이너 내부에서 관리하는 서비스 디스커버리 정보인 Service Name 를 통해 요청을 전달합니다. 
6. Backend Service Connect Agent 는 ECS Backend 서비스의 응답 결과를 Frontend Service Connect Agent 로 전달합니다.

<br>

## ECS Service Connect 설정


![img_3.png](/assets/images/23q3/img_3.png)

Service Connect 의 네임스페이스는 HTTP_NAMESPACE 로 설정 됩니다. 이는 Service Name 을 식별할 때 DNS 네임 해석을 하지 않는다는 것입니다. 그리고 Service Discovery 항목은 설정되지 않은 것을 확인할 수 있습니다.

<br>

![img_4.png](/assets/images/23q3/img_4.png)

Service Connect 구성은 `Client side only` 와 `Client and server` 두 가지 모드가 있습니다.

- Client side only: 현재 ECS 서비스가 다른 ECS 서비스를 호출하는 경우에 선택 합니다. 
- Client and server: 다른 ECS 서비스가 현재 ECS 서비스를 호출 하는 경우에 선택 합니다. 현재 서비스를 HTTP_NAMESPACE 에 등록되며 다른 서비스는 현재서비스를 Discovery 할 수 있습니다. 

현재 ECS 서비스가 다른 서비스로부터 호출되는 서비스라면 `Client and Server` 를 선택합니다.
`Client and Server` 방식은 Service Connect Agent 가 요청을 받아들이고, 요청을 전달할 서비스를 식별하기 위해 Service Name 을 사용한다는 것을 의미합니다.

|            |                                                                                                                    | 
|------------|--------------------------------------------------------------------------------------------------------------------|
| Port alias | Service Connect Agent 가 요청을 받아들이는 포트를 의미합니다. 컨테이너의 이름 입니다.                                                         |
| Discovery  | Service Connect Agent 가 요청을 전달할 서비스를 식별하기 위해 사용하는 서비스 디스커버리 정보를 의미합니다. Service Discovery 의 Service Name 으로 매핑 됩니다. |
| DNS        | Service Connect Agent 가 요청을 전달할 서비스를 식별하기 위해 사용하는 DNS 네임을 의미합니다.                                                   |
| Port       | ECS 서비스의 서비스 Listen 포트 입니다.                                                                                      |


<br>

## ECS Service Connect 네임 디스커버리 원리 및 주의 사항 

ECS Service Connect 를 위와 같이 설정하면 CloudMap 의 HTTP_NAMESPACE 등록을 하게 됩니다. 

**1.** 최초로 `apple-svc` ECS 서비스를 Service Connect 로 등록 했을 때 HTTP_NAMESPACE 는 `apple-svc` 만 존재하게 됩니다.   
   참고로, [session-manager-plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html) 을 사용하면 `apple-svc` ECS 서비스의 컨테이너 내부로 진입할 수 있습니다.  

`apple-svc` 컨테이너의 `/etc/hosts` 파일을 열어보면 아래와 같이 보여집니다.  

- `apple-svc's` /etc/hosts 

```
127.0.0.1 localhost
10.210.51.94 ip-10-210-51-94.me-central-1.compute.internal
127.255.0.1 apple.discovery.internal.local
2600:f0f0:0:0:0:0:0:1 apple.discovery.internal.local
```

이 시점에서 `apple-svc` 는 Service Connect 를 통해서 다른 어떤 ECS 서비스를 디스커버리할 수 없으므로 통신을 할 수 없습니다. 

<br>

**2.** 두번째로  `banana-svc` ECS 서비스를 Service Connect 로 등록 하면 HTTP_NAMESPACE 는 `apple-svc` 와 `banana-svc` 가 존재합니다. 

`banana-svc` ECS 서비스의 컨테이너 내부로 진입하여 `/etc/hosts` 파일을 열어보면 아래와 같이 보여집니다. 

- `banana-svc's` /etc/hosts

```
127.0.0.1 localhost
10.210.51.94 ip-10-210-51-94.me-central-1.compute.internal
127.255.0.1 apple-svc.discovery.internal.local
2600:f0f0:0:0:0:0:0:1 apple-svc.discovery.internal.local
127.255.0.2 banana-svc.discovery.internal.local
2600:f0f0:0:0:0:0:0:2 banana-svc.discovery.internal.local
```

 `banana-svc` 서비스는 `apple-svc` 가 `/etc/hosts` 에 존재 하므로 `apple-svc` 서비스를 호출 할 수 있게 됩니다.  

문제는, `apple-svc`는 여전히 `banana-svc` 에 대한 정보가 /etc/hosts 파일에 없기때문에, `banana-svc` 서비스를 호출할 수 없습니다.  


우리는 여기에서 ECS Service Connect 가 ECS 서비스를 디스커버리하는 것이 자동으로 등록되어 참조되는 것이 아닌, HTTP_NAMESPACE 레지스트리에 등재된것만 디스커버리가 가능한 반 자동이라는 제약 사항에 주의하여 운영 해야만 하는 것 입니다.

<br>

### ECS Service Connect 운영 Tip
ECS Service Connect 를 잘 운영하려면 디스커버리 제약 사항으로 인해 ECS 서비스를 배포하는 순서가 아주 중요 합니다.   
Cache, Database, Message Queue 와 같이 다른 ECS 서비스를 호출하지 않고, 다른 ECS 서비스에 의해 호출되어지는 서비스를 우선적으로 배포하여야 합니다.  

최초에 등록한 ECS 서비스도 다른 서비스를 호출홰야하는 상황이라면, 모든 ECS 서비스를 배포한 뒤에 최초에 배포한 서비스를 다시 한번 Update 해주어 /etc/hosts 파일을 갱신하도록 할 수 있습니다.  

<br>

### 참고 사항
`/etc/hosts` 파일에 정의된 `127.255.0.0/24` 아이피 대역은 localhost 자신의 네트워크을 가리키는 LoopBack IP 주소 입니다.   
여기에 등록된 호스트 이름과 IP 를 통해 다른 ECS 서비스를 찾고 통신을 할 수 있습니다. 


<br>


## Service Connect 적용으로 강화된 모니터링 대시보드 

ECS 서비스에 Service Connect 를 구성하면 보다 강화된 모니터링 메트릭을 확인할 수 있습니다. 


- 기존 CPU / Memory 모니터링 

![img_7.png](/assets/images/23q3/img_7.png)

<br>

- 네트워크 트래픽 핼스 모니터링

![img_9.png](/assets/images/23q3/img_9.png)

<br>

- 서버사이드 응답 모니터링

![img_10.png](/assets/images/23q3/img_10.png)

<br>

## 결론

AWS ECS Service Connect를 사용하여 Bahrain 리전에서 운영되는 모든 리소스를 UAE 리전으로 안전하게 마이그레이션 하였습니다.  
특히 ECS Service Connect 는 애플리케이션 컨테이너를 위한 Envoy 로서 내부 마이크로서비스간 API 통신과 네트워크 트래픽 및 라우팅을 제어하고 네트워크 보안을 강화합니다.
결과적으로 기존의 CloudMap Discovery 기능을 Service Connect 가 완벽하게 대신해 주었기 때문에 계획된 일정과 절차대로 예외없이 안전하게 완료될 수 있었습니다.


**PS)** 마이그레이션이 완료되어 QA 검증 기간중에 UAE 리전에서도 CloudMap 을 통한 Service Discovery 기능을 지원한다는 AWS 발표가 있었습니다.  
Athena 도 그렇고 CloudMap 의 Service Discovery 기능도 그렇고 AWS 는 생각보다 빠르게 릴리즈 되므로, 가능한 예외를 만들지 않고 마이그레이션 영향도를 적게 하려면 일정을 조금 더 기다리는 것도 방법이 될 수 있을 것 같습니다. 