---
layout: post
title: "AWS Lambda 컨테이너 이미지 형식으로 자동화 배포"
date:  2022-07-20 18:00:00 +0900
categories: 
  - DevOps
  - AWS Lambda
---

지난 2014년 11월 AWS re:Invent 행사에서 Serverless 서비스로 Lambda 가 처음 발표 되었습니다.  
Lambda 는 서버를 프로비저닝 하지 않고도 모든 유형의 애플리케이션 또는 백엔드 서비스에 대한 코드를 실행할 수 있는 이벤트 중심의 서버리스 컴퓨팅 서비스입니다.  
출시 이후로 지금까지 Lambda 는 200 개가 넘는 AWS 서비스와 통합 할 수 있으며 다양한 목적으로 활용 되어 이제는 너무나 중요한 서비스로 자리잡고 있습니다. 

그리고 지난 2020. 12. 1. 에는 AWS Lambda 의 새로운 기능으로 컨테이너 이미지를 지원하기 시작했습니다.  
이로서 최대 10GB 크기의 컨테이너 이미지로 패키징 및 배포할 수 있게 되었고, 기계 학습 또는 대용량 데이터 처리를 위한 워크로드 유형 등 상당한 종속성이 수반되는 대규모 워크로드를 Lambda 로 쉽게 구축하고 배포할 수 있게 되었습니다.

여기서는 AWS Lambda 에 대해 알아보고 지난번 간단하게 구현한 lotto 서비스를 새롭게 지원하는 Lambda 컨테이너 이미지 타입으로 배포 하는것을 연습해 보겠습니다.  

<br>

## Serverless 컴퓨팅 소개 

시작하기에 앞서서 Serverless 에 대해 간단히 소개하고자 합니다.  

![](/assets/images/22q2/img_12.png)

위 그림의 우측 영역으로 Serverless 가 등장하기 전에는 개발팀이 구현한 애플리케이션 서비스가 제대로 동작하려면 애플리케이션을 감사꼬 있는 레거시 인프라를 필요로 했습니다.  
여기에는 OS 와 그 위에 SDK 기반의 런타임 환경 그리고 애플리케이션을 기동하고 서비스를 할 수 있도록 돕는 WAS(Web Application Server) 가 구성되어 있습니다.    
레거시 인프라에서는 개발자가 코드를 작성하면 먼저 코드를 컴파일하고 그 코드가 참조하는 라이브러리를 함께 패키징하여 WAS 에 Deploy 를 해야만 서비스가 정상 동작을 하게 됩니다.  

근데 엉뚱하고 기발했던 우리의 선배는 문득 이런 생각을 했나 봅니다. 레거시 인프라와 WAS 가 이미 준비되어 있다고 가정하면 우리는 코드만 올려서 바로 동작시킬 수 있지 않을까? 라고 말입니다.  
  
역으로 생각하면 개발자의 코드를 쏙 빼고 그 코드를 감싸고 있는 모든 레거시를 규격화하여 미리 생성해 놓는다면 개발자가 코드만 올려서 바로 실행 가능한 것이 됩니다.  
이 결론에 다다른 Guru 선배님이 몇몇 주요한 프로그램 언어를 기반으로 레거시와 런타임 환경을 선정하여 프레임워크화 하고 컴퓨팅 자원을 보다 탄력적으로 운용 가능하도록 아낌없이 기술을 쏟아 넣었습니다.  
이렇게 Serverless 컴퓨팅이 탄생 되었고 Go, Python, Ruby, Node JS, Java 및 C# 와 같은 대중적인 프로그램 언어를 기반으로 Serverless 를 지원하고 있습니다.

이제부터 개발자는 코드(Function)만 올리는 것만으로 해당 기능이 제대로 서비스가 가능해짐은 물론, 애플리케이션 서비스를 별도로 구동하거나 중지하거나 런타임 환경을 설정하거나 안정화를 위한 시스템 패치 등 일련의 모든 작업을 
할 필요가 없어지게 되었습니다.

그럼 Serverless 방식으로 코드를 구현하고 검증하고 동작시키는게 쉬울까요?  
  
제 대답은 **아니오** 입니다.  
  
왜냐하면 기존 레거시 환경에선 코드를 실행하기 위한 진입 포인트와 프로그램이 참조하는 라이브러리들 등을 개발하면서 개발자는 실시간으로 확인할 뿐만 아니라 로컬 환경에서 구현한 코드를 테스트를 실행하여 즉각적인 검증이 가능했습니다.  
하지만 Serverless 환경에선 코드를 동작하게 하는 진입(이벤트 소스)점과, 코드 실행에 필요한 라이브러리 및 컨텍스트 정보 등 필요로하는 많은 정보가 이미 제공 된다고 가정하고 코드를 구현해야 합니다.  
  
Serverless 환경으로 코드를 작성 하는건 이미 정해놓은 Input - Processing - Output 의 형식에 틀에 맞추어 제한된 범위로 구현해야하는 강력한 구속이 있습니다.  

<br>

## AWS Lambda 특징 
그럼에도 아래와 같이 Lambda 가 가지는 장점을 열거하면 사용 하지 않을 수가 없습니다.

### 고 가용성
Lambda 함수는 기본적으로 인터넷에 액세스할 수 있는 VPC 에서 실행됩니다.
Lambda 함수가 VPC 에서 실행되는 경우 해당 VPC 리전의 여러 AZ 에서 실행되는 가용성을 담당합니다.
또 다른 요점은 Lambda 컴퓨팅 용량을 가용 영역에 분산하여 데이터 센터 장애가 발생할 경우 Lambda 를 본질적으로 내결함성으로 만드는 것입니다.

* 내결함성이란 시스템의 일부 구성 요소가 작동하지 않더라도 계속 작동할 수 있는 기능을 말합니다. 애플리케이션 구성 요소의 내장된 중복 기능이라고 볼 수 있습니다.

### 비용 최적화
Lambda 함수는 컴퓨팅 처리를 위해 Memory 와 CPU 가 같이 조정됩니다. 메모리를 늘리면 CPU 할당도 함께 늘여야 합니다.  
Lambda 실행 시간을 줄이고 더 빠르게 처리하기 위해 Memory 와 CPU 를 워크로드에 맞에 늘여야 하지만 늘렸는데도 처리시간이 변화가 없다면 비용만 증가하므로 성능과 비용간의 균형이 필요하다는 것을 알 수 있습니다.
CloudWatch 로그를 통해 메모리 사용량과 실행 시간을 모니터링하고 그에 따라 Memory 와 CPU 사용량 구성을 조정하는 것을 추천합니다.

### 성능
zip 타입으로 패키징을 하게 되면 Lambda 를 처음 호출하면 S3 에서 코드를 다운로드하고, 모든 종속성을 다운로드하고, 컨테이너를 생성하고, 코드를 실행하기 전에 애플리케이션을 시작합니다.   
이 전체 기간(코드 실행 제외)을 콜드 시작 시간이라고 합니다. 컨테이너가 가동되어 실행되면 후속 Lambda 호출을 위해 Lambda 가 이미 초기화되어 있으며 애플리케이션 로직을 실행하기만 하면 되며 이 기간을 웜 스타트 시간이라고 합니다.

우리는 아래 몇가지 고려사항을 통해 콜드 시작 시간과 웜 스타트 시간을 줄여서 보다 나은 성능을 끌어낼 수 있습니다.  

1. Go, Java, C++ 에 비해 Python, Nodejs 와 같은 스크립트 기반 언어를 사용하면 콜드 스타트 시간을 줄일 수 있습니다. 
2. VPC 에 기본적으로 할당된 Private IP 대역을 사용 합니다. 별도로 ENI 를 설정하는 데 상당한 시간이 걸리고 콜드 스타트 시간이 늘어나기 때문입니다.
3. Lambda 함수를 실행하는데 필요한 코드 및 라이브러리(의존성)만 배포 합니다.
4. 전역 / 정적 변수, 싱글톤 객체를 활용 하면 컨테이너가 다운될 때까지 활성 상태로 유지됩니다. 이럴 경우 특별히 주의하여 메모리 관리 및 자원해제 기법을 필요로 합니다.
5. ConnectionPool 과 같이 전역 수준에서 재사용할 수 있도록 연결을 정의하세요.


### 보안 
Lambda 는 무엇이든 액세스할 수 있으므로 보안이 주요 고려 사항이 됩니다. 보안을 위해서는 Lambda 함수(함수 정책)를 실행시키고, 해당 Lambda 함수가 어떤 자원들에 대해 액세스 하는지를 컨트롤 해야 합니다.

- Lambda 함수에 연결된 하나의 IAM 역할: 여러 기능에 동일한 IAM 정책이 필요하더라도 하나의 IAM 역할로 Lambda 함수에 매핑되도록 함으로써 최소 권한 정책을 보장하는 데 도움이 됩니다.
- Lambda 는 공유 VPC 에서 실행되므로 AWS 자격 증명(액세스키 등)을 코드에 노출하는 것은 보안에 취약합니다.  
  : 대부분의 경우 IAM 실행 역할은 AWS SDK 를 사용하여 AWS 서비스에 연결하는 데 충분합니다.  
  : 함수가 교차 계정 서비스를 호출해야 하는 경우 자격 증명이 필요할 수 있습니다. 그런 다음 AWS Security Token Service 내에서 Assume Role API를 사용하고 임시 자격 증명을 검색하기만 하면 됩니다.  
  : 함수가 DB 자격 증명, 액세스 키와 같이 장기 자격 증명을 저장해야 하는 경우 관리 콘솔의 암호화 도우미 또는 [AWS Secrets Manager](https://aws.amazon.com/ko/secrets-manager/) 와 함께 환경 변수를 사용합니다.  


### 테스트
AWS Lambda 는 클라우드의 런타임 환경에서 실행되는 코드입니다.  
Lambda 는 직접 테스트할 엔드포인트 URL 을 제공하지도 않고 Lambda 의 호출(시작)과 Lambda 가 액세스 하는 자원을 로컬 환경에서 컨트롤 할 수 없으므로 많은 제약이 있지만 대안으로 AWS SAM 이나 LocalStack 을 활용할 수 있습니다.

1. AWS [SAM](https://docs.aws.amazon.com/ko_kr/serverless-application-model/latest/developerguide/serverless-getting-started.html) 을 사용하여 Lambda 함수를 구현 및 테스트 할 수 있습니다. 
2. [LocalStack](https://github.com/localstack/localstack) 오픈 소스 프로젝트를 사용하여 대부분의 AWS 리소스 및 서비스를 로컬 환경에 구성하여 테스트 할 수 있습니다.


### 배포
Lambda 에는 버전 관리 및 별칭(Alias) 기능을 제공하여 여러 버전의 함수를 게시 할 수 있습니다.   
기본적으로 $LATEST 버전을 사용 하지만 개발 중에 dev / prd 와 같은 환경(stack)에 대응하는 버전을 사용할 수도 있습니다.  
$LATEST 버전을 사용하는 경우 새 코드를 업로드 하면 제 기능을 올바르게 동작한다는 검증 없이 즉시 적용되기에 프로덕션 환경에서 사용하지 않는 것이 좋습니다.  
이 경우 항상 특정 버전을 가리키는 별칭(Alias)을 사용하면 코드가 변경 되고 최신 버전이 게시되더라도 영향을 받지 않고, 기능 검증을 완료한 뒤에 새로운 버전을 가리키도록 별칭을 조정 할 수 있으므로  
블루/그린 또는 카나리 배포를 계획하는 데 도움이 됩니다.

![](/assets/images/22q2/img_13.png)


### 모니터링
AWS Lambda 의 가용성 및 안정성을 향상시키는 가장 좋은 방법 중 하나는 Lambda 실행에 대한 세부 정보를 제공하는 CloudWatch 를 통합하는 것 입니다. 
Lambda 의 주요 메트릭인 요청 수, 요청당 실행 기간, 오류를 유발하는 요청 수를 자동으로 추적하고 지표를 게시합니다. 이러한 지표를 활용하여 CloudWatch 임계치(Threshold)를 지정 하여 수준별 경보를 설정할 수 있습니다.
또한 X-Ray 를 사용하여 Lambda 실행에서 잠재적인 병목 현상을 식별할 수 있습니다.  
X-Ray 는 함수의 실행 시간을 어디에 소비하는지 시각화하여 한번에 파악할 때 유용하며 전체 흐름과 연결되는 모든 다운스트림 시스템을 추적하는 데 도움이 됩니다.

<br>

## Pre-Requisite
- 인터넷 서비스를 제공하려면 DNS 서비스를 사전에 구성 해야 하는데 이 과정은 [AWS Route 53 을 통한 도메인 서비스 관리](/devops/route53/acm/hosting/2022/01/11/aws-route53.html) 를 참고 하기 바랍니다.
- 애플리케이션 서비스의 기능은 로또 645 게임에서 6개의 번호를 추천하는 아주 간단한 API 를 서비스 하는 것을 목표로 하겠습니다.  

AWS Lambda 서버리스 컴퓨팅 서비스를 프로비저닝 하기 위해 다음의 Tool 들을 로컬 PC 에 설치 및 구성 하여야 합니다.
- [Terraform 설치](https://learn.hashicorp.com/tutorials/terraform/install-cli)
- [AWS CLI 설치](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html)
- [AWS Profile 구성](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html)
- [Docker 설치](https://docs.docker.com/desktop/mac/install/)


## Lotto 서비스 아키텍처 개요
 
![](/assets/images/22q2/img_15.png)


## 주요 리소스 구성 개요
- Route 53: 인터넷 사용자가 도메인 이름을 통해 서비스에 접근 합니다.
- VPC: 컴퓨팅 리소스를 배치하는 공간으로 네트워크 구성 및 네트워크 연결 리소스로 서로 통합 되어 있습니다.
- ALB: Route 53 으로부터 유입되는 트래픽을 요청에 대응하는 Lambda 애플리케이션 서비스로 라우팅 합니다.
- ECR: lambda-rest 컨테이너 (도커) 이미지를 등록 관리하는 레지스트리 서비스로 작업 정의에서 설정 됩니다.
- Lambda: lotto 애플리케이션 서비스 입니다.
- CloudWatch LogGroup: Lambda 애플리케이션 서비스의 로그를 수집합니다.
- IAM Role: Lambda 배치 및 실행을 위한 롤 및 정책이 구성 됩니다.

<br>

## Lambda 애플리케이션 구성 정보

lotto 서비스를 하는 Lambda 함수의 구성 정보를 확인할 수 있습니다. 여기엔 람다 Lambda 함수를 트리거 하는 이벤트 소스와 실행 권한 Lambda 가 배치된 VPC 네트워크 대역 및 보안 그룹 등 주요한 정보를 확인 할 수 있습니다.  

### 함수 개요 

![](/assets/images/22q2/lambda_1.png)

이벤트 소스와 람Lambda 함수다 함수 그리고 Lambda 함수가 처리된 결과를 보내는 Target 이 있습니다.  이 예제에는 ALB 로부터 데이터가 유입되며 결과 데이터를 ALB(타겟) 로 보내게 됩니다. 
또한 사용된 코드를 확인할 수 있습니다. zip 으로 패키징한 경우는 코드가 보여지지만 image 로 패키징한 경우  

### 일반 구성 

Lambda 함수의 CPU, Memory 의 컴퓨팅 자원을 확인할 수 있습니다. Lambda 함수의 실행 제한 시간을 설정함으로써 내결함성을 높이게 됩니다. 

![](/assets/images/22q2/lambda_2.png)

### 트리거 

이벤트 소스의 상세 내역을 확인할 수 있습니다. ALB 의 host-header 정보를 확인 할 수 있습니다.  

![](/assets/images/22q2/lambda_3.png)

### 실행 권한
Lambda 함수는 VPC 네트워크의 특정 대역에 배치 되고 이벤트 소스로부터 데이터를 받고 처리 하며 그 결과를 타겟에 넘겨 주게 됩니다. 여기에 관련된 일련의 권한들을 필요로 합니다. 

- Amazon EC2 권한: VPC 의 특정 위치에 배치 하는 권한 
- Amazon CloudWatch Log: Lambda 함수 코드에 포함된 로그 출력 정보를 CloudWatch Log 그룹에 전송 하는 권한
- Amazon RDS IAM Authentication: 처리된 결과를 RDS 에 저장 하기위한 RDS 인증 권한 

![](/assets/images/22q2/lambda_4.png)

### 환경 변수 

환경 변수는 static 정보 또는 액세스 정보와 같이 코드에 노출되어선 안되는 중요한 정보를 설정 할 수 있습니다.

![](/assets/images/22q2/lambda_5.png)


### VPC 

Lambda 함수가 배치될 VPC 및 Subnet 과 보안 그룹 정보를 확인 할 수 있습니다.   

![](/assets/images/22q2/lambda_6.png)

<br><br>

## 테라폼을 통한 One-Step 프로비저닝
[aws-alb-lambda-lotto](https://github.com/chiwoo-cloud-native/aws-alb-lambda-lotto.git) 프로젝트를 통해 위의 lotto Lambda 함수를 한번에 프로비저닝 할 수 있습니다.


### Git
```
git clone https://github.com/chiwoo-cloud-native/aws-alb-lambda-lotto.git

cd aws-alb-lambda-lotto
```

### Build

terraform 명령을 통해 한번에 프로비저닝 할 수 있습니다. 

```
# 프로젝트 초기화 
terraform init

# 프로비저닝 프랜 확인 
terraform plan

# 프로비저닝 실행 
terraform apply
```

### Check
프로비저닝이 완료된 이후 `cURL` 명령을 통해 Lambda 가 정상적으로 동작하는지 확인 할 수 있습니다.

```
curl -v -X GET https://lotto.sympleops.ml/ 
...
...
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: awselb/2.0
< Date: Fri, 05 Aug 2022 01:55:40 GMT
< Content-Type: application/json
< Content-Length: 18
< Connection: keep-alive
< 
* Connection #0 to host lotto.sympleops.ml left intact
[13,28,39,38,19,8]%  
```

<br>

CloudWatch 로그를 통해 Lambda 함수의 실행시간, 처리내역, Memory 사용내역 등 주요 정보를 확인 할 수 있습니다.
```
2022-08-05T01:55:40.379000+00:00 2022/08/05/[$LATEST]5bdb128046f34f24904d0e754adf11b0 START RequestId: afeb5180-e36c-4def-a21b-e0d8ec32f4d1 Version: $LATEST
2022-08-05T01:55:40.382000+00:00 2022/08/05/[$LATEST]5bdb128046f34f24904d0e754adf11b0 2022-08-05T01:55:40.382Z	afeb5180-e36c-4def-a21b-e0d8ec32f4d1	INFOCalled lambdaHandler: lotto
2022-08-05T01:55:40.382000+00:00 2022/08/05/[$LATEST]5bdb128046f34f24904d0e754adf11b0 2022-08-05T01:55:40.382Z	afeb5180-e36c-4def-a21b-e0d8ec32f4d1	INFOresult: [ 13, 28, 39, 38, 19, 8 ]
2022-08-05T01:55:40.383000+00:00 2022/08/05/[$LATEST]5bdb128046f34f24904d0e754adf11b0 END RequestId: afeb5180-e36c-4def-a21b-e0d8ec32f4d1
2022-08-05T01:55:40.383000+00:00 2022/08/05/[$LATEST]5bdb128046f34f24904d0e754adf11b0 REPORT RequestId: afeb5180-e36c-4def-a21b-e0d8ec32f4d1	Duration: 1.73 ms	Billed Duration: 2 ms	Memory Size: 128 MB	Max Memory Used: 58 MB
```

<br><br>


다음으로 Lambda 컨테이너 Image 타입이 가지는 특징과 기존 서비리스 컴퓨팅 및 Lambda Zip 타입과의 차이점을 살펴 보도록 하겠습니다.    

## Fargate 및 EKS vs Lambda Image

Fargate 및 EKS 와 Lambda 의 이미지 타입 패키징은 동일한 컨테이너 기반의 서버리스 서비스인데 그 둘의 차이가 무엇인지 살펴 보겠습니다. 

Lambda 컨테이너는 Fargate, ECS 및 EKS 에 비해 아래의 기능들이 더 많이 향상 되었습니다. 
- Lambda 의 통합
- 응답성이 뛰어난 빠른 확장 
- 애러 처리
- 대상 연결 
- 가용성 및 내결함성을 위한 DLQ 와 queueing
- 운용 자동화를 위한 조절 및 지표

반먼, Fargate, ECS 및 EKS 는 Lambda 의 특징인 상태가 없고 수명이 짧은 약점을 극복하는 좋은 서비스 입니다.

<br>

## Lambda Image vs Zip
Lambda 컨테이너 Image 타입과 기존 Zip 타입의 차이를 살펴 보자면 아래와 같습니다.  

![](https://res.cloudinary.com/practicaldev/image/fetch/s--z54aoECE--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://dev-to-uploads.s3.amazonaws.com/i/u9dpdmn93m12es6ohy63.png)

| Lambda Layers                                    | Container Image Layers                |
|--------------------------------------------------|---------------------------------------|
| 5개로 제한                                           | 	최대 127                               |
| 명시적으로 정의됨                                        | 이미지 빌드의 일부로 암시적으로 정의됨                 |
| 단일 번호 버전 관리 (SAR 애플리케이션으로 패키징하면 의미론적 버전 관리가 허용됨) | 버전 관리 체계를 사용하여 하나 이상의 레이어를 이미지로 태그 가능 |
| Lambda 계층 리소스로 배포                              | 이미지가 푸시되면 이미지 레지스트리(ECR)에 자동으로 푸시     |

- ZIP 타입으로 패키징된 애플리케이션의 메모리는 250MB 로 제한되는 반면 컨테이너 Image 타입의 패키징은 40배가 증가된 10GB 임 
- ZIP 타입의 패키징은 배치 Data 프로세싱이나 AI/ML 모델링과 같은 대룡량 처리에 지극히 제한적인 반면 Image 타입은 대욜량 처리및 빠른 확장성을 가지는 애플리케이션 등 폭넓은 비즈니스 케이스에서 채택될 수 있음  
- Image 타입은 이벤트 핸들러를 지정 할 필요가 없음 (Dockerfile 의 CMD 로 정의)
- 동일한 애플리케이션 레이어에서 Zip 타입과 Image 타입으로 패키징만 다르게 했을 뿐인데 Zip 은 300 ms vs Image 는 5200 ms 라는 Billed Duration 의 차이가 납니다.  

**Zip 타입**
```
REPORT RequestId: 5aa36dcc-db7b-4ce6-9132-eae75a97466f 
Duration: 292.24 ms Billed Duration: 300 ms
...
Init Duration: 758.94 ms
```
<br>

**Image 타입**
```
REPORT RequestId: 679c6323-7dff-434d-9b63-d9bdb054a4ba
Duration: 502.81 ms Billed Duration: 5200 ms
...
Init Duration: 4638.39 ms
```
    
- Lambda Image 타입으로 ML 학습을 할때 /dev/shmPython 다중 처리 대기열을 사용하여 모델 실행애서 병렬 데이터 가져오기를 허용 하는 PyTorch DataSet 로더에서 문제가 발생 합니다.
  [stackoverflow issue](https://stackoverflow.com/questions/34005930/multiprocessing-semlock-is-not-implemented-when-running-on-aws-lambda) 참조 

<br><br>


## Conclusion

개발팀이 컨테이너 환경에 익숙하고 Docker 기반의 애플리케이션 런타임과 운영 모니터링 환경등 많은 영역에서 표준화 한다면 Lambda 컨테이너 이미지 타입이 큰 도움이 됩니다. 
뿐만 아니라 대용량 처리나 AI/ML 과 같은 워크로드에서도 Lambda 가 활용될 수 있습니다.  
  
반면에 개발팀이 ZIP 패키지가 더 익숙하고 컨테이너 도구를 사용할 필요가 없는 경우리면 기존과 같이 Zip 타입으로 배포 및 운영하는 것이 좋을 것 같습니다.  
  
또 한가지로 컨테이너 Image 타입의 경우 Lambda 의 런타임에 관련된 일체를 공동 책임 모델에서 AWS 가 아닌 고객이 책임지게 됩니다. 

<br>

## References
- [Serverless](https://spring.io/serverless) 소개 
- [AWS Lambda의 새로운 기능 — 컨테이너 이미지 지원](https://aws.amazon.com/ko/blogs/korea/new-for-aws-lambda-container-image-support/)
- [Container Image Support in AWS Lambda Deep Dive](https://dev.to/eoinsha/container-image-support-in-aws-lambda-deep-dive-2keh)
- [Container Image Support on AWS Lambda Bridges the Gap to Much Wider Adoption](https://www.fourtheorem.com/blog/container-image-lambda)
- [What Does Lambda's Big Memory Increase Enable?](https://www.fourtheorem.com/blog/lambda-10gb)
- [공동 책임 모델](https://aws.amazon.com/ko/compliance/shared-responsibility-model/)
