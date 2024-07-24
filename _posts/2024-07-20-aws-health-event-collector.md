---
layout: post
title: "Integrate AWS Health Event and check major events through real-time channels"

date:  2024-07-20 18:30:00 +0900
categories:
   - DevOps
   - PEOps
---


AWS Health Event 통합하고 실시간 채널로 주요 이벤트 확인 하기



## AWS Health Event Collector 구현의 배경

회사에서 운영하는 Cloud 규모가 커지면서 다양한 아키텍처, 그리고 많은 리소스를 운영 관리해야 합니다. 리소스는 언제나 상태가 변하기 마련입니다. AWS 메인트넌스, 개발팀의 애플리케이션 변화, 뜻하지 않은 장애 등 다양한 상황에 놓이게 됩니다.    

AWS Health 이벤트를 통합하고 실시간 알림을 받음으로써 이 문제를 어느 정도 완하할 수 있습니다. 


다음은 AWS Health 이벤트를 통합하고 실시간 알림을 받는 주요 이유입니다.

- **신속한 문제 대응**: AWS Health는 AWS 서비스의 상태 변화와 관련된 이벤트를 실시간으로 제공합니다. 문제 발생 시 즉각적인 알림을 통해 빠르게 대응할 수 있습니다.

- **서비스 가용성 유지**: 중요한 서비스 장애나 유지보수 이벤트에 대한 알림을 받아 서비스 가용성을 최대한 유지하고, 비즈니스 연속성을 확보할 수 있습니다.

- **비용 절감**: 예기치 않은 장애나 성능 저하로 인한 손실을 최소화하여 비용을 절감할 수 있습니다. 빠른 대응으로 문제를 조기에 해결하면 추가적인 비용 발생을 방지할 수 있습니다.

- **보안**: 보안 이벤트와 관련된 알림을 실시간으로 받아 보안 문제에 신속히 대응하고, 데이터와 인프라를 보호할 수 있습니다.

- **서비스 운영 신뢰 유지**: 서비스의 신뢰성과 안정성을 유지함으로써 조직의 신뢰를 유지하고, 사용자 경험을 향상시킬 수 있습니다.



## 아키텍처

아키텍처 다이어그램을 보듯이 Lambda 를 제외하고 SaaS 서비스로 통합되어 집니다. 특히 Organizations 에 가입된 각각의 맴버용 서비스 계정을 대상으로 일관된 방식으로 Health 이벤트를 통합하여 수집하는 점이 장점입니다. 

![img_17.png](/assets%2Fimages%2F24q3%2Fimg_17.png)

<br>

## 주요 컴포넌트 

- **Data Collector**: Organizations 에 가입된 AWS 계정을 대상으로 Health 이벤트를 수집합니다.
- **Lambda Consumer**: `Data Collector` Event Bus 로부터 전달받은 Health 이벤트를 Google Hangout 채널로 실시간 전송 합니다.
- **Event Forwarder**: Organizations 에 가입된 AWS 계정이 Health 이벤트를 전송합니다.

<br>

## 구현 전략

서비스를 이루는 컴포넌트는 `Data Collector` 와 `Event Forwarder` 입니다. `Data Collector`가 이벤트를 수신할 모든 준비가 되어 있다면, 동일한 규격의 다수의 `Event Forwarder` 요청을 처리할 수 있습니다.

이런 이유로 CloudFormation 통해 구현 하였습니다. 
- CloudFormation 템플릿을 통해 이벤트를 통합 수집하고 노티 할 AWS 계정 및 리전에 `Data Collector` 를 프로비저닝  
- CloudFormation Stack-Sets를 통해 다수의 `Event Forwarder`를 자동화된 방식으로 프로비저닝 


CloudFormation 템플릿은 `자동화된 배포`, `일관성`, `버전관리`, `확장성`, `재사용성`, `보안 및 감사` 등 주요한 장점이 있습니다.

<br>

## Data Collector 컴포넌트  

![img_12.png](/assets%2Fimages%2F24q3%2Fimg_12.png)

CloudFormation 다이어그램에서 보듯이 생각 보다 많은 리소스가 통합됩니다. 특히 고려한 부분은 KMS 를 적용한 `데이터 보안`과 `리소스 상호간의 IAM 정책의 구성`입니다.   

- [aws-cf-template-health-collector-v1.0.yaml](https://raw.githubusercontent.com/simplydemo/aws-health-collector/main/cf-stacks/aws-cf-template-health-collector-v1.0.yaml) 템플릿 소스 코드  

<br>

## Event Forwarder 컴포넌트

CloudFormation 다이어그램에서 보듯이 AWS Health 이벤트만 수집하여 포워딩하므로 아주 단촐합니다. 이 컴포넌트는 Organizations 에서 다양한 OU, Account, Region 등 운영자가 프로비저닝을 원하는 어떤 형태로든 프로비저닝을 할 수 있습니다.


![img_18.png](/assets%2Fimages%2F24q3%2Fimg_18.png)


<br>

## 배포 순서

CloudFormation 템플릿을 통해 완전 자동화 하려면 먼저 커스텀 Notification 을 담당하는 Lambda 이미지를 ECR 에 업로드 해 두어야 합니다.

1. `SNS Subscriber` 역할로 실시간 통보를 담당하는 [aws-health-delibird](https://hub.docker.com/r/symplesims/aws-health-delibird) 컨테이너 이미지로 ECR 저장소에 업로드 합니다.

2. `Data Collector 스택`을 [aws-cf-template-health-collector-v1.0.yaml](https://raw.githubusercontent.com/simplydemo/aws-health-collector/main/cf-stacks/aws-cf-template-health-collector-v1.0.yaml) 템플릿 으로 배포합니다.

3. `Event Forwarder 스택`을 [aws-cf-template-health-sender-orgs-v1.0.yaml](https://raw.githubusercontent.com/simplydemo/aws-health-collector/main/cf-stacks/aws-cf-template-health-sender-orgs-v1.0.yaml) Stack-Sets 으로 배포합니다.


<br>

## 배포 절차 리뷰

**1.** `Data Collector 스택`을 구성하기 위한 ECR 저장소와 이미지를 업로드 합니다.

[aws-health-ecr.sh](https://raw.githubusercontent.com/simplydemo/aws-health-collector/main/shells/aws-health-ecr.sh) 쉘 파일을 이용하여,
[symplesims/aws-health-delibird:1.0.0](https://hub.docker.com/r/symplesims/aws-health-delibird) 도커 이미지를 로컬에 내려받고 ECR 저장소를 생성하고 업로드를 합니다. 

정상적으로 진행 되려면 로컬 PC 에 `AWC CLI`와 `docker` 명령을 실행할 수 있어야 합니다. 

```shell
docker pull symplesims/aws-health-delibird:1.0.0

LAMBDA_IMAGE="symplesims/aws-health-delibird:1.0.0"


# 아래는 Data Collector 스택이 구성 될 AWS_REGION 및 ECR 저장소 이름입니다. KMS_ALIAS_NAME 는 AWS 관리형 키로 이 값은 변경하지 말아주세요.
REGION="ap-northeast-2"
ECR_NAME="cops-health-delibird-lambda-ecr"
ECR_TAG="1.0.0"
KMS_ALIAS_NAME="aws/ecr"
```

**2.** `Data Collector 스택`을 배포 합니다.

CloudFormation > Stacks - `Create stack`

![img_11.png](/assets%2Fimages%2F24q3%2Fimg_11.png)

<br>

![img_13.png](/assets%2Fimages%2F24q3%2Fimg_13.png)

[aws-cf-template-health-collector-v1.0.yaml](https://raw.githubusercontent.com/simplydemo/aws-health-collector/main/cf-stacks/aws-cf-template-health-collector-v1.0.yaml) 템플릿을 CF 스택을 실행할 수 있는 S3 버킷에 업로드 합니다.

`Amazon S3 URL` 속성에 해당 S3 객체의 URL을 입력하고 다음 단계로 이동 합니다.

- 형식: "https://<your-template-s3-bucket>.s3.<your-region>.amazonaws.com/aws-cf-template-health-collector-v1.0.yaml"


<br>

![img_14.png](/assets%2Fimages%2F24q3%2Fimg_14.png)

- 주요 사용자 정의 파라미터 입력

```
Project         : 프로젝트 코드입니다. 리소스 네임 및 태깅 속성을 통한 일관성을 유지하기 위한 코드입니다.
Region          : 스택이 배포될 리전 입니다.  
ECRImageUri     : 111122223333.dkr.ecr.<your-region>.amazonaws.com/cops-health-delibird-lambda-ecr:1.0.0
GchatWebhookUrl : 메시지를 전송 할 채널로 Google Hangout WebHooK 주소입니다.
OrgId           : Organizations 조직 아이디 입니다. OrgId 에서오는 모든 AWS Health 이벤트를 수신받기 위함입니다.  
```

<br>

![img_15.png](/assets%2Fimages%2F24q3%2Fimg_15.png)

입력 항목들을 리뷰하고 스택을 생성 합니다. 

<br>

![img_16.png](/assets%2Fimages%2F24q3%2Fimg_16.png)

스택이 생성되는 과정 입니다. 




**3.** `Event Forwarder 스택`을 배포 합니다.

- `CloudFormation > StackSets` - `Create StackSet` 으로 진행됩니다.

반드시 **조직 마스터 계정** 에서 프로비저닝 해야 합니다. 

StackSets 는 리전을 선택하여 프로비저닝 하게 되므로 가급적 `us-east-1` 버지니아 리전에서 관리하는걸 추천 합니다.


![img.png](/assets%2Fimages%2F24q3%2Fimg.png)


<br>


![img_1.png](/assets%2Fimages%2F24q3%2Fimg_1.png)

[aws-cf-template-health-sender-orgs-v1.0.yaml](https://raw.githubusercontent.com/simplydemo/aws-health-collector/main/cf-stacks/aws-cf-template-health-sender-orgs-v1.0.yaml) 템플릿을 CF 스택을 실행할 수 있는 S3 버킷에 업로드 합니다.

`Amazon S3 URL` 속성에 해당 S3 객체의 URL을 입력하고 다음 단계로 이동 합니다.

- 형식: "https://<your-template-s3-bucket>.s3.<your-region>.amazonaws.com/aws-cf-template-health-sender-orgs-v1.0.yaml"


<br>


![img_2.png](/assets%2Fimages%2F24q3%2Fimg_2.png)

- 주요 사용자 정의 파라미터를 입력 합니다.

```
Project              : 프로젝트 코드입니다. 리소스 네임 및 태깅 속성을 통한 일관성을 유지하기 위한 코드입니다.
CollectorEventBusArn : 앞서 프로비저닝한 Data Collector의 Event Bus ARN 을 입력 합니다. 
  Ex) arn:aws:events:ap-northeast-2:111122223333:event-bus/cops-health-collector-bus  
```

<br>


![img_3.png](/assets%2Fimages%2F24q3%2Fimg_3.png)

![img_4.png](/assets%2Fimages%2F24q3%2Fimg_4.png)

- `Event Forwarder`를 프로비저닝 할 맴버 계정을 규칙을 통해 타게팅 합니다.

조직 전체 또는 특정 OU를 선택적으로 적용할 수 있습니다.

특히 Organizations 에 전체 맴버 또는, 특정 계정만 포함, 특정 계정만 제외 등의 규칙을 적용하고 이렇게 타게팅 된 맴버 계정에서 
프로비저닝 할 리전을 다중으로 선택할 수 있습니다.   

<br>

![img_5.png](/assets%2Fimages%2F24q3%2Fimg_5.png)

최종 리뷰 단계를 거쳐서 타게팅 된 맴버 계정을 대상으로 프로비저닝을 하게 됩니다.

<br>

![img_6.png](/assets%2Fimages%2F24q3%2Fimg_6.png)

현재 StackSet 을 통해 프로비저닝되는 Stack 인스턴스를 볼 수 있습니다. 또한 조건을 조정하여 AWS 계정을 더 추가하거나 제외 할 수 있습니다.

<br>

![img_8.png](/assets%2Fimages%2F24q3%2Fimg_8.png)

![img_9.png](/assets%2Fimages%2F24q3%2Fimg_9.png)

위 화면은, 프로비저닝이 완료된 맴버 계정 중 하나입니다. `Event Forwarder` Bus 가 잘 구성된 걸 확인할 수 있습니다.

<br>

![img_10.png](/assets%2Fimages%2F24q3%2Fimg_10.png)

StackSet 을 통해 다수의 프로비저닝 진행상황을 모니터링 할 수 있습니다. 

<br>

![img_19.png](/assets%2Fimages%2F24q3%2Fimg_19.png)