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

[AWS ECS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)  는 서비스 중심의 DevOps 문화를 기능적 관점에서 빠르게 시작할 수 있는 수단으로 훌륭한 대안입니다.  

애플리케이션을 컨테이너 환경으로 빠르게 구성함은 물론, 경량화된 워크로드로 효율적인 트래픽에 대응하는 탄력적인 확장, 운영의 자동화 등 다양한 잇점을 가져올 수 있습니다.   

DevOps 에서 중요한 것 중 하나로 고객의 피드백의 빠른 확인 인데, 결국 애플리케이션 서비스를 신속하게 출시해서 인터넷 사용자에게 서비스를 경험 하도록 열어 주는 것 입니다.    

야기서는 인터넷 사용자에게 아주 간단한 API 를 서비스를 경험 하도록 AWS Fargate 로 배포하는 연습해 보겠습니다. 

## Pre-Requisite
- 인터넷 사용자의 접근을 위해 DNS 서비스를 사전에 구성 해야 하는데 이 과정은 [AWS Route 53 을 통한 도메인 서비스 관리](https://symplesims.github.io/devops/route53/acm/hosting/2022/01/11/aws-route53.html) 를 참고 하기 바랍니다. 
- 애플리케이션 서비스의 기능은 로또 645 게임에서 6개의 번호를 추천하는 아주 간단한 API 를 서비스 하는 것을 목표로 하겠습니다.  
  참고로, DevOps 는 서비스 중심의 개발 문화이므로 Life Cycle 을 `서비스 기획` > `애플리케이션 구현` > `애플리케이션 출시 및 돌봄` 의 흐름과도 같습니다.  

<br>

## AWS ECS Fargate

DevOps 가속화하는 기술 중 하나로 [AWS ECS Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html) 는 컨테이너 기반으로 애플리케이션을 쉽게 배포하고 자동화된 관리 환경을 제공 합니다.    

<br>

## Fargate 서비스 주요 구성 단계

ECS Fargate 서비스로의 애플리케이션 배포는 크게 3 가지가 있습니다.   

![](/assets/images/22q1/aws-fargate-0001.png)

<br>

#### 1. ECS Fargate 클러스터 생성
 
![ecs-cluster](/assets/images/22q1/aws-fargate-0002.png)

<br>

#### 2. ECS 작업 정의 생성  

![ecs-dd](/assets/images/22q1/aws-fargate-0003.png)

<br>

#### 3. ECS 서비스 생성  

![ecs-ss](/assets/images/22q1/aws-fargate-0004.png)  

위 그림과 같이 AWS 관리 콘솔을 통해 AWS Fargate 를 구성 할 수 있지만, 여기서는 Terraform 을 통해 진행 하도록 하겠습니다.

먼저 애플리케이션부터 구현 하고 컨테이너로 배포할 준비를 하도록 합니다. 

<br><br>



## 서비스 제공을 위한 애플리케이션 개요   

컨테이너 환경으로 애플리케이션을 배포 하려면 동작하는 컨테이너 이미지를 빌드하여 테스트 해 보면 됩니다.  

애플리케이션 기능 요건은 1 부터 45번 까지의 숫자 중 랜덤으로 6개의 숫자를 중복되지 않게 반환하는 간단한 애플리케이션 입니다.

6 개의 로또 숫자를 추천하는 핵심 로직은 아래와 같이 간단하게 기술 할 수 있습니다.  

```kotlin
{
    val numbers = (1..45).toList()
    shuffle(numbers)
    numbers.stream().limit(6).sorted().collect(Collectors.toList())
}
```

참고로, Cloud Native Application 을 위한 애플리케이션 프레임워크로 [spring-boot](https://spring.io/projects/spring-boot) 를 선택하고 비교적 현대적인 트랜드를 쫓아서 kotlin 기반의 reactive 스타일로 구현 하도록 하겠습니다.


<br>

### 애플리케이션 빌드 
[spring-lotto-router-handler](https://github.com/chiwoo-samples/spring-lotto-router-handler.git) github 프로젝트를 checkout 하여 애플리케이션을 빌드 합니다.

사전에 [Java 11](https://www.azul.com/downloads/?package=jdk) 버전과 [Maven](https://maven.apache.org/) 이 설치 및 구성 되어 있어야 합니다.

Mac 사용자라면 [Mac OS 개발자를 위한 로컬 개발 환경 구성](https://symplesims.github.io/development/setup/macos/2021/12/02/setup-development-environment-on-macos.html) 을 참고하여 편리하게 구성 가능 합니다.  

#### Git clone 

```
git clone https://github.com/chiwoo-samples/spring-lotto-router-handler.git
```

#### Build
```
cd spring-lotto-router-handler; mvn clean package -DskipTests=true
```

#### Build Container Image 

```
docker build -t "lotto-service:1" -f ./docker/Dockerfile .
```

컨테이너 이미지를 빌드하는 `Dockerfile` 파일은 다음과 같습니다.  

```
FROM amazoncorretto:11-alpine

ENV JAVA_OPTS="-server -Xverify:none"
ENV JAVA_OPTS="$JAVA_OPTS -Dsun.misc.URLClassPath.disableJarChecking=true"

WORKDIR /app

COPY ./target/*.jar /app/springApp.jar
COPY ./docker/entrypoint.sh /app/

RUN chmod +x /app/entrypoint.sh
EXPOSE 8080
ENTRYPOINT ["/app/entrypoint.sh"]
```

OpenJDK 기반의 AWS 프로덕션용 JVM 배포판인 [amazoncorretto](https://aws.amazon.com/ko/corretto/) 를 베이스 이미지로 사용 합니다.  

#### Container Image 확인
docker images 명령어로 빌드된 컨테이너 이미지 및 버전을 확인 합니다. 

```
docker images | grep lotto-service
```

![](/assets/images/22q1/aws-fargate-0005.png)

#### Run Container
docker run 명령어로 컨테이너를 실행하고 ps 명령어로 프로세서를 확인해 봅시다.  
```
docker run -d --name lotto-service --publish "0.0.0.0:8080:8080" lotto-service:1

docker ps
CONTAINER ID   IMAGE             COMMAND                CREATED         STATUS         PORTS                    NAMES
79d6f647dae8   lotto-service:1   "/app/entrypoint.sh"   6 seconds ago   Up 7 seconds   0.0.0.0:8080->8080/tcp   lotto-service
```

#### 로컬 환경에서 기능 테스트
```
curl -v -X GET 'http://localhost:8080/api/lotto/lucky' -H 'Content-Type: application/json'
```

기능이 정상적으로 동작하는 것을 확인 하였다면 ECS Fargate 에 배포해 봅시다. 

<br><br>

## AWS Cloud 아키텍처

애플리케이션 서비스를 위한 AWS Cloud 아키텍처를 다음과 같이 설계 하였습니다.  
 
![](/assets/images/22q1/aws-fargate-1001.png)

### 주요 리소스 개요  
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

<br><br>


## Terraform 을 통한 AWS Fargate 프로비저닝 

테라폼의 주요 모듈을 이용하여 One-Step 자동화 빌드를 구현 합니다.  

이를 위해 프로그램 방식의 IAM 어카운트를 생성 하고, 관련 리소스를 생성 관리 할 수 있는 권한을 할당 하고 AccessKey 를 발급 합니다.    
발급 받은 AccessKey 는 외부에 유출되면 바로 보안 사고로 이어지며 해킹과 같은 심각한 피해를 가져올 수 있으므로 아주 아주 주의해야 합니다.  

[AWS Profile 구성 및 자격 증명 파일 설정](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html) 을 참고 하여 테라폼을 실행할 수 있도록 준비 합니다.   

테라폼 프로젝트는 [aws-fargate-magiclub](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub.git) 를 참고 합니다. 


### Checkout 

```
git clone https://github.com/chiwoo-cloud-native/aws-fargate-magiclub.git
```

## Build

aws-fargate-magiclub 프로젝트엔 vpc, alb, fargate, lotto 애플리케이션 으로 각 폴더로 구분이 되어 있습니다.  
순서대로 각 프로젝트를 한번에 프로비저닝 할 수 있습니다.  
```
cd aws-fargate-magiclub 

# aws-fargate-magiclub 폴더에서 아래 명령들을 한번에 실행하세요. 
terraform -chdir=vpc init && terraform -chdir=alb init && terraform -chdir=fargate init  && terraform -chdir=services/lotto \
terraform -chdir=vpc apply -auto-approve && \
terraform -chdir=alb apply -auto-approve && \
terraform -chdir=fargate apply -auto-approve && \
terraform -chdir=services/lotto apply -auto-approve
```
프로비저닝이 완료될 때까지 다소 시간이 걸리게 됩니다. 프로비저닝이 완료된 후에도 lotto 서비스가 running 상태로 전환되기까지 얼마간 시간이 걸립니다. 

<br>

## Test
우리가 배포한 lotto 애플리케이션 서비스가 동작하는지 cURL 명령을 통해 확인 합니다. 
```
curl --location -X GET 'http://lotto.mystarcraft.ml/api/lotto/lucky' -H 'Content-Type: application/json'
```

<br>

## 결론

과거엔 인터넷 서비스로 애플리케이션을 배포 하기 위한 주요 활동을 간단히만 정리 하더라도 수개월이 걸렸습니다. 

장비 구입을 위한 품의, 발주, 운송, IDC 와 관련된 유지 보수 업체와의 계약, 인터넷 서비스 가입, 네트워크 및 서버 설정, 애플리케이션 구현과 배포 등 수 많은 일들을 순서대로 처리하여야 왔습니다.  

지금 경험한 것과 같이 이제는 DevOps 의 가치 중심인 고객이 원하는 서비스를 빠르게 제공 하여 경험하게 하고 고객의 피드백 수렴을 통해 서비스의 가치를 높이는 활동을 할 수 있게 되었습니다.  


<br><br>

## 테라폼 코드 개요

[Terraform]() 은 디렉토리 단위로 프로비저닝을 실행 합니다. REAL 인프라스트럭처의 상태를 `terraform.tfstate` 파일로 관리 합니다.  
작성한 테라폼 코드와 terraform.tfstate 파일을 비교하여 리소스를 추가, 삭제, 갱신 작업을 통해 REAL 인프라스트럭처를 프로비저닝 및 동기화 하게 됩니다. 

<br>

### 프로바이더와 모듈 

- 프로바이더
  클라우드를 프로비저닝 하기 위해 클라우드 벤더가 제공 하고 있는 API 를 액세스하여 프로비저닝을 할 수 있도록 도와줍니다. 프로바이더를 제공 하는 주요 클라우드 벤더는 AWS, GPC, Azure 등이 있습니다. 

[providers.tf](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub/blob/main/vpc/providers.tf) 파일엔 프로바이더와 버전을 정의 하고, 특히 AWS 를 액세스할 접속 정보를 포함 합니다. 

여기서는 `active-stack` 프로파일을 통해 AWS 리소스를 액세스 하게 됩니다.

```
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 3.75.1"
    }
  }
}

provider "aws" {
  region  = "ap-northeast-2"
  profile = "active-stack" # AWS 클라우드에 액세스할 프로파일 명
}
```

- 모듈  
  복잡한 리소스 구성을 단순화 하여 편리하게 코드를 작성할 수 템플릿화 하였습니다.

모듈은 'module' 과 'source' 를 통해 정의 합니다.   
아래는 VPC 를 구성하기 위해 모듈을 정의하는 예시 입니다.      

```
module "vpc" {
  source = "registry.terraform.io/terraform-aws-modules/vpc/aws"

}
```

<br>

### VPC 구성

terraform-aws-modules 커뮤니티의 [테라폼 VPC](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest) 모듈을 사용 하여 Cloud 아키텍처 설계서의 [VPC](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/what-is-amazon-vpc.html) 를 구현 합니다.

[vpc/main.tf](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub/blob/main/vpc/main.tf) 파일의 주요 구성 내역으로 VPC 및 관련 리소스의 이름과, CIDR 블럭, AvailAbility Zone, Subnet 등을 간편하게 정의 하고 있습니다. 

```
locals {
  name_prefix = "magiclub-an2p"
}

module "vpc" {
  source = "registry.terraform.io/terraform-aws-modules/vpc/aws"

  name = local.name_prefix # VPC 및 VPC 관련 리소스의 이름
  cidr = "172.76.0.0/16"   # VPC 의 CIDR 블럭 

  azs                  = ["apne2-az1", "apne2-az3"] # 두개의 AvailAbility Zone 
  public_subnets       = ["172.76.11.0/24", "172.76.12.0/24"] # public 서브넷의 CIDR 
  public_subnet_suffix = "pub"

  private_subnets       = ["172.76.21.0/24", "172.76.22.0/24"] # Fargate 컨테이너를 위한 서브넷의 CIDR 
  private_subnet_suffix = "apps"

  enable_dns_hostnames = true

  # Fargate 컨테이너가 외부 인터넷 리소스에 액세스 하기위해선 NAT 게이트웨이를 설정 필요 
  enable_nat_gateway   = true
  single_nat_gateway   = true

  # VPC 및 관련 리소스의 태그 설정 정보
  tags = {
    Owner       = "opsmaster@your.company.com"
    Environment = "PoC"
    Team        = "DevOps"
  }

  vpc_tags         = { Name = format("%s-vpc", local.name_prefix) }
  igw_tags         = { Name = format("%s-igw", local.name_prefix) }
  nat_gateway_tags = { Name = format("%s-nat", local.name_prefix) }

}
```

<br>

### ALB 구성 

terraform-aws-modules 커뮤니티의 [테라폼 ALB](https://registry.terraform.io/modules/terraform-aws-modules/alb/aws/latest) 모듈을 사용 하여 [Application Load Balancer](https://docs.aws.amazon.com/ko_kr/elasticloadbalancing/latest/application/introduction.html) 를 구현 합니다.

[alb/main.tf](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub/blob/main/alb/main.tf) 파일의 주요 구성 내역으로 ALB 및 HTTP 와 HTTPS 리스너를 구성 하고 있습니다.

ALB 리스너 구성에서, HTTP 는 80 프로토콜로 트래픽이 유입되면 443 포트로 리-다이렉트 하도록 구성 되어 있습니다.  
인터넷 사용자의 접근을 허용하는 Load Balancer 이므로 80 포트는 안전하지 않은 통신이므로 TLS 보안 프롵토콜로 통신 하도록 리-다이렉트 하게 됩니다.  
TLS 보안 프로토콜에 사용되는 공인 인증서(ACM Certificate)를 포함하고 있습니다. 

```
locals {
  name_prefix = "magiclub-an2p"
}

module "alb" {
  source = "registry.terraform.io/terraform-aws-modules/alb/aws"

  name               = "magiclub-an2p-alb"          # 로드 밸런서 이름
  load_balancer_type = "application"                # 로드 밸런서 타입 - Application 
  vpc_id             = data.aws_vpc.this.id         # ALB 개 배치될 VPC 
  security_groups    = [aws_security_group.this.id] # ALB 의 보안 그룹 
  subnets            = data.aws_subnet_ids.pub.ids  # ALB 는 public 서브넷에 배포 하게 됩니다.  

  # HTTP 80 리스너를 정의 하며, 443 포트로 전달 합니다. 
  http_tcp_listeners = [
    {
      protocol    = "HTTP"
      port        = 80
      action_type = "redirect"
      redirect    = {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }
  ]

  # HTTPS 리스너를 정의 하며 기본은 200 OK 메시지를 반환 합니다.  
  https_listeners = [
    {
      protocol        = "HTTPS"
      port            = 443
      certificate_arn = data.aws_acm_certificate.this.arn
      action_type     = "fixed-response"
      fixed_response  = {
        content_type = "text/plain"
        message_body = "OK"
        status_code  = "200"
      }
    },
  ]

  tags = {
    Owner       = "opsmaster@your.company.com"
    Environment = "PoC"
    Team        = "DevOps"
  }

}
```

<br>

### ECS 구성

terraform-aws-modules 커뮤니티의 [테라폼 ECS](https://registry.terraform.io/modules/terraform-aws-modules/ecs/aws/latest) 모듈을 사용 하여 AWS [ECS 클러스터](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/Welcome.html)를 구현 합니다.

[fargate/main.tf](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub/blob/main/fargate/main.tf) 파일의 주요 구성 내역으로 ECS 클러스터를 구성 하고 있습니다.

```
module "ecs" {
  source  = "registry.terraform.io/terraform-aws-modules/ecs/aws"
  version = "3.5.0"

  name               = format("%s-ecs", var.name_prefix)
  container_insights = false
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  tags = merge(var.tags, {
    Name = format("%s-ecs", var.name_prefix)
  })
}
```

<br>

### 애플리케이션 Service 구성

[services/lotto/main.tf](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub/blob/main/services/lotto/main.tf) 파일의 주요 구성 내역으로 lotto 애플리케이션 서비스 구성 하고 있습니다.

애플리케이션은 frontend, backend 등 다양한 유형으로 정의 될 수 있으며 lotto 는 그 중 하나일 뿐 입니다.  

여기엔 ecs-service 라는 커스텀 모듈을 사용하여 ECS 서비스를 구성하는 task definition, ecs service, cloudwatch 로그 그룹, cloud map 등의 주료 리소스를 구성 합니다.   

```
locals {
  region             = data.aws_region.current.name
  ecr_repository_url = format("%s", aws_ecr_repository.this.repository_url)
}

module "lotto" {
  source = "../ecs-service/"

  project         = var.project
  region          = local.region
  name_prefix     = var.name_prefix
  container_name  = var.container_name
  container_port  = var.container_port
  container_image = local.ecr_repository_url
  cpu             = 512
  memory          = 1024
  desired_count   = 1
  port_mappings   = [
    {
      "protocol" : "tcp",
      "containerPort" : 8080
    },
  ]

  vpc_id                 = data.aws_vpc.this.id
  cluster_id             = data.aws_ecs_cluster.this.id
  task_role_arn          = data.aws_iam_role.ecs_task_ssm_role.arn
  execution_role_arn     = data.aws_iam_role.ecs_task_execution_role.arn
  subnets                = data.aws_subnets.apps.ids
  security_group_ids     = [aws_security_group.container_sg.id]
  target_group_arn       = aws_lb_target_group.tg8080.arn
  cloud_map_namespace_id = data.aws_service_discovery_dns_namespace.this.id

  depends_on = [aws_ecr_repository.this]
}
```

ecs-service 모듈의 [main.tf](https://github.com/chiwoo-cloud-native/aws-fargate-magiclub/blob/main/services/ecs-service/main.tf) 참조 


