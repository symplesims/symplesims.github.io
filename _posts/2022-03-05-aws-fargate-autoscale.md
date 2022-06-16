---
layout: post
title: "AWS Fargate Auto-scaling 정책 적용"
date:  2022-03-05 19:00:00 +0900
categories: 
  - AWS
  - Fargate
  - Autoscale
  - Terraform
  - Automation
---

AWS Fargate Auto-Scale 정책을 테라폼 코드로 빠르게 적용하여 서비스를 보다 탄력적으로 운영합시다.    


<br>


## Background 

AWS 가 제공하는 서비스들을 보면 Elastic 을 아주 많이 강조 합니다.  
10 명의 사용자가 서비스를 이용하는 워크로드와 10,000 명의 사용자가 서비스를 이용하는 워크로드는 사용하는 리소스의 규모 및 통합면에서 상당한 차이가 있습니다.  
문제는 워크로드 규모를 고정 할 수 없다는 것 입니다.  
서비스 품질 및 운영 비용을 최적화 하기 위해 서비스 이용자가 폭발적으로 늘어나면 이에 대응하여 워크로드를 확장해야 하고 이용자가 줄어들면 축소 해야 하니까요.  
AWS 는 이런 문제를 Elastic 서비스를 통해 자동적으로 워크로드를 탄력적으로 확장 하거나 축소 하도록 지원 합니다. 

<br>

## Auto-Scale 배경 및 작동 방식 
![](/assets/images/220304/img.png)

위 그림과 같이 ECS 서비스는 몇몇 리소스의 협력 으로 Auto-Scale 정책을 통해 워크로드 규모를 조정 합니다. 

Auto-Scale 동작 방식의 컨셉은

1. CloudWatch 에서 수집된 매트릭 측정 지표로 
2. 어떤 기준으로 확장(Scale-Out) 또는 축소(Scale-In) 할 것인가?

입니다. 


<br>

### 리소스 메트릭 수집
Amazon ECS 는 Auto-Scale 작동을 위한 애플리케이션의 CPU, MEMORY, ALB Request 서비스 측정 메트릭 지표를 1분 간격으로 CloudWatch 로 보냅니다.

<br>

### Auto-Scale 정책 구성
![](/assets/images/220304/img_1.png)

Auto-Scale 정책 구성을 통해 기준 정보를 설정 합니다.   

주요 속성으로 `최소 작업 개수`, `원하는 작업 개수`, `최대 작업 개수` 와 함께 `Auto Scaling 을 동작 시키기 위한 서비스 연결 역할(IAM 역할)`이 필요 합니다. 

- [Application Auto Scaling에 대한 서비스 연결 역할](https://docs.aws.amazon.com/ko_kr/autoscaling/application/userguide/application-auto-scaling-service-linked-roles.html) 중 `AWSServiceRoleForApplicationAutoScaling_ECSService` 참조  

<br>

### Auto-Scale 조정 정책
`Auto-Scale 조정 정책` 은 Scale-In 과 Scale-Out 의 조건을 기입 하여 실제로 Auto-Scale 이 작동(트리거) 되도록 설정 합니다.  

조정 정책 유형은 AWS 관리 메트릭 기준의 `대상 추적 조정 정책` 과 사용자 정의 기준의 `단계 조정 정책`이 있습니다. 

![](/assets/images/220304/img_2.png)

주요 속성은 다음과 같습니다. 

| 속성 명          | 필수 여부 | 설명                                                                                                                                                                                                                |
|---------------|-------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 정책 이름         | Y     | 조정 정책 이름 입니다.                                                                                                                                                                                                     |
| ECS 서비스 측정치  | Y     | Scale In/Out 을 위한 메트릭 기준 정보 입니다. ECSServiceAverageCPUUtilization, ECSServiceAverageMemoryUtilization, ALBRequestCountPerTarget 중 에 선택이 가능 하며, ALBRequestCountPerTarget 메트릭을 기준으로 하려면 ECS 서비스가 ALB 와 연결되어 있어야 합니다. |
| 대상 값          | Y     | Scale In 또는 Scale Out 이 작동 되기 위한 매트릭 기준 값 입니다.                                                                                                                                                                    |
| 확장 휴지 기간     | N     | Scale Out 작업 동안의 휴지 시간 입니다. (기본 값: 300 초)                                                                                                                                                                         |
| 축소 휴지 기간     | N     | Scale In 작업 동안의 휴지 시간 입니다.  (기본 값: 300 초)                                                                                                                                                                         |
| 축소 비활성화      | N     | 축소 비활성화를 체크 하면 Scale In 동작을 하지 않습니다.  ((기본 값: 비활성)                                                                                                                                                                |

휴지 시간은 Scale In/Out 작업이 완료될 때까지 대기하는 시간으로 안정적인 서비스 

- [Target-Tracking 대상 추적 조정 정책](https://docs.aws.amazon.com/ko_kr/autoscaling/application/userguide/application-auto-scaling-target-tracking.html) 참고
- [Stepscaling 단계 조정 정책](https://docs.aws.amazon.com/ko_kr/AmazonECS/latest/developerguide/service-autoscaling-stepscaling.html) 참고

<br><br>

## Pre-Requisite
사전에 AWS ECS Fargate 구성이 되어 있어야 합니다. 

Fargate 구성은 지난 글 ["Automation Building AWS Fargate & Deploy application"](/devops/aws%20fargate/terraform/automation/2022/01/15/automation-building-aws-fargate.html) 을 참고 합시다.   

<br><br>


## 대상 추적 조정 정책 시나리오 

[대상 추적 조정 정책](https://docs.aws.amazon.com/ko_kr/autoscaling/ec2/userguide/as-scaling-target-tracking.html) 은 리소스 메트릭의 평균 값을 통해 Scale 을 조정 합니다.  

리소스 메트릭(CPU, Memory, ALB Request Count)의 지표가 Auto-Scale 조정 정책에 정의한 대상(평균) 값을 초과 하는 경우 Scale-Out 을 하고, 미만인 경우 Scale-In 을 합니다. 

<br>

### 평균 CPU 사용율 기준 대상 추적 조정 정책의 적용 

Cloudwath 에 1분간 수집한 Auto-Scale 그룹의 평균 CPU 사용율 60% 를 기준으로 Scale In - Out 이 동작 하도록 AWS 관리 콘솔을 통해 다음과 같이 구성 할 수 있습니다.

![](/assets/images/220304/img_3.png)

<br>

- 위의 정책을 Terraform 을 통해 구현하는 예시는 다음과 같습니다.

```
# AutoScale 대상 ECS 애플리케이션 정의 및 Scale 정책 구성 
resource "aws_appautoscaling_target" "scale_target" {
  service_namespace  = "ecs"
  resource_id        = "<service>:<ecs-cluster-id>/<ecs-service-id>"
  scalable_dimension = "ecs:service:DesiredCount"
  min_capacity       = <min_capacity>
  max_capacity       = <max_capacity>
}

# 평균 CPU 사용율과 Scale In / Out 의 휴지 시간을 정의
resource "aws_appautoscaling_policy" "policy_cpu" {
  name               = "<ecs-service-id>-CPUUtilization"
  resource_id        = aws_appautoscaling_target.scale_target.resource_id
  scalable_dimension = aws_appautoscaling_target.scale_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.scale_target.service_namespace
  policy_type        = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }

    target_value       = 60
    scale_in_cooldown  = 180
    scale_out_cooldown = 240
    disable_scale_in = true
  }
}
```


### 평균 Memory 사용율 기준 대상 추적 조정 정책의 적용

Cloudwath 에 1분간 수집한 Auto-Scale 그룹의 평균 Memory 사용율 80% 를 기준으로 Scale In - Out 이 동작 하도록 AWS 관리 콘솔을 통해 다음과 같이 구성 할 수 있습니다.

![](/assets/images/220304/img_4.png)

<br>

- 위의 Memory Autoscaling 정책을 Terraform 을 통해 구현하는 예시는 다음과 같습니다.

```
# AutoScale 대상 ECS 애플리케이션 정의 및 Scale 정책 구성 
resource "aws_appautoscaling_target" "scale_target" {
  service_namespace  = "ecs"
  resource_id        = "<service>:<ecs-cluster-id>/<ecs-service-id>"
  scalable_dimension = "ecs:service:DesiredCount"
  min_capacity       = <min_capacity>
  max_capacity       = <max_capacity>
}

# 평균 Memory 사용율과 Scale In / Out 의 휴지 시간을 정의
resource "aws_appautoscaling_policy" "policy_mem" {
  name               = "<ecs-service-id>-MemoryUtilization"
  resource_id        = aws_appautoscaling_target.scale_target.resource_id
  scalable_dimension = aws_appautoscaling_target.scale_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.scale_target.service_namespace
  policy_type        = "TargetTrackingScaling"

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }

    target_value       = 80
    scale_in_cooldown  = 180
    scale_out_cooldown = 240
    disable_scale_in = true
  }
}
```



<br><br>

# Reference
- [AWS Auto Scaling](https://aws.amazon.com/ko/autoscaling/)
- [AWS Auto Scaling – Unified Scaling For Your Cloud Applications](https://aws.amazon.com/ko/blogs/aws/aws-auto-scaling-unified-scaling-for-your-cloud-applications/)


