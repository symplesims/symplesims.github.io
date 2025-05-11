---
layout: post
title: "Security Threats and Response Strategies through AWS WAF"

date:  2025-03-21 18:00:00 +0900
categories:
   - AWS WAF
   - Best Practices
---

## AWS WAF를 통한 보안 위협과 대응 전략

인터넷 기반 웹 서비스는 다양한 보안 위협에 노출 되어 있습니다. 특히, DDoS 공격, SQL 인젝션, XSS, Bot 트래픽 등은 서비스의 가용성과 신뢰성을 저해할 수 있습니다.
이러한 위협에 대응하기 위해 AWS에서는 **AWS WAF(Web Application Firewall)**를 제공하여, 애플리케이션 계층(Layer 7)의 공격을 효과적으로 차단할 수 있도록 지원합니다.


## WAF 와 통합 가능한 서비스

AWS WAF(Web Application Firewall)는 아래와 같이 보안이 필요한 AWS 서비스와 통합할 수 있습니다. 

![waf-supported-resources](https://aws.github.io/aws-security-services-best-practices/guides/waf/prerequisites/images/waf-supported-resources.png)



## 웹서비스 아키텍처 모범 사례 

AWS 클라우드에서 웹 서비스 제공을 위해 보안이 강화 되고 간단한 아키텍처로 WEB UI 는 `CloudFront`로 배포 하고 `Backend API`는 `ALB (Application Load Balancer)` 또는 `API Gateway + NLB(Network Load Balancer)` 와 통합 하는 방식을 권고합니다.

## Application Load Balancer (ALB) 보안 그룹 방어 전략

![img.png](/assets/images/25q2/img.png)

위 아키텍처는 ALB (Application Load Balancer) 의 보안 그룹을 통해 CloudFront 의 요청만 허용하고 다른 어떤 요청도 거부할 수 있습니다.

특히 AWS를 이용하는 고객은 ALB 를 위한 보안 그룹에 AWS가 제공하는 CloudFront 에 대한 Managed Prefix List 를 사용 하여 아주 쉽게 방어 할 수 있습니다. 

VPC Managed prefix lists 목록에서 [global.cloudfront.origin-facing](https://aws.amazon.com/ko/blogs/korea/limit-access-to-your-origins-using-the-aws-managed-prefix-list-for-amazon-cloudfront/) 로 검색하면 확인 할 수 있습니다.


## WAF + Application Load Balancer (ALB)를 통합한 아키텍처

위의 ALB 보안 그룹 방어 전략은 Hacker 의 공격이 보안 그룹에 의해 차단 될 수 있겠지만, 해커의 공격 트래픽이 Backend ALB 영역 까지 도달 하게 된다는 것 입니다. 

이 것 자체가 DDoS 공격을 하게 되면 직접 적인 피해를 입게 되므로 보다 앞서서 해커의 공격을 방어할 필요가 있습니다. 

아래 다이어그램은 [ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html) 와 [WAF Reginal](https://aws.amazon.com/waf/) 를 통합한 아키텍처 입니다.

AWS 클라우드가 제공하는 관리형 WAF 방화벽을 Backend 서비스 레이어 앞쪽에 배치함으로써 대부분의 침입자 공격을 차단할 수 있습니다.


![img_1.png](/assets/images/25q2/img_1.png)



### CloudFront HTTP 헤더를 이용한 WAF 방화벽 전략

`CloudFront`액세스만 허용하도록 WAF 방화벽 규칙을 구성할 수 있습니다.  

`CloudFront`의 Behavior 설정 중 `Cache policy and origin request policy` 정책 에서 `AllViewerAndCloudFrontHeaders`를 설정 하면 CloudFront의 Geo-Location 정보가 아래와 같이 HTTP 헤더로 제공됩니다.

![img_2.png](/assets/images/25q2/img_2.png)

- [cloudfront-geolocation-headers](https://aws.amazon.com/ko/about-aws/whats-new/2020/07/cloudfront-geolocation-headers/) 샘플

```
{
  'user-agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/136.0.0.0 Safari/537.36',
  'x-amz-cf-id': 'R14kBawplRj6Xk98UVmNry1HNwde-M76F3TkdoeWcbKUZpnw6QHc2A==',
  'x-forwarded-for': '222.0.0.106, 165.0.0.9',
  'via': '1.1 0859cbbec76cd864e788367b8aaae34a.cloudfront.net (CloudFront)',
  'accept-encoding': 'br,gzip',
  'cloudfront-viewer-country': 'KR',
  'cloudfront-viewer-country-name': 'South Korea',
  'cloudfront-viewer-country-region': '11',
  'cloudfront-viewer-country-region-name': 'Seoul',
  'cloudfront-viewer-city': 'Seoul',
  'cloudfront-viewer-postal-code': '04****',
  'cloudfront-viewer-time-zone': 'Asia/Seoul',
  'cloudfront-viewer-latitude': 'x7.50030',
  'cloudfront-viewer-longitude': '1**.95650',
  ....
}
```

`CloudFront` 헤더 정보 중 Geo-Location 중 하나인 `cloudfront-viewer-country` 값이 `KR`, `US` 인 항목만 허용 하도록 WAF 방화벽을 아래와 같이 구성할 수 있습니다. 

- `AllowKRAndUSCloudFrontViewerCountry` Rule-Group 설정 참고 

```json
{
  "Name": "AllowKRAndUSCloudFrontViewerCountry",
  "Priority": 10,
  "Action": {
    "Allow": {}
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "AllowKRAndUSCloudFront-Country-metric"
  },
  "Statement": {
    "OrStatement": {
      "Statements": [
        {
          "ByteMatchStatement": {
            "SearchString": "KR",
            "FieldToMatch": {
              "SingleHeader": {
                "Name": "cloudfront-viewer-country"
              }
            },
            "TextTransformations": [
              {
                "Priority": 0,
                "Type": "LOWERCASE"
              }
            ],
            "PositionalConstraint": "EXACTLY"
          }
        },
        {
          "ByteMatchStatement": {
            "SearchString": "US",
            "FieldToMatch": {
              "SingleHeader": {
                "Name": "cloudfront-viewer-country"
              }
            },
            "TextTransformations": [
              {
                "Priority": 0,
                "Type": "LOWERCASE"
              }
            ],
            "PositionalConstraint": "EXACTLY"
          }
        }
      ]
    }
  }
}
```

위의 WAF Rule-Group 동작은 아래 CloudWatch Logs Insights 쿼리를 통해 확인을 할 수 있습니다. 

```
fields @timestamp, @message
| parse @message '{"name":"CloudFront-Viewer-Country","value":"*"}' as geoloc
| filter geoloc = 'KR'
| sort @timestamp desc
| limit 20
```

![img_3.png](/assets/images/25q2/img_3.png)



### CloudFront `IPSet`을 이용한 WAF 방화벽 전략

경험 많은 해커의 경우 HTTP 헤더를 변조 하여 WAF 방화벽을 무력화 할 수 있습니다.

심지어 해커의 Device IP Address 를 `CloudFornt`의 IP 로 변조한 공격도 가능합니다. 

하지만, 해커의 IP 주소를 변조 공격이 WAF를 통과했다 하더라도 TCP 프로토콜 규격상 SYN-ACK는 결국 CloudFront IP로 응답하기에 Hacker 로 전달 되지 않기 때문에 안전 하게 공격이 차단 됩니다.


AWS는 주요 서비스에 대해 `VPC Managed prefix list`의 [global.cloudfront.origin-facing](https://aws.amazon.com/ko/blogs/korea/limit-access-to-your-origins-using-the-aws-managed-prefix-list-for-amazon-cloudfront/) 뿐만 아니라, 
[aws-ip-ranges](https://docs.aws.amazon.com/vpc/latest/userguide/aws-ip-ranges.html) 를 통해 CloudFront 의 Edge Location 에 대한 IP 정보도 API 를 통해 제공 하고 있습니다.


아래 `ip-ranges.json` CLOUDFRONT 서비스에서 `region` 속성이 `GLOBAL`과 `ap-south-1`와 같이 해당 리전의 Edge Location 에 관한 CIDR 를 확인할 수 있습니다.

```
curl -O https://ip-ranges.amazonaws.com/ip-ranges.json 
```

우리는 `WAF & Shield` 의 `IP sets` 을 통해 Global 및 Regional 에 대한 CloudFront IP 를 구성할 수 있습니다.

- `cloudfront-ipsets` WAF Rule 예시

![img_4.png](/assets/images/25q2/img_4.png)

위 AWS IP Sets 으로 관리 되는 `cloudfront-ipsets` 의 ARN 값이 `arn:aws:wafv2:us-east-1:111122223333:regional/ipset/cloudfront-ipsets/a5a6` 이라면 
아래와 같이 WAF Rule Group 을 생성할 수있습니다. 

- cloudfront-rule-group Rule Group 예시

```json
{
  "Name": "cloudfront-ipsets-rule",
  "Priority": 11,
  "Action": {
    "Allow": {}
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "cloudfront-ipsets-metric"
  },
  "Statement": {
    "IPSetReferenceStatement": {
      "ARN": "arn:aws:wafv2:us-east-1:111122223333:regional/ipset/cloudfront-ipsets/a5a6"
    }
  }
}
```


## WAF 를 활용한 공사중 페이지 전략

대규모 서비스 패치와 같이 휴일을 이용하여 장시간 서비스 절체 후 작업을 해야 하느 경우 공사중 페이지를 로드 하고 작업 해야 하는 경우가 있습니다.

이를 위해 Route53 도메인 부터 ALB 라우팅 등 프로덕션 환경을 대상 으로 여러 리소스를 수정 하는 것은 운영 환경에 변경이 일어 나는 위험한 작업일 뿐만아니라, 
몇몇 리소스의 설정을 변경해야 하는 부담스러운 작업이 됩니다.

하지만 ₩WAF`를 이용 하여 `maintenance-rule` 과 같은 단순한 규칙 으로 아주 쉽고 안전 하게 공사중 페이지를 작업할 수 있습니다. 

그리고 작업을 위한 내부 개발팀의 담당자에 대한 IP 들은 WAF 에 차단 되지 않도록 예외 처리를 하는 IP sets `deny-not-included-ipsets` 를 두어 패치 및 검증을 위해 허용할 수 있습니다. 


### 공사중 페이지를 위한 `maintenance-rule` WAF Rule  
`maintenance-rule` Rule Group 을 아래와 같이 `deny-not-included-ipsets` 에 포함된 IP 가 아니 라면 차단(Block) 하도록 규칙을 생성 하고, 
그 이외의 모든 클라이언트의 Action은 `maintenance-html` 키에 대한 CustomResponse HTML 컨텐츠를 응답하도록 구현할 수 있습니다.   

- `arn:aws:wafv2:us-east-1:111122223333:regional/ipset/deny-not-included-ipsets/0e3a****` IP sets 엔 개발팀 네트워크에 대한 IP 대역이 등록되어 있어야 합니다.
- `Custom response bodies` 의 key `maintenance-html` 엔 공사중 페이지를 렌더링 할 Html 컨텐츠가 구성 됩니다. 

- `maintenance-rule` Rule Group 예제  

```json
{
  "Name": "maintenance-rule",
  "Priority": 1,
  "Statement": {
    "NotStatement": {
      "Statement": {
        "IPSetReferenceStatement": {
          "ARN": "arn:aws:wafv2:us-east-1:111122223333:regional/ipset/deny-not-included-ipsets/0e3a****"
        }
      }
    }
  },
  "Action": {
    "Block": {
      "CustomResponse": {
        "ResponseCode": 503,
        "CustomResponseBodyKey": "maintenance-html"
      }
    }
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "maintenance-rule-metric"
  }
}
```

- `maintenance-html` 구성 예시

`maintenance-rule` Rule Group 에 `Custom response bodies` 를 통해 아래와 같이 공사중 페이지를 생성 할 수 있습니다. 

![img_5.png](/assets/images/25q2/img_5.png)

참고로 html 항목은 최대 4 K bytes 로 제한 됩니다.


## Conclude

