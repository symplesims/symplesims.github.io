---
layout: post
title: "Securing S3 Origins with CloudFront Signed URLs"

date:  2024-03-15 20:00:00 +0900
categories:
   - Security
   - Modernization
   - HandsOn
---

CloudFront Signed URL로 S3 오리진 컨텐츠 보호 하기   

<br> 

얼마전에 S3 버킷에 비용 리포트를 S3버킷에 export 하고 해당 리포트를 `pre-signed-url`을 생성하여 고객에게 이메일을 통해 전달하는 형태의 애플리케이션 서비스를 살펴보았습니다.

문제는 S3 버킷의 `pre-signed-url`은 사용자와 애플리케이션에 따라 각각 제약사항이 있습니다. 
[Who can create a presigned URL](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html#who-presigned-url) 를 보면 사람의 경우 1분에서 최대 7일 까지이며,
애플리케이션은 1분에서 STS 임시자격증명이 허용되는 최대 36 시간을 초과할 수 없습니다.  

고객은 이메일로 제공받은 비용리포트 url이 7일간 다운로드할 수 있어야 했으므로 비용리포트 서비스의 애플리케이션에서 `pre-signed-url`을 생성하는 절차가 복잡했습니다.
STS 토큰을 얻고 내부적으로 Pre-Signed URL 을 생성할 수 있는 사용자 AssumeRole 역할로 전환하여 `pre-signed-url`을 생성하고 있었습니다.

여기에서 두가지 중요한 보안 개선점이 나타납니다.

- 애플리케이션이 사용자 AssumeRole 역할로 전환하기 위해선 반드시 사용자의 credentials 키가 필요하게 됩니다. 물론, 이 정보가 소스코드에 노출되면 안되기에 secrets-manager를 통해 키를 액세스 하고 있었습니다.
- IAM 보안정책에서 사용자에게 발급받은 credentials 키들은 90일 간격으로 로테이션 되도록 보안 활동과 감사가 이루어지고 있습니다. 이 경우 액세스키를 주기적으로 로테이션 해야만 하는 아주 번거러운일이 발생됩니다. 

이 문제를 CloudFront 와 S3 오리진을 통합하는 아주 작은 아키텍처개선으로 쉽게 해결할 수 있었습니다. 


<br>


## AWS CloudFront 소개
AWS CloudFront를 소개하자면, Amazon S3(Simple Storage Service)에 저장된 객체에 대해 콘텐츠 전송 네트워크(CDN: Content Delivery Network)를 해주는 서비스입니다. 

뿐만아니라, 전 세계에 분산된 엣지 로케이션을 통해 정적 및 동적 웹 콘텐츠를 빠르고 안전하게 전송할 수 있습니다.

Amazon S3(Simple Storage Service)는 AWS의 객체 스토리지 서비스로, 웹 애플리케이션의 정적 파일, 미디어 파일, 백업 데이터 등 다양한 유형의 데이터를 저장하고 관리할 수 있습니다.

### CloudFront를 이용한 CDN 서비스의 장점:

- 글로벌 콘텐츠 전송: CloudFront의 전 세계 엣지 로케이션을 활용하여 콘텐츠를 사용자 가까이에서 빠르게 제공할 수 있습니다.
- 높은 가용성: 엣지 로케이션의 중복성과 자동 장애 조치로 가용성이 높아집니다.
- 보안 강화: CloudFront와 S3의 다양한 보안 기능으로 데이터를 안전하게 보호할 수 있습니다.
- 비용 절감: 콘텐츠 배포 비용이 절감되고, S3의 저렴한 스토리지 요금으로 비용 효율성이 높습니다.
- 간편한 배포 및 관리: AWS 콘솔이나 API를 통해 손쉽게 구성하고 관리할 수 있습니다.

<br>

## CloudFront 기반 S3 Origin 보호 아키텍처 


아래 다이어그램은 클라인트가 직접적으로 S3 객체를 접근하는 것을 원천 차단하고, `CloudFront`를 통해서만 액세스가 가능한 아키텍처입니다. 

![ProtectS3BucketOrigin](https://raw.githubusercontent.com/symplesims/symplesims.github.io/main/assets/images/24q1/cfarch01.svg)

<br>

## S3 버킷 및 Origin 보안


S3 버킷에 저장된 데이터를 안전하게 보호하기 위해서는 적절한 보안 조치가 필요합니다. 다음과 같은 방법을 통해 S3 오리진의 보안을 강화할 수 있습니다.

- S3 버킷의 `Permissions` 에서 `Block public access`를 다음과 같이 제한 합니다.

![img.png](/assets%2Fimages%2F24q1%2Fimg.png)


- AWS 관리형 키(SSE-S3) 또는 고객 관리형 키(SSE-KMS)를 사용하여 서버 측 암호화을 다음과 같이 설정 합니다. 

![img_2.png](/assets%2Fimages%2F24q1%2Fimg_2.png)

AWS 관리형 또는 고객 관리형 KMS 키인 SSE-KMS 를 사용할 경우 Bucket Key 를 캐시하여 사용하게 되어 비용은 줄이고 성능을 올릴 수 있습니다.


- S3 버킷 정책 및 액세스 제어 목록(ACL)으로 제한합니다. 대게 아래와 같이 전송 프로토콜을에 대해 TLS만 허용하고, S3 버킷에 액세스하는 주체를 CloudFront OAC 로 한정하도록 제한합니다.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "denyInsecureTransport",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "<your-s3-bucket-arn>/*",
                "<your-s3-bucket-arn>"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        },
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "<your-s3-bucket-arn>/*"
            "Condition": {
                "StringEquals": {
                    "aws:SourceArn": "<your-cloudfront-oac-arn>"
                }
            }
        }
    ]
}
```

![img_1.png](/assets%2Fimages%2F24q1%2Fimg_1.png)


<br>

## CloudFront 배포 및 S3 오리진 보안

먼저 Origin 컨텐츠 주체인 S3 버킷이 필요합니다. 이 버킷을 대상으로 CloudFront OAC (Origin Access Control)을 생성하고 CloudFront 를 배포할 수 있습니다.  

<br>

### CloudFront OAC 생성  

S3 버킷을 CloudFront만 액세스 할 수 있도록 CloudFront OAC 생성합니다.

![img_3.png](/assets%2Fimages%2F24q1%2Fimg_3.png)

<br>

### CloudFront Key Group 등록 

CloudFront Signed URL 또는 Signed Cookies를 통해 Origin을 액세스하는것을 제어할 할 수 있도록 Public Key를 등록하고 Key-Group 을 관리 합니다.  

Signed URL 또는 Signed Cookies 를 통해 클라이언트의 요청이 리소스에 대한 액세스 권한을 가진 사용자임을 식별할 수 있습니다.

<br> 

- Public Key 생성 및 Key Group 에 추가

RSA 알고리즘의 2048 비트로 CloudFront 액세스 키 `my-cf.pem` 를 다음과 같이 생성합니다.

```
openssl genrsa -out my-cf.pem 2048
```

생성된 `my-cf.pem` 로부터 `my-cf.pub` Public Key 를 추출하여 생성합니다.

```
openssl rsa -pubout -in my-cf.pem -out my-cf.pub
```

![img_5.png](/assets%2Fimages%2F24q1%2Fimg_5.png)


위와 같이 생성한 Public Key 를 `CloudFront > Public keys > Create public key` 를 통해 등록 합니다.

![img_6.png](/assets%2Fimages%2F24q1%2Fimg_6.png)


<br>


- Key Group 등록

![img_4.png](/assets%2Fimages%2F24q1%2Fimg_4.png)

`CloudFront > Key groups > Create key group` 을 통해 Key Group 을 등록 합니다. Public Keys 는 앞에서 등록한 Public Key 를 추가합니다.


<br>


## CloudFront 배포

컨텐츠 서비스를 제공한 S3 버킷을 Origin 으로 선택하여 CloudFront 배포합니다.  

S3 버킷 이름 기준으로 `<s3-bucket-name>.s3.<region>.amazonaws.com` Origin 도메인, CloudFront 이름이 자동으로 기입됩니다.   

`Origin access` 항목에서 앞에서 생성한 `CloudFront OAC`를 선택합니다.   

![img_4.png](/assets%2Fimages%2F24q1%2Fimg_7.png)


### CloudFront 의 주요 설정

- CloudFront 배포에서 CDN 서비스 범위를 지정하는 `Price class`를 선택합니다. 

CloudFront는 콘텐츠 배포를 위해 세 가지 `Price Class`을 제공합니다. 각 등급은 CloudFront 엣지 로케이션의 지리적 위치에 따라 다른 요금이 적용됩니다.  

`Price Class All`: 모든 CloudFront 엣지 로케이션에서 콘텐츠를 제공할 수 있습니다. 가장 높은 가격이지만 전 세계적으로 균일한 성능을 제공합니다.
`Price Class 200`: 북미, 남미, 유럽, 중동, 아프리카, 아시아, 호주 등 주요 대륙 전체를 커버하며 약 200개 이상의 엣지 로케이션에서 콘텐츠를 제공합니다.
`Price Class 100`: 북미, 유럽, 아시아의 일부 지역을 커버하며 약 100 여개의 엣지 로케이션에서 콘텐츠를 제공합니다.

범위가 클수록 컨텐츠 배포 영역이 넓기 때문에 전세계 인터넷사용자에게 그만큼 빠르게 서비스가 제공되지만 그만큼 비용이 올라갑니다. 


- CDN 서비스를 제공할 Public 도메인과 인증서를 선택 합니다.

CloudFront는 리전 구분이 없는 글로벌 서비스이므로 Public 도메인(예: 'myappservice.com')에 대한 ACM Certificate 는 us-east-1 버지니아 리전에 생성하여야 하며 인증서는 Verified 된 Issued 상태여야 합니다. 


![img_8.png](/assets%2Fimages%2F24q1%2Fimg_8.png)


- Behaviors 설정 

S3 버킷의 /uploads 경로에 대해 클라이언트가 직접적으로 액세스 되는것을 방지하고 signed-url 을 통해 객체를 접근할 수 있도록 S3 버킷의 특정 Prefix에 대해 Behaviors 설정할 수 있습니다. 

CloudFront의 Behavior로 `/uploads` 버킷 경로에 대해 캐싱 동작, 라우팅 룰, Viewer 제한 및 CORS와 같은 보안 정책 등을 다음과 같이 설정 합니다.

 
![img_9.png](/assets%2Fimages%2F24q1%2Fimg_9.png)


![img_10.png](/assets%2Fimages%2F24q1%2Fimg_10.png)


특히, `Viewer` 설정에서 `Viewer protocol policy` 라우팅 정책, `Allowed HTTP methods` 허용 메서드, `Restrict viewer access`를 통해 클라이언트의 액세스를 제한합니다. 

`Restrict viewer access` 의 `Trusted key groups (recommended)`에서 앞서 생성한 CloudFront Key Group 을 설정합니다. 

이로써 CloudFront 배포 설정 및 S3 Origin 을 안전하게 보안되도록 구성이 완료 되었습니다.


## CloudFront Signed URL 생성 및 Origin 객체 액세스 테스트

CloudFront 의 Signed URL 을 생성하려면 Public Key 의 `key-pair-id`, RSA 알고리즘으로 생성한 `private-key` 그리고 액세스할 S3 버킷의 객체에 대한 `url`(CloudFront 의 도메인 URL)이 필요 합니다.    


다음과 같이 [AWS CloudFront CLI](https://docs.aws.amazon.com/cli/latest/reference/cloudfront/sign.html) 명령을 통해 Signed URL을 생성할 수 있습니다.  

```
aws cloudfront sign --key-pair-id K2Z******** \
    --private-key file://<your-private-key.pem> \
    --date-less-than 2024-03-20 \
    --url "https://<yourcfhost>.cloudfront.net/uploads/AIM352-R-securely-build-generative-AI-with-bedrock.pdf"
```

위 명령의 실행 결과로 아래와 같은 `Signed URL`이 생성되고 `2024-03-20` 이전까지 클라이언트는 URL을 통해 Origin 객체를 액세스할수 있게 됩니다.   
```
https://d2owqjwir1q8d8.cloudfront.net/uploads/AIM352-R-securely-build-generative-AI-with-bedrock.pdf?Expires=1712016000&Signature=nrghUaxaGx9~2OAPIH9K0ud9LTsCI....&Key-Pair-Id=K2Z6XUBK23XBDH
```

![img_13.png](/assets%2Fimages%2F24q1%2Fimg_12.png)

<br>

##  Conclusion

이제 우리는 CloudFront 기반의 간단한 아키텍처 개선으로 악의적인 클라이언트의 요청으로부터 안전하게 S3 Origin 객체를 보호하고,
개발자가 `Signed URL`을 생성하기 위해 불필요한 사용자 AssumeRole로 전환하거나 코드에서 위함한 Credentials를 더이상 사용할 필요가 없게 되었습니다.