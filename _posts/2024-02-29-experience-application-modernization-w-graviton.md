---
layout: post
title: "AWS Graviton Migration Experience"

date:  2024-02-29 15:00:00 +0900
categories:
   - AWS
   - Modernization
---


AWS Graviton 마이그레이션 경험 


## Graviton 프로세서로의 마이그레이션 결정 배경과 과정 소개

제가 운영중인 서비스의 워크로드는 10개 이상의 스테이지(AWS 아카운트), 5개 이상의 리전에서 다양한 AWS 서비스들를 통합하여 고객에게 SaaS 서비스를 제공하고 있습니다.

여러 스테이지 중 가장 큰 서비스의 워크로드 월 비용이 대략 45,691 USD 가 발생중이였고 이중 RDS 가 50% 가량 차지하고 있었습니다.  

비용절감 및 성능을 별다른 노력 없이 가장 빠르게 개선할 수 있는 방법 중 하나가 컴퓨팅 인스턴스의 타입을 적절하게 변경(컴퓨팅 인스턴스 현대화)하는 것이므로 빠르게 의사결정 하였습니다.

참고로, [Amazon AWS 서울 리전에서 Graviton 인스턴스의 타입 지원](https://aws.amazon.com/ko/about-aws/whats-new/2023/05/amazon-rds-m6g-r6g-database-instances-four-regions/)은 23년 5월 부터 입니다.  


<br/>

## 마이그레이션 전/후 성능 비교 및 비용 절감 효과 설명

작년 2023년 5월 기준의 R5 인스턴스 타입과 현재 2월 기준의 R6G Graviton2 타입으로 변경 이후의 변화를 아래 그래프로 확인할 수 있습니다. 

- MRR: 2023. 5. 

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="/assets/images/24q1/img_13.png" />
</p>


- MRR: 2024. 1.

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="/assets/images/24q1/img_14.png" />
</p>

또한, 전체 절감액이 6,317 USD 이고, 이중 RDS가 차지하는 비중이 6,046 USD 으로 절감액의 95%가 RDS 가 차지하고 있는 것을 확인할 수 있습니다.


- Cost reduction trend change from 2023. 5. to 2024. 2.

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="/assets/images/24q1/img_15.png" />
</p>


위 비용절감 트랜드 차트를 보면 2023. 5. 이후로 서비스 워크로드는 지속적으로 증가했음에도 불구하고 2023.8 월을 기점으로 RDS 비용이 지속적으로 감소하고 있음이 확인됩니다.  

Graviton 2 인스턴스타입 조정 뿐만 아니라 RI/SP 적용으로 현재 이와 같은 절감을 실현할 수 있었습니다. 

<br/>


## Graviton2 마이그레이션 과정

Aurora Mysql 클러스터, EC2 인스턴스, ECS 컨테이너를 대상으로 Graviton 인스턴스 타입으로 마이그레이션 하였고, 그 과정은 아래와 같습니다. 

<br/>

### Aurora Mysql 클러스터 마이그레이션


AWS Aurora RDS 인스턴스를 대상으로 Graviton 2 의 마이그레이션은 Read Nod 를 추가하는 방식으로 진행하면 다운타임이 거의 없이 안전하게 전환할 수 있습니다.

RDS 클러스터를 선택하고 `Action > Add reader`메뉴를 통해 Reader 노드를 추가할 수 있습니다. 이 때 변경할 Graviton2 타입을 지정하면 됩니다. 

새롭게 추가된 Graviton 2 타입의 Reader 노드가 사용가능한 상태가 되면 Write 노드를 제거하기만 하면 됩니다.

참고로, RDS 접속을 위한 Endpoint는 클러스터 이름 및 인스턴스의 이름에 의해 결정되므로 클러스터 Endpoint 는 변경되지 않습니다.

혹시라고 애플리케이션이 Write 인스턴스의 Endpoint 를 직접 연결한 경우를 대비하여 새롭게 승격한 Reader 노드의 이름을 기존 Writer 인스턴스 이름으로 변경하는 것을 권고합니다.

<br/> 

### 마이그레이션 이전에 다음을 고려해야 합니다.


- RDS 엔진(mysql, postgresql) 버전이 Graviton 2를 지원하는지 확인 합니다.
- 별도의 RDS 컴플라이언스 툴이 있는지 사전에 확인이 필요합니다. RDS 보안 관리툴은 RDS 인스턴스의 아이피를 등록하여 통제하는 경우가 있습니다. 이 경우 RDS 인스턴스의 아이피가 변경되므로 사전에 보안 관리툴의 설정을 염두해 두는것이 좋습니다.


<br/>


## EC2 인스턴스 마이그레이션

사용중인 대부분의 EC2 애플리케이션은 Java로 구현되었습니다.  

Java 애플리케이션 은 JNI만 없다면 Graviton을 완전히 지원합니다. 그러므로, 애플리케이션 및 Dependencies Jar 파일에서 JNI(Java Native Interface)를 사용하는지 먼저 식별합니다. 

다음 명령으로 쉽게 식별할 수 있습니다. 

```
jar tf app.jar | grep .so
```

별도의 JNI 를 사용하는 애플리케이션이 없었고, Graviton 기반의 AWS Managed AMI를 선택하여 인스턴스 생성 및 CICD 파이프라인으로 앱을 빌드하여 배포 합니다. 


<br/>


## 컨테이너 마이그레이션 과정

AWS 클라우드를 이용중이므로 Java, Python, Node을 위한 Docker Base 이미지를 AWS Managed 이미지를 사용하여 Dockerize 합니다.

[gallery.ecr.aws](https://gallery.ecr.aws/search?architecture=ARM+64&operatingSystems=Linux&popularRegistries=amazon&verified=verified) AWS ECR 저장소를 통해 플랫폼별 Verified된 이미지를 찾을 수 있습니다. 

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="/assets/images/24q1/img_16.png">
</p>


`Dockefile` 의 Base 이미지를 따라 구성되므로 특별이 문제가 없었습니다.  

아래는 Builder 서버가 `x86_64`인 환경에서 ARM64 기반의 Docker 이미지를 빌드하기 위해 Qemu 에뮬레이터를 이용하여 빌드한 샘플 입니다.     

```
FROM multiarch/qemu-user-static:x86_64-aarch64 as qemu
FROM public.ecr.aws/lambda/python:3.11.2023.07.13.17-arm64

COPY --from=qemu /usr/bin/qemu-* /usr/bin

ADD *.py $LAMBDA_TASK_ROOT
ADD requirements.txt  .

RUN pip --no-cache-dir install -r ${LAMBDA_TASK_ROOT}/requirements.txt --target ${LAMBDA_TASK_ROOT}

CMD [ "app.handler" ]
```

참고로, Python 의 경우 실제 동작하는 모듈을 C/C++ 로 제작되었습니다. 그러므로 `requirements.txt`과 같이 정의된 Dependency 모듈들이 ARM64를 지원하는지 사전에 체크해야 합니다.  

<br/>

이와같은 절차로 Graviton 프로세서로 마이그레이션이 완료되었고 효과 도한 확실하게 증명되었습니다.

더 나아가 ARM 기반 프로세스와 이점, 그리고 몇가지 벤치마크와 제약사항을 다루어 보려고합니다.


<br/>
<br/>

## ARM 기반 프로세서란?

ARM(Advanced RISC Machine)은 마이크로프로세서 아키텍처로 약자로 RISC(Reduced Instruction Set Computing)는 아키텍처의 한 유형입니다.
RISC 아키텍처는 프로세서의 명령어 세트를 단순화하여 더 적은 수의 트랜지스터로 더 빠르게 명령을 실행할 수 있도록 해줍니다.  

<br/>

### RISC 아키텍처 이점은 다음과 같습니다.

- x86 기반 CPU의 성능에 필적하는 많은 최신 ARM 기반 CPU를 갖춘 고성능.
- ARM 프로세서는 대게 x86 프로세서보다 저렴하기 때문에 비용 효율성이 뛰어납니다.
- 전력 소비가 낮아 모바일 장치 및 기타 배터리 구동 애플리케이션에 매우 적합합니다.
- 유연성과 사용자 정의, 모듈식 설계를 통해 다양한 코어 수, 캐시 크기 및 기타 기능을 갖춘 프로세서를 더 쉽게 만들 수 있습니다.

이러한 배경으로 AWS 에서 자체 개발한 ARM 기반 프로세서로  `AWS Graviton` 이 출시되었습니다.

ARM 프로세서는 대부분의 AWS 서비스에서 지원되고 있으며, 이제 스마트폰, 태블릿과 같은 모바일 장치에 널리 사용될 뿐만아니라, 임베디드 시스템, IoT 장치, 서버 및 데이터 센터에서도 점점 더 많이 사용되고 있습니다.

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="https://upload.wikimedia.org/wikipedia/commons/2/2b/ARMCortexA57A53.jpg" width="50%" />
    [이미지 출처: wikimedia] 
</p>

<br/>

## AWS Griviton 소개


<p style="text-align: center;">
    <a href="https://youtu.be/Fvh4djznuuM" target="_blank"><img style="display: block; margin: auto;" src="https://d2908q01vomqb2.cloudfront.net/f1f836cb4ea6efb2a0b1b99f41ad8b103eff4b59/2021/04/10/Site-Merch_Graviton_SocialMedia_2.jpg" width="80%"/></a>
</p>


AWS Graviton 프로세서는 Amazon Web Services(AWS) 인스턴스에서 사용하도록 특별히 설계된 ARM 기반 프로세서 제품군입니다. 
이 제품은 ARM의 Neoverse N1 아키텍처를 기반으로 한 맞춤형 설계를 사용하여 제작되었으며, 이는 다른 ARM 기반 프로세서와 차별화되는 몇 가지 고유한 기능과 기능을 제공합니다. 
AWS Graviton 프로세서는 Amazon EC2 및 EC2 기반 PaaS에서 실행되는 클라우드 워크로드에 최고의 가격 대비 성능을 제공하도록 AWS에서 설계했습니다.

AWS는 범용 워크로드를 위해 Graviton 프로세서를 주력하고 있고, 23년 작년 re:Invent 에서 AWS Graviton 4를 소개하기도 했습니다.  

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="https://fuse.wikichip.org/wp-content/uploads/2023/12/graviton4_keynote_header.png" />
    [이미지 출처: 2023 re:Invent] 
</p>


<br/>

## AWS Graviton과 x86 프로세서의 성능 비교


Graviton과 Intel 프로세서는 서로 다른 영역에서 다른 목적으로 제 역할을 하고 있습니다.

- Graviton은 높은 수준의 병렬 처리가 필요한 클라우드 워크로드에 최적화되어 있는 반면, Intel은 범용적 애플리케이션을 위해 설계되었습니다. 
- Graviton은 Intel보다 적은 전력을 사용하고 일반적으로 저렴하지만 Intel은 더 넓은 소프트웨어 지원을 제공합니다. 

SW 호환성을 중점으로 둔다면 Intel을, 저 전력으로 병렬처리 기반의 고속의 SW 를 위해선 Graviton을 선택할 수 있습니다. 

- [MySQL on x86 vs ARM](https://mysqlonarm.github.io/MySQL-on-x86-vs-ARM/)
- [Performance Analysis for Arm vs x86 CPUs in the Cloud](https://www.infoq.com/articles/arm-vs-x86-cloud-performance/)
- [AWS Graviton2: Arm Brings Better Price-Performance than Intel](https://resources.scylladb.com/cost-efficiency/aws-graviton2-arm-brings-better-price-performance-than-intel)

이들 벤치마크 결과는 대부분 아래와 같은 결론에 도달합니다. 

- 전반적으로 Graviton2는 Intel x86 CPU에 비해 더 나은 비용/성능을 제공합니다.
- Graviton2는 네이티브 바이너리를 실행할 때 Intel에 비해 상당한 성능 향상을 보여주었습니다.
- Graviton2와 Intel 간의 Node.js 및 SSVM 성능 비교는 혼합되어 있습니다. 하지만 Graviton2 인스턴스가 24% 저렴하다는 점을 고려하면 비용 대비 성능면에서 앞서 있습니다

<br/>

## AWS Graviton 마이그레이션 전략

### 성능 최적화 전략

- 가장쉬운 방법은 AWS 클라우드라면 Arm 아키텍처에 최적화된 AWS Managed AMI(Amazon Linux 2 등)를 사용하는 것입니다.
- 애플리케이션 SW가 Functional 또는 React 와 같은 병렬처리 기반인 경우 성능이 극대화 됩니다.
- Java 의 경우 JDK11 이상에서 64비트 ARM에 대한 다양한 최적화를 지원하고 있습니다. JDK 8 기반에선 오히려 성능저하가 확인되는 경우도 있습니다. 
 

사실 Graviton 프로세서의 성능을 최대한 발휘하기 위해서는 운영체제(OS)와 소프트웨어(SW)가 Arm 아키텍처를 완벽히 지원해야 하며, 구체적으로 `Arm64 아키텍처용 컴파일된 바이너리`, `최적화된 커널 및 라이브러리`, `Arm 네이티브 명령어 셋 활용`, `Arm 친화적인 컴파일러 플래그 사용`, `최신 드라이버/펌웨어 업데이트` 과 같은 고려사항이 있습니다.

<br/>

### 마이그레이션 Tip

Graviton 마이그레이션에서 컴퓨팅 인스턴스를 위한 애플리케이션의 Scale In-Out, Blue/Green 과 같은 탄력적인 환경에 대응하는 것이 무엇보다 중요합니다.

이를 위해 Immutable 전략을 추천합니다. 

- 운영 단순화: 컴퓨팅 인스턴스에 대해 구성 변경없이 새로운 인스턴스를 교체하는것이 절차가 단순해집니다.
- 일관성 보장: 항상 동일한 상태의 Immutable 이미지를 사용하므로 일관성이 유지됩니다.
- 롤백 용이성: 장애 또는 신규 애플리케이션 배포에서 문제가 발생하는 경우 쉽게 롤백 가능합니다.

<br/>

## Conclusion

요약하자면, AWS Graviton 인스턴스로의 마이그레이션은 애플리케이션 성능을 크게 향상시키는 동시에 운영 비용을 대폭 절감할 수 있는 기회였습니다. 

사전 준비와 단계적 접근을 통해 마이그레이션 프로세스 자체도 생각보다 수월하게 진행할 수 있었습니다.

특히 Graviton의 높은 가격 대비 성능 덕분에 상당한 비용 절감 효과를 누렸으며, 이는 향후 클라우드 리소스 활용에 있어 더욱 유연성을 제공할 것으로 기대됩니다. 

이번 성공적인 Graviton 마이그레이션 경험을 통해 AWS 최신 기술을 리서치하고 활용하는 데 있어 보다 적극적인 행동변화도 가져왔습니다.

앞으로도 지속적으로 새로운 클라우드 네이티브 기술을 찾고 변화하는 트렌드에 발맞추어 애플리케이션의 민첩성, 비용 효율성, 보안성을 지속적으로 개선해 나가는 것으로 결론을 맺고자 합니다. 

<br/>

## References

- [Effortless migration - Is your app already Graviton-ready](https://www.youtube.com/watch?v=aa_FjqYCJpY)
- [Enable up to 40% better price-performance with AWS Graviton2 based Amazon EC2 instances](https://pages.awscloud.com/rs/112-TZM-766/images/2020_0501-CMP_Slide-Deck.pdf) 
- [Kafka Benchmarking on AWS Graviton2, Graviton3 and AMD](https://thenewstack.io/kafka-benchmarking-on-aws-graviton2-graviton3-and-amd/) 
- [Performance Runbook for Graviton](https://github.com/aws/aws-graviton-getting-started/tree/main/perfrunbook/utilities)
