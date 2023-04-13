---
layout: post
title: "Enhancing Security with AWS Secrets Manager"
date:  2023-02-15 18:00:00 +0900
categories:

- AWS
- Security

---



AWS에서 제공하는 [Secrets Manager](https://aws.amazon.com/ko/secrets-manager/)는 보안 관련 인증 정보, 비밀번호, API 키 등 애플리케이션에서 사용하는 민감한
정보들을 안전하게 저장하고 관리하는 Managed 서비스입니다.

일반적으로 애플맄이션 코드 안에 보안에 관련된 민감한 정보를 기술하는 것은 매우 위험하며, 의도하지 않게 상황 또는 실수에 의해 민감한 정보는 노출될 수 있습니다.
보안 관련 정보는 안전한 곳에 저장하고 필요할 때에만 호출하여 사용하는 것이 좋습니다.

실제로 프로젝트를 하다 보면 간단한 PoC 나 협럽 업체와 공동 작업하는 경우가 있습니다.

예전 저의 경험담을 말하자면 협력 업체분과 작업 하면서 특정 S3 버킷의 ReadOnly 권한만 부여한 API Key 를 제공 한적이 있는데,
문제는 그분이 AccessKey 를 코드에 기재하고 외부 소스코드 저장소에 업로드를 한 것이였습니다.

사실 최소 권안 액세스원칙을 준수하고 제한된 정책을 적용하여서 큰 피해는 발생하지 않겠지만,
정말 문제는 AWS 의 경고 메일과 함께 즉시로 보안 관련된 문제를 조치 하는 것이였습니다.

AccessKey 를 발급한 이전 시간의 모든 Key 를 교체 하는 등 여러 조치 항목들이 있는데 AWS 운영계 Account 였다면 정말이지 너무나 아찔한 경험을 할 뻔 하였습니다.

이 사례에서 보듯이 사람에게 아무리 주의를 준다고 하더라도 애초에 AccessKey 를 사용하지 않도록 AssumeRole 을 사용하도록 하거나 Secrets Manager 의 Key-Store 를 사용했으면 아무런
문제가 없었을 텐데 말입니다.

<br>

그렇다면 AWS Secrets Manager 를 사용하면 어떤 잇점이 있는지 살펴 보겠습니다.

1. 애플리케이션에서 민감한 정보들을 하드코딩하지 않아도 되므로, 코드의 보안성이 향상됩니다.
2. 데이터를 안전하게 저장하고 암호화하므로, 데이터 유출과 같은 보안 문제를 사전에 예방할 수 있습니다.
3. 애플리케이션에 필요한 데이터를 중앙에서 관리할 수 있으므로, 유지 및 관리가 간편합니다.
4. 액세스 이력 등 모니터링 기능을 제공하므로, 데이터 액세스를 추적하고 보안 상태를 모니터링할 수 있습니다.

<br>

## Architecture & Use Case

Secrets Manager 의 아키텍처는 그림에서 보듯이 완전한 SaaS 서비스로 관리되며 Serverless 로 구성됩니다.

![usecase](https://d1.awsstatic.com/diagrams/Secrets-HIW.e84b6533ffb6bd688dad66cfca36622c2fa7c984.png)

Secrets Manager 는 AWS KMS 키를 기반으로 보안 데이터를 안전하게 저장 관리 하고, 운영 관리자의 개입 없이도
AWS Lambda 를 통해 주기적으로 Key 정보를 교체가 가능합니다.    
애플리케이션은 Secrets Manager 에서 관리하는 중요 보안 정보를 런타임 시점에 안전하게 가져와서 해당 서비스들을 액세스 할 수 있습니다.


<br>

## Secrets Manager 를 통한 정보 저장

AWS Secrets Manager를 사용하면 비밀 번호와 같은 보안 정보를 중앙 집중식으로 안전하게 저장할 수 있습니다.     
이렇게 함으로서 중요한 데이터가 외부로 유출 된다 하더라도 암호화된 상태로 저장되므로 보안 위험을 줄일 수 있습니다.

우선, AWS 관리 콘솔에 로그인을 한 뒤 'Secrets Manager' 화면의 `Store a new secret` 버튼을 클릭하여 손쉽게 Secrets 을 구성 할 수 있습니다.

### Choose secret type

AWS 주요 서비스인 RDS, Redshift 의 접속 정보나 또는 외부 인증을 위한 O-Auth 크리덴셜 정보 등을 Secrets Manager 로 안전하게 보호 할 수 있습니다.
우리는 이미 액세스 해야할 RDS 클러스터가 있고 여기에 안전하게 액세스 하는 Secrets 를 생성하고자 한다면 `Credentials for Amazon RDS database`
유형을 선택 할 수 있습니다.

![img_13.png](/assets/images/23q1/img_13.png)

RDS 액세스를 위한 `user name` 과 `password` 를 기입 합니다.

<br>

![img_14.png](/assets/images/23q1/img_14.png)

암호화를 위한 Encryption key 를 선택 합니다. AWS 관리형 Key 는 `aws/secretsmanager` 입니다. 만약 CMK 를 생성 했다면 해당 Key 를 선택 해도됩니다.

Database 는 위 username, password 로 액세스 가능한 RDS 인스턴스를 선택 하고 화면 하단의 `Next` 버튼을 클릭 합니다.

### Configure secret

![img_15.png](/assets/images/23q1/img_15.png)

`Secrets name` 과 `Description` 정보 및 Tags 정보를 기입 합니다.

<br>

![img_16.png](/assets/images/23q1/img_16.png)

`Resource permissions` 정책 구성을 통해 애플리케이션이 Secrets Keys 를 액세스 가능하도록 합니다.

- 애플리케이션이 특정 secret store 의 key 를 액세스 하기 위한 IAM 정책 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AccessSecretKeysForApps",
      "Effect": "Allow",
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-2:111111111:secret:dev/aurora/apple*"
      ],
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:ListSecrets",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "*"
    }
  ]
}

```

- Lambda 를 통해 secret store 의 key 를 주기적으로 교체 하려면 다음과 같은 IAM 정책이 필요 합니다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RotateSecretsKeysByLambda",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:PutSecretValue",
        "secretsmanager:UpdateSecret",
        "secretsmanager:CreateSecret",
        "secretsmanager:DeleteSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-northeast-2:111111111:secret:dev/aurora/apple*"
      ],
      "Condition": {
        "ForAllValues:StringEquals": {
          "secretsmanager:VersionStage": "AWSCURRENT"
        }
      }
    }
  ]
}

```

<br>

`Replicate secret` 은 현재 Secrets Key 를 다른 리전으로 복제 합니다.
이렇게 하면 중앙에서 관리하는 메타 정보를 다른 리전에서도 참조 할 수 있습니다.

화면 하단의 `Next` 를 클릭하여 다음 단계로 이동 합니다.

<br> 

### Configure rotation

![img_17.png](/assets/images/23q1/img_17.png)

선택적 구성으로 자동화된 방식으로 주기적으로 암호화 Key 를 교체하기 위한 설정을 할 수 있습니다.

암호화 Key 를 교체하기 위해선 사전에 Key 교체를 위한 Lambda 애플리케이션을 구현하고 배포 하여야 합니다.

관련한 자세한
정보는 [AWS Secrets Manager rotation function templates](https://docs.aws.amazon.com/secretsmanager/latest/userguide/reference_available-rotation-templates.html)
과
[AWS Secrets Manager Rotation Lambda Functions](https://github.com/aws-samples/aws-secrets-manager-rotation-lambdas) 예제를
참조 할 수 있습니다.

추후에 이것과 관련된 코드를 프로젝트로 공유 하도록 할까 합니다.

여기서는 `Automatic rotation` 속성을 비활성화 하고 다음 단계로 이동 합니다.

<br>

### Review

Review 를 통해 최종 검토를 하고 Secrets Key 정보를 생성 합니다.

![img_18.png](/assets/images/23q1/img_18.png)

![img_19.png](/assets/images/23q1/img_19.png)

위와 같이 프로그램 언어별로 Secret Keys 정보를 액세스할 수 있는 샘플 코드를 템플릿으로 제공 하고 있습니다.

### Check

아래와 같이 `dev/aurora/apple` 이름으로 저장된 Secrets Key 를 확이할 수 있습니다.

![img_20.png](/assets/images/23q1/img_20.png)

`Secret valueInfo` 의 `Retrieve secret value` 를 통해 현재 저장된 key 정보는 물론 여기에 필요한 속성을 더 추가 할 수 있습니다.

![img_21.png](/assets/images/23q1/img_21.png)

Json 형태로 아래와 같은 형태로 관리 됩니다. 
 ```json
 {
  "username": "*****",
  "password": "******",
  "engine": "mysql",
  "host": "*********",
  "port": 3306,
  "dbClusterIdentifier": "******"
}
 ```

## 애플리케이션 참조 예시

Spring Boot 애플리케이션을 통해 Secret Keys 정보를 편리하게 액세스 하려면 
[spring-boot-starter-aws-secrets-manager](https://central.sonatype.com/artifact/io.github.thenovaworks/spring-boot-starter-aws-secrets-manager/0.9.5/overview)
라이브러리를 이용 할 수 있습니다.

- Maven dependency 예시

```
<dependency>
    <groupId>io.github.thenovaworks</groupId>
    <artifactId>spring-boot-starter-aws-secrets-manager</artifactId>
    <version>0.9.5</version>
</dependency>
```

- Secret Keys 에 구성된 정보를 `@SecretsValue` 어노테이션을 통해 쉽게 참조 할 수 있습니다.

```java
import io.github.thenovaworks.spring.aws.secretsmanager.autoconfigure.SecretsValue;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
public class SampleService {

    @SecretsValue("dev/aurora/apple")
    private Map<String, String> appleRdsInfo;

}
```

<br>

## Conclusion

지금까지 AWS SecretsManager를 살펴보고 어떻게 하면 민감한 데이터에 대해 보안을 강화하고, 
애플리케이션 코드에서 분리하여 유지보수성과 보안성을 높일 수 있는지 알아 보았습니다. 

덧붙여 기업이나 개인이 보안에 민감한 데이터를 보호하고 애플리케이션에서도 쉽게 액세스 하는 방법이 어렵지 않다는걸 예제로 살펴 볼 수 있었습니다. 


