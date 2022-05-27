---
layout: post
title: "Automation Building AWS Fargate & Deploy application"
date:  2022-01-15 11:00:00 +0900
categories: 
  - DevOps
  - AWS Fargate
  - Terraform
  - Automation
---

Automation Building AWS Fargate & Deploy application  
---------

DevOps 의 중심에는 언제나 서비스가 자리잡고 서비스는 사용자 원하는 기능을 구현햔 애플리케이션이 있습니다.  
여기서는 로또 645 번호를 추천하는 아주 단순한 백엔드 애플리케이션(API)을 ECS Fargate 로 배포하는 과정을 진행 합니다.  
중요한건 애플리케이션 서비스를 신속하게 출시하여 인터넷을 이용하는 사용자에게 서비스를 경험 하게 한다는 것 입니다.  

<br>

## AWS ECS Fargate

DevOps 가속화하는 기술 중 하나로 [AWS ECS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html) 는 컨테이너 기반으로 애플리케이션을 쉽게 배포하고 자동화된 관리 환경을 제공 합니다.    

<br>

## Fargate 서비스 주요 구성 단계

ECS Fargate 서비스로의 애플리케이션 배포는 크게 3 영역으로 구분 됩니다. 

![](/assets/images/22q1/aws-fargate-0001.png)

<br>

1. ECS Fargate 클러스터 생성
 
![ecs-cluster](/assets/images/22q1/aws-fargate-0002.png)

<br>

2. ECS 작업 정의 생성  

![ecs-dd](/assets/images/22q1/aws-fargate-0003.png)

<br>

4. ECS 서비스 생성  

![ecs-ss](/assets/images/22q1/aws-fargate-0004.png)  

<br>

물론 그림과 같이 AWS 관리 콘솔에서 리소스를 구성 하려면 상당히 많은 양의 정보를 세심하게 서로 연관하여 설정 해야 합니다.  
구성 과정에서 실수가 있을 수 있고 잘못된 구성이 있는 경우 그것을 식별하고 교정하기 위해 많은 시간이 걸릴 수 있습니다.  

<br>

## 서비스 제공을 위한 애플리케이션 개요   
먼저 로또 645 번호를 추천 하는 Backend 애플리케이션을 구현 합니다.  
규칙은 1 부터 45번 까지의 숫자 중 랜덤으로 6개의 숫자를 중복되지 않게 반환하는 간단한 애플리케이션 입니다. 

Cloud Native Application 을 위한 애플리케이션 프레임워크로 [spring-boot](https://spring.io/projects/spring-boot) 를 선택하고 비교적 현대적인 트랜드를 쫓아서 kotlin 기반의 reactive 스타일로 구현 하도록 하겠습니다.

6 개의 로또 숫자를 추천하는 핵심 로직은 아래와 같이 간단하게 기술할 수 있습니다. 
```kotlin
{
    val numbers = (1..45).toList()
    shuffle(numbers)
    numbers.stream().limit(6).sorted().collect(Collectors.toList())
}
```

<br>

### 애플리케이션 빌드 
[spring-lotto-router-handler](https://github.com/chiwoo-samples/spring-lotto-router-handler.git) github 프로젝트를 checkout 하여 애플리케이션을 빌드 합니다.

- git clone & packing 
```
git clone https://github.com/chiwoo-samples/spring-lotto-router-handler.git

cd spring-lotto-router-handler
mvn clean package -DskipTests=true
```

- container(도커) 이미지 빌드 
```
docker build -t "symplesims/lotto-service:0.0.1" -f ./docker/Dockerfile .
```

- 로컬 환경에서 서비스 구동 및 기능 검증
```
docker run -d --name lotto-service --publish "0.0.0.0:8080:8080" symplesims/lotto-service:0.0.1
```

- 로컬 환경에서 기능 테스트
```
curl --location -X GET 'http://localhost:8080/api/lotto/lucky' -H 'Content-Type: application/json'
```

기능이 정상적으로 동작하는 것을 확인 하였다면 본격적으로 ECS Fargate 로 서비스를 합니다. 

<br>

## Cloud Architecture for Application

먼저 로또 번호를 추천하는 애플리케이션 서비스를 위한 클라우드 아키텍처를 설계 합니다.  
 
![](/assets/images/22q1/aws-fargate-1001.png)

### 주요 리소스 설명 
- Route 53: 인터넷 사용자가 도메인 이름을 통해 서비스에 접근 합니다. 
- VPC: 컴퓨팅 리소스를 배치하는 공간으로 네트워크 구성 및 네트워크 연결 리소스로 서로 통합 되어 있습니다.  
- ALB: Route 53 으로부터 유입되는 트래픽을 요청에 대응하는 애플리케이션 서비스로 라우팅 합니다.
- ECS Fargate: 클러스터, 작업 정의, 서비스로 생성된 컨테이너 기반 애플리케이션 서비스를 제공 하는 컨테이너 서비스 입니다.  
- CloudWatch: ECS 애플리케이션 서비스의 로그를 수집 관리하는 로거 드라이브로 작업 정의를 통해 구성 합니다. 
- IAM Role: 태스크를 정의하는 Role과 ECS 서비스를 실행하는 Role 을 작업 정의에서 설정 됩니다. 
- ECR: 컨테이너 (도커) 이미지를 등록 관리하는 레지스트리 서비스로 작업 정의에서 설정 됩니다.  
- Cloud Map: 컨테이너 애플리케이션을 위한 디스커버리 서비스로 Route 53 의 호스팅 정보가 사전에 구성되어 있어야 합니다.

* [drawio-desktop](https://github.com/jgraph/drawio-desktop/releases/tag/v18.0.6) 툴을 사용하면 좀 더 편라하게 아키텍처를 설계 할 수 있습니다.  

<br>

### 서비스 플랫폼을 위한 애플리케이션 주요 속성

애플리케이션 서비스 플랫폼(Project)의 이름을 `magiclub` 으로 하고, 네이밍 및 태깅 규칙 등 일관된 정책을 적용하기 위해 아래와 같이 주요 속성을 정의 합니다.   
```
Project : magiclub
Region  : ap-northeast-2
Environment: PoC
Team    : DevOps
Owner   : opsmaster@your.company.com
Apps    : lotto 
```

위 정보를 기반으로 [Terraform](https://www.terraform.io/) 을 통해 자동화된 방식으로 프로비저닝 하도록 합니다. 

<br>


## AWS Fargate 프로비저닝 

테라폼의 주요 모듈을 이용하여 One-Step 자동화 빌드를 구현 하고자 합니다. 

이를 위해 프로그램 방식의 IAM 어카운트를 생성 하고, 관련 리소스를 생성 관리 할 수 있는 충분한 권한을 할당 하고 AccessKey 를 발급 하되,   
AccessKey 는 외부에 유출되게 되면 바로 보안 사고로 이어지며 심각한 해킹 피해를 가져올 수 있으니 특별히 주의하여 관리 하기를 당부 합니다.   

### Checkout 

```
git clone https://github.com/chiwoo-cloud-native/aws-fargate-magiclub.git
```

### Build
```
terraform init

terraform plan

terraform apply
```

