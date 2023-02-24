---
layout: post
title: "AWS IAM User Management Practice"
date:  2023-02-15 18:00:00 +0900
categories:
- AWS
- Security
---

AWS IAM (Identity and Access Management)은 AWS 서비스 및 리소스에 대한 액세스를 관리하는 데 사용되는 서비스입니다. 

사용자 관리를 효과적으로 하기 위해서는 역할에 기반을 둔 사용자 그룹을 생성하고 여기에 적절한 권한(Policy)을 부여한 뒤 사용자를 추가 하는 것이 좋습니다.

여기서는 사용자 그룹, 그룹 정책 할당, 사용자 추가 및 MFA 할당과 패스워드 규칙 적용 등을 살펴보고 Matrix 를 통한 효과적인 IAM 관리 방법을 소개 합니다.

<br>

## 사용자 그룹 만들기

AWS 관리 콘솔 > IAM 대시보드 > `그룹`을 선택하고 `그룹 만들기` 버튼을 클릭하면 사용자 그룹을 생성 할 수 있습니다. 

![img.png](/assets/images/23q1/img.png)

아래와 같이 사용자 그룹명을 입력하여 생성할 수 있으며 생성 단계에서 사용자 (맴버쉽) 추가 와 액세스 권한 (Policy) 를 옵션으로 추가할수 있습니다. 

![img_1.png](/assets/images/23q1/img_1.png)

하단의 "Create Group" 버튼을 클릭하면 사용자 그룹이 생성 됩니다.  

<br>

## 사용자 그룹에 사용자 (Membership) 추가

사용자 그룹 상세 화면에서 `Add users` 버튼을 클릭 하면 아래와 같이 사용자를 추가할 수 있는 화면이 열립니다.    
여기에서 사용자를 검색 후 사용자를 선택 후 `Add users` 버튼을 통해 맴버쉽 추가를 할수 있습니다.   
  
![img_2.png](/assets/images/23q1/img_2.png) 


<br>


## 사용자 그룹에 권한 (Policy) 할당

사용자 그룹 상세 화면에서 `Add permissions > Attach policies` 버튼을 클릭 하면 아래와 같이 액세스 권한(Policy)를 추가할 수 있는 화면이 열립니다.    
여기에서 필요한 권한(Policy)를 검색 및 선택 하고 화면 하단의 "Add permissions" 버튼을 클릭하면 액세스 권한을 추가 할수 있습니다.

![img_3.png](/assets/images/23q1/img_3.png)


<br>


## MFA 사용 설정 
사용자 생성시 MFA 사용 설정을 활성화 하여 보안을 강화 하는 것이 좋습니다.

다음과 같이 특정 사용자를 생성 했다면 해당 사용자는 사용자 상세 화면에서 `Security credentials > Multi-factor authentication (MFA)` 메뉴의 `Assign MFA devices` 를 클릭하여 MFA 가상 디바이스를 할당 할 수 있습니다.    

![img_4.png](/assets/images/23q1/img_4.png)


<br>


아래 화면과 같이 Device Name 을 입력 후 Authenticator app 을 선택 후 `Next` 버튼을 클릭 합니다.  
**중요]** Device Name 이름은 사용자 이름과 일치하는 것이 좋습니다.  
사용자 이름이 `scott@sample.com` 이라고 한다면 Device Name 역시 동일하게 `scott@sample.com` 으로 입력 합니다.   

![img_5.png](/assets/images/23q1/img_5.png)

<br>

![img_6.png](/assets/images/23q1/img_6.png)

위 그림과 같이 QR 코드를 스캔하면 MFA code(토큰) 이 6자리 숫자로 보여 지게 되고, 
2 개의 토큰을 차례대로 기입 하고 하단의 `Add MFA` 버튼을 통해 MFA 디바이스를 할당 할 수 있습니다.    


<br>


## 비밀번호 정책 설정

IAM 대시보드에서 "Account Settings"을 선택하고 'Password policy' 항목을 편집 하면 새로운 비밀번호 정책을 작성합니다.  

![img_7.png](/assets/images/23q1/img_7.png)


이를 통해 최소한의 길이, 대문자와 소문자, 특수 문자, 숫자, 비밀번호 만료 기간 중복된 비밀번호 사용 금지 등 보다 강화된 비밀번호 정책을 적용 할 수 있습니다.  등이 포함될 수 있습니다.


<br>


## 주기적인 감사

AWS IAM 은 주기적인 감사를 통해 보안 위반 여부를 확인하고 보안 정책을 준수하는지 확인해야 합니다.

### 1. IAM 사용자 및 그룹 검토
- 모든 사용자 및 그룹에 대해 사용자 및 그룹이 필요한지 여부를 확인하고 필요하지 않은 사용자나 그룹을 삭제합니다.  
- 그룹에 사용자가 포함되어 있는지 확인하고 필요하지 않은 사용자를 삭제합니다.
- 그룹 및 사용자에게 부여된 권한을 확인하고 필요 없는 권한을 제거합니다.

### 2. IAM 정책 검토
- IAM 대시보드에서 모든 인라인 정책 및 연결된 정책을 검토 하여 필요하지 않은 정책을 삭제하거나 수정합니다.
- 모든 정책이 최신 상태인지 확인합니다.

### 3. IAM 액세스 로그 검토
IAM 액세스 로그는 AWS CloudTrail 서비스를 사용하여 확인 할 수 있습니다.
- CloudTrail 대시보드는 로그인 이벤트, 권한 부여 이벤트, 암호 재설정 이벤트 등을 확인 할 수 있습니다. 
- CloudTrail 로그에서 이상한 활동이나 보안 위반이 있는지 확인합니다.

### 4. MFA(Multi-Factor Authentication) 사용 검토
- IAM 대시보드에서 모든 사용자의 MFA 사용 여부를 검토합니다.
- 모든 사용자가 MFA를 사용하도록 정책을 통해 강제할 수 있습니다.
- MFA를 사용하지 않는 사용자에 대한 조치를 취합니다.

### 5. 비밀번호 정책 검토
- IAM 대시보드에서 비밀번호 정책을 검토합니다.
- 정책이 최신 상태이며, 보안 정책을 준수하는지 확인합니다.
- 필요에 따라 비밀번호 정책을 수정하고 사용자에게 알립니다.


<br>


## IAM 관리를 위한 모범 사례

우리는 사용자 그룹 관련된 주요 리소스를 생성하고 간단하게 구성 하는 것을 살펴보았습니다.  

여기서는 Matrix 를 통해 Admin, Developer, DBA, SysAdm, Viewer 과 같은 사용자 그룹을 만들고 여기에 적절한 액세스 정책과 사용자를 할당하는 전략을 살펴보도록 하겠습니다.

### 사용자 현황 Matrix 

사용자 액세스 권한을 관리하는 Matrix를 정의하면 사용자 및 액세스 정책을 보다 체계적으로 관리할 수 있습니다.


| User                     | Admin | Developer | DBA | Viewer | SysAdm |  MFA  |
|--------------------------|:-----:|:---------:|:---:|:------:|:------:|:-----:|
| admin@demoasacode.io     |   Y   |           |     |        |        |   Y   |
| manager@demoasacode.io   |       |           |     |        |   Y    |   Y   |
| lamp@demoasacode.io      |       |     Y     |  Y  |        |        |   Y   |
| symple@demoasacode.io    |       |     Y     |     |        |        |   Y   |
| devapple@demoasacode.io  |       |     Y     |  Y  |        |        |   Y   |
| scott@demoasacode.io     |       |           |  Y  |        |        |   Y   |
| tiger@demoasacode.io     |       |           |     |        |        |   Y   |
| banana@demoasacode.io    |       |           |     |   Y    |        |   Y   |
| melon@demoasacode.io     |       |           |     |   Y    |        |   Y   |
| car@demoasacode.io       |       |     Y     |  Y  |   Y    |        |   Y   |
| bicycle@demoasacode.io   |       |           |  Y  |   Y    |        |   Y   |



### 액세스 정책 Matrix

사용자 그룹의 역할에 적합한 액세스 정책(Policy) 관리 또한 다음과 같이 Matrix를 정의하면 효과적으로 관리할 수 있습니다.


| Policies                         | Admin | Developer | DBA | SysAdm | Viewer |
|----------------------------------|:-----:|:---------:|:---:|:------:|:------:|
| AdministratorAccess              |   Y   |           |     |        |        |
| AWSCertificateManagerFullAccess  |   Y   |           |     |        |        |
| AmazonRoute53FullAccess          |   Y   |           |     |   Y    |        |
| AmazonRoute53ReadOnlyAccess      |       |     Y     |     |        |        |
| ReadOnlyAccess                   |       |           |     |        |   Y    |
| AmazonRDSFullAccess              |       |           |  Y  |   Y    |        |
| AmazonRDSReadOnlyAccess          |       |     Y     |     |        |        |
| AmazonRedshiftFullAccess         |       |           |  Y  |        |        |
| AmazonRedshiftReadOnlyAccess     |       |           |     |   Y    |        |
| AmazonDynamoDBFullAccess         |       |           |  Y  |   Y    |        |
| AmazonDynamoDBReadOnlyAccess     |       |     Y     |     |        |        |
| AmazonElastiCacheFullAccess      |       |           |     |   Y    |        |
| AmazonElastiCacheReadOnlyAccess  |       |     Y     |     |        |        |
| AmazonAPIGatewayAdministrator    |       |     Y     |     |        |        |
| AmazonAPIGatewayReadOnlyAccess   |       |           |     |   Y    |        |
| AmazonMQFullAccess               |       |           |     |   Y    |        |
| AmazonMQReadOnlyAccess           |       |     Y     |     |        |        |
| AWSBatchServiceRole              |       |     Y     |     |   Y    |        |
| AWSBatchFullAccess               |       |     Y     |     |   Y    |        |
| CloudWatchFullAccess             |       |           |     |   Y    |        |
| CloudWatchReadOnlyAccess         |       |     Y     |  Y  |        |        |
| CloudWatchLogsFullAccess         |       |           |     |   Y    |        |
| CloudWatchLogsReadOnlyAccess     |       |     Y     |  Y  |        |        |
| AmazonEC2FullAccess              |       |           |     |   Y    |        |
| AmazonEC2ReadOnlyAccess          |       |     Y     |  Y  |        |        |
| AmazonECS_FullAccess             |       |           |     |   Y    |        |
| AmazonS3FullAccess               |       |           |     |   Y    |        |
| AmazonS3ReadOnlyAccess           |       |     Y     |  Y  |        |        |
| AWSCodeDeployFullAccess          |       |     Y     |     |        |        |
| AWSCodeDeployReadOnlyAccess      |       |           |  Y  |   Y    |        |
| AWSLambda_FullAccess             |       |     Y     |     |        |        |
| AWSLambda_ReadOnlyAccess         |       |           |  Y  |   Y    |        |
| AmazonEventBridgeFullAccess      |       |     Y     |     |        |        |
| AmazonEventBridgeReadOnlyAccess  |       |           |  Y  |   Y    |        |
| AmazonAthenaFullAccess           |       |     Y     |     |        |        |
| AmazonEMRFullAccessPolicy_v2     |       |           |     |   Y    |        |
| AmazonEMRReadOnlyAccessPolicy_v2 |       |     Y     |     |        |        |
| AWSCloudFormationFullAccess      |       |     Y     |     |        |        |
| DenyIPAddressPolicy              |   Y   |     Y     |  Y  |   Y    |   Y    |
| MFAForcePolicy                   |   Y   |     Y     |  Y  |   Y    |   Y    |
| DenyDataPipelinePolicy           |       |     Y     |  Y  |   Y    |        |
| DenyRDSLargeCreationPolicy       |       |           |  Y  |   Y    |        |

<br>

AWS 관리형 정책 뿐 아니라 관리를 위해 별도의 Custom 정책을 생성하는 경우 리소스 identifier 를 지정하면 보다 효과적으로 관리 할 수 있습니다.  


<br>


## Custom 액세스 정책 

AWS IAM 을 세밀하고 효과적으로 관리하기 위해선 사용자 관리형(Customer managed) 정책을 생성하고 적용 하는것이 좋습니다.  

AWS 클라우드 운영에 꼭 필요한 사용자 관리형(Customer managed) 정책을 소개할까 합니다.  

### DenyIPAddress

AWS 클라우드의 액세스를 특정 IP 대역만 접근 하도록 설정 할 수 있습니다. 

예를 들어 회사 네트워크의 NAT 아이피 또는 몇몇 VPN 네트워크에 대해서만 접근을 설정 할 수 있습니다. 

회사 네트워크의 NAT 아이피 대역이 `192.0.2.0/24` 이고 VPN 네트워크 대역이 `203.0.113.0/24` 이라면 여기에 정의된 아이피를 제외한 모든 접근을 거부하는 DenyIPAddress 정책은 다음과 같습니다. 

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "NotIpAddress": {
        "aws:SourceIp": [
          "192.0.2.0/24",
          "203.0.113.0/24"
        ]
      },
      "Bool": {"aws:ViaAWSService": "false"}
    }
  }
}

```

### MFAForcePolicy

MFA 인증을 하지 않은 사용자에 대해서 AWS 액세스를 거부 하도록 할 수 있습니다.  

주요한 Statement 로   
`AllowUserToCreateVirtualMFADevice` 은 로그인한 사용자만이 MFA 가상 디바이스를 생성할 수 있습니다.    
`BlockMostAccessUnlessSignedInWithMFA` 은 MFA 인증을 하지 않은 사용자는 NotAction 에 포함된 엑션을 제외한 모든 엑션이 Deny 됩니다.   
`AllowUserToManageTheirOwnMFA` 은 로그인한 사용자는 MFA 가상 디바이스를 조회하고, 활성화 하고, 다시 동기화를 할 수 있습니다.     
`AllowUserToDeleteTheirOwnMFAOnlyWhenUsingMFA` 과 `AllowUserToDeactivateTheirOwnMFAOnlyWhenUsingMFA` 은 MFA 를 통해 로그인한 사용자에 한하여 MFA 가상 디바이스를 비활성화 하고 삭제 할 수 있습니다.  

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListActions",
      "Effect": "Allow",
      "Action": [
        "iam:ListUsers",
        "iam:ListVirtualMFADevices"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowUserToCreateVirtualMFADevice",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice"
      ],
      "Resource": [
        "arn:aws:iam::*:mfa/*",
        "arn:aws:iam::*:user/${aws:username}"
      ]
    },
    {
      "Sid": "AllowUserToManageTheirOwnMFA",
      "Effect": "Allow",
      "Action": [
        "iam:ListMFADevices",
        "iam:EnableMFADevice",
        "iam:ResyncMFADevice"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    },
    {
      "Sid": "AllowUserToDeleteTheirOwnMFAOnlyWhenUsingMFA",
      "Effect": "Allow",
      "Action": [
        "iam:DeleteVirtualMFADevice"
      ],
      "Resource": [
        "arn:aws:iam::*:user/${aws:username}"
      ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "AllowUserToDeactivateTheirOwnMFAOnlyWhenUsingMFA",
      "Effect": "Allow",
      "Action": [
        "iam:DeactivateMFADevice"
      ],
      "Resource": [
        "arn:aws:iam::*:user/${aws:username}"
      ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "BlockMostAccessUnlessSignedInWithMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:ListMFADevices",
        "iam:ListUsers",
        "iam:ListVirtualMFADevices",
        "iam:ResyncMFADevice"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}

```

### DenyEC2StopAndTerminationMFAPolicy

MFA 로 로그인 하지 않은 사용자에 대해서 EC2 인스턴스를 중지하거나 삭제 하는것을 거부할 수 있습니다. 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyStopAndTerminateWhenMFAIsNotPresent",
      "Effect": "Deny",
      "Action": [
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": false
        }
      }
    }
  ]
}

```

[AWS Example policies](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/access_policies_examples.html) 예제를 참고하면 보다 체계적이고 강화된 정책을 구성 할 수 있습니다.  


<br>

### IAM 식별자(identifier) 를 통한 정책 관리 

[IAM 식별자](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/reference_identifiers.html) 는 사용자, 사용자 그룹, 역할, 정책 및 서버 인증서에 효과적으로 분류하는 기능을 합니다.  

액세스 정책(Policy)을 예를 들면 service-role, aws-service-role, job-function 과 같은 identifier 로 AWS 관리형 정책의 Baseline 을 확인할 수 있습니다.  

고객 역시 사용자 관리형 정책의 Baseline 을 위한 사용자 특화된 식별자(identifier) 를 정의할 수 있습니다.  
사용자 관리형 정책 Baseline 의 식별자(identifier)로 `/foundation` 으로 `DenyIPAddressPolicy`, `MFAForcePolicy` 와 `DenyEC2StopAndTerminationMFAPolicy` 정책을 생성 한다면, 사용자 관리형 정책의 ARN 은 다음과 같습니다. 

```
arn:aws:iam::111111111:policy/foundation/DenyIPAddressPolicy
arn:aws:iam::111111111:policy/foundation/MFAForcePolicy
arn:aws:iam::111111111:policy/foundation/DenyEC2StopAndTerminationMFAPolicy
```

아래는 특정 식별자(identifier) 로 분류된 정책들을 조회 할 수 있는 예시 입니다.  

![img_8.png](/assets/images/23q1/img_8.png)


<br>


## 자동화 

우리는 IAM 사용자 그룹, 사용자 그리고 정책을 어떻게 구성하면 좋은지 살펴 보았습니다.  
하지만 `주기적인 감사` 에 대해서 우기가 직접 AWS 관리 콘솔에 로그인 하고 AWS 리소스를 일일이 살펴본다면 너무 피로하고 시간이 오래 걸리는 작업일 것입니다.  
AWS 는 몇몇 핵심 서비스를 통해 자동화된 방법으로 `주기적인 감사` 를 진행 합니다.  

### CloudTrail

[CloudTrail](https://docs.aws.amazon.com/ko_kr/awscloudtrail/latest/userguide/cloudtrail-tutorial.html) 은 AWS의 모든 리소스에 대해서 변경한 내역에 대한 로깅 서비스입니다.  
AWS 리소스의 변경 내역을 추적하는 데 편리할 뿐만아니라 이를 통해 보안, 규정 준수, 운영 문제 해결 및 사건 대응 등에 도움이 됩니다. 


### AWS Config

[AWS Config](https://aws.amazon.com/ko/blogs/aws/aws-config-update-aggregate-compliance-data-across-accounts-regions/)는 AWS 리소스 구성을 추적하고 모니터링하여 이를 이용해 리소스 구성 변경 사항을 평가하고, 리소스 구성 준수 및 보안 준수를 검사할 수 있는 서비스입니다.

AWS Config는 지속적으로 AWS 리소스의 구성을 추적하고, 변경 사항을 검사하여 리소스 구성 준수 및 보안 준수를 유지할 수 있도록 도와주는 서비스입니다.  
[AWS Config 의 관리형 Rule](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html) 을 살펴보면 IAM 관련해서도 아래와 같이 주요한 Rule 을 확인할 수 있습니다.    

![img_9.png](/assets/images/23q1/img_9.png)  

### Access Analyzer
[Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) AWS 리소스에 대한 엑세스 권한 부여 시 발생할 수 있는 잠재적인 액세스 접근 관련 보안 위협을 식별하고, 수정할 수 있는 서비스입니다.  

예를 들어, S3 버킷을 위한 정책을 생성 했다고 가정하면 이 정책으로 어떤 클라이언트 또는 리소스가 접근하고 어떤 엑션을 수행할 수 있는지 가시적으로 보여주고 보완 할 수 있도록 도와 줍니다.   

![img_10.png](/assets/images/23q1/img_10.png)


![img_12.png](/assets/images/23q1/img_12.png)


<br>

또한 IAM Access Analyzer Policy Validation 은 IAM 정책 구성시 리소스의 ARN 오류나 속성 값 또는 과도한 액세스 권한에 대해서 보완할 것을 권고 합니다.   

![img_11.png](/assets/images/23q1/img_11.png)


<br>


## Conclusion 

IAM 을 사용하여 AWS 서비스 및 리소스에 대한 액세스를 관리하는 것은 보안에 중요한 역할을 합니다.
 
IAM을 효과적으로 사용하려면 위에서 설명 것과 같이 IAM을 계획하고 이를통해 보다 가시적으로 살펴볼 수 있도록 함으로써 예측 가능한 범주로 제어할 수 있는것이 중요합니다.  

그리고 자동화된 방식으로 IAM 액세스 내역을 지속적으로 감사 및 모니터링 함으로써 쉽고 강력하게 클라우드 서비스를 보호합시다.  

