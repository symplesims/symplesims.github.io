---
layout: post
title: "Generating Secure CloudFront URLs with a AWS Lambda Function"

date:  2024-04-01 05:30:00 +0900
categories:
   - Security
   - Lambda
   - DevOps
   - HandsOn
---


S3 오리진 보호를 위한 CloudFront Signed URL 생성을 위한 AWS Lambda 구현 하기


<br>


## Signed URL 생성용 Lambda 구현의 배경 

앞서 `CloudFront Signed URL로 S3 오리진 컨텐츠 보호 하기`애서 CloudFront를 활용하여 S3 오리진 컨텐츠의 보호와 Signed URL을 통해 안전하게 클라이언트에게 컨텐츠를 전달하는 것을 보았습니다. 

하지만, `Signed URL`을 생성하기 위한 `Private Key` 정보를 클라이언트가 가지고 있어야 하며 Signed URL 생성 절차 역시 비교적 까다롭다고 할 수 있습니다. 

이런 기능은 공통적으로 처리하는 유틸리티성 모듈로 `AWS Lambda`로 구현하여 개발팀이 편리하게 이용할 수 있도록 돕는것이 효과적입니다. 

AWS Lambda 를 지원하는 런타임과 구현 방식은 많이 있습니다. 하지만 개인적으로 AWS Lambda 를 구현하기 위해 가장 높은 우선 순위는 무엇보다 안전한 런타임 환경이고 그에 못지않게 개발 생산성과 즉시성입니다.     

런타임 플랫폼을 Python으로 할 경우 boto3 라이브러리를 통해 안전하고 빠른 Lambda 구현을 돕고있습니다. 프로젝트 템플릿이 간단하고 쉬우며 테스트를 위한 구성 역시 Lambda 인점을 감안하면 비교적 단순합니다. 

Python 으로 구현하는 Lambda 프로젝트의 템플릿은 많이 확보되었기에, 이번엔 Nodejs 기반으로 도전해 보기로 했습니다.   

<br>

## 사전 준비 사항

시작하기에 앞서, nodejs에 대한 기술적 지식이 부족하므로 배경 지식 및 적합한 프레임워크와 프로젝트 템플릿을 조사 하였습니다.


### Node.js 개발 프레임워크 및 특징

여기에 나열된 각각의 프레임워크를 프로젝트를 구성하여 즉시성 및 난이도를 시험해 보았으며, 결론은 가장 대중적인 `Serverless Framework`가 러닝 커브가 낮고 구현 속도가 가장 빠르다고 생각되었습니다. (지극히 개인적 의견 입니다.)


<table>
<thead>
<tr>
    <td>프레임워크</td>
    <td>특징</td>
    <td>장점</td>
    <td>단점</td>
</tr>
</thead>
<tbody>
<tr>
    <td><a href="https://serverless.com/" target="_blank">Serverless Framework</a></td> 
    <td>Serverless 애플리케이션을 쉽게 개발, 배포 및 관리할 수 있는 가장 대중적인 오픈 소스 프레임워크입니다.</td>
    <td>
        - 서버리스 애플리케이션을 CLI를 통해 간편하게 배포합니다. <br/> 
        - AWS의 여러 서비스들 뿐만아니라, Azure, GCP과 같은 Public CSP 플랫폼을 지원합니다. <br/>
        - 많은 플러그인과 더불어 사용자 커스터마이징이 가능한 플러그인 확장성을 제공합니다.
    </td>
    <td>
        - 대규모 프로젝트에서 아키텍처를 관리하는 경우 다소 복잡합니다. <br/>
        - 서버리스 환경에서 디버깅이 다소 어려울 수 있습니다.
    </td>
</tr>
<tr>
    <td><a href="https://aws.amazon.com/serverless/sam/" target="_blank">SAM(AWS Serverless Application Model)</a></td>
    <td>
        AWS SAM(Serverless Application Model)은 서버리스 애플리케이션을 모델링하고 배포하기 위한 프레임워크로, AWS 서비스를 사용하여 서버리스 애플리케이션을 개발하고 배포하는 데 사용됩니다.
    </td>
    <td>
        - YAML 또는 JSON 형식의 간단한 템플릿을 사용하여 서버리스 애플리케이션을 모델링하고 배포합니다. <br/>
        - SAM CLI를 사용하면 로컬 환경에서 서버리스 애플리케이션을 개발하고 테스트할 수 있습니다. <br/>
        - CloudFormation과 통합되어 배포되며 개발자는 일관된 배포 프로세스를 유지하고 자동화할 수 있습니다.
    </td>
    <td>
        - AWS의 일부 서비스만 지원하며 일부 고급 기능을 사용하는경우 제한될 수 있습니다. <br/>
        - 처음 사용하는경우 SAM 템플릿의 구조에 대한 이해와 일정한 학습 곡선이 필요합니다. 
    </td>
</tr>
<tr>
    <td><a href="https://aws.amazon.com/cdk/" target="_blank">AWS CDK</a></td>
    <td>AWS CDK (AWS Cloud Development Kit)는 인프라 및 애플리케이션 리소스를 프로그래밍 방식으로 정의하고 배포하기 위한 오픈 소스 프레임워크입니다. 
        CDK를 사용하면 프로그래밍 언어를 통해 AWS 리소스를 정의할 수 있으며, 이를 통해 클라우드 리소스를 코드로 관리할 수 있습니다.</td>
    <td>
        - CDK로 정의된 인프라는 CloudFormation 템플릿으로 변환되어 배포됩니다. 
        - CDK를 사용하여 인프라를 정의하고 관리하는 동시에 CloudFormation의 기능을 활용합니다.</td>
    <td>
        - 효과적으로 사용하기 위해서는 AWS 서비스와 CloudFormation에 대한 이해가 필요합니다. <br/>
        - CDK를 사용하여 인프라를 코드에서 프로그래밍 언어의 오류가 발생할 수 있습니다. <br/>
        - 버전 호환성 문제가 발생할 수 있으며, 새로운 버전이 출시될 때 이전 버전과의 호환성을 유지가 어렵습니다.
    </td>
</tr>
<tr>
    <td><a href="https://claudiajs.com/" target="_blank">Claudia.js</a></td>
    <td>Claudia.js는 AWS Lambda 및 API Gateway를 간단하게 배포하고 관리하는 데 사용되는 도구입니다. 특히 Node.js 애플리케이션을 AWS Lambda 함수로 변환하고 API Gateway를 통해 액세스할 수 있는 RESTful API로 만드는 데 특화되어 있습니다. Claudia.js를 사용하면 명령줄 인터페이스(CLI)를 통해 몇 가지 간단한 명령만으로 애플리케이션을 배포하고 관리할 수 있습니다.</td>
    <td>
        - 복잡한 설정 없이 명령줄에서 간단한 명령만으로 AWS Lambda 함수 및 API Gateway를 배포할 수 있습니다. <br/>
        - AWS Lambda 함수 및 API Gateway를 자동으로 구성합니다. 따라서 개발자는 애플리케이션 코드에만 집중할 수 있습니다. <br/>
        - 코드 변경 사항을 즉시 반영하여 핫 리로드를 지원합니다.
    </td>
    <td>
        - AWS Lambda 및 API Gateway의 모든 기능을 지원하지는 않으며, Claudia API 가 감싸고 있으므로 자유도가 떨어집니다. 
        - 특정한 사용 사례에 초점을 맞추어 설계 되었으므로 범용적으로 사용하기에 제한이 많습니다. 
    </td>
</tr>
<tr>
    <td><a href="https://www.localstack.cloud/" target="_blank">Localstack</a></td>
    <td>
        로컬 환경에서 AWS 클라우드를 에뮬레이션하는 오픈 소스 도구로, 개발 및 테스트 시에 AWS 서비스를 로컬에서 실행하고 테스트할 수 있도록 도와줍니다.
    </td>
    <td>
        - 개발자는 실제 AWS 계정을 사용하지 않고도 AWS의 서비스를 로컬에서 실행하고 테스트할 수 있습니다. <br/>
        - 로컬에서 구현 및 테스트가 가능하므로 AWS 계정을 사용하는 것보다 비용이 저렴합니다.  <br/>
        - 서비스의 동작을 시뮬레이트하거나 가상의 데이터를 생성하여 다양한 시나리오를 테스트 할수 있습니다.
    </td>
    <td>
        - AWS의 모든 서비스를 완벽하게 모방하지는 않습니다. Pro 라이센스를  구독하면 더 많은 에뮬레이터를 지원합니다. <br/> 
        - 로컬에서 실행되는 AWS 서비스는 실제 AWS 환경과는 다를 수 있습니다. <br/>
        - 통합되는 서비스가가 많으면 LocalStack를 설정하고 구성하는 환경 또한 복잡하고 무겁게 동작합니다. 
    </td>
</tr>
</tbody>
</table>


<br/>

## Node.js 개발 환경 구축

MacOS 에서 개발하였으며 개발 환경의 Node 버전이 `v20.11.1` 이고, npm 버전은 `10.2.4` 입니다.  

MacOS 기반에서 Node.Js 개발 환경 구성은 [Node 패키지 매니저](https://symplesims.github.io/development/setup/macos/2021/12/02/setup-development-environment-on-macos.html#node)를 참고 하면 쉽게 구성할 수 있습니다. 

<br/>

### serverless 프레임워크 설치
```
npm install -g serverless

# 설치 버전 확인  
serverless --version
```

<br/>

### lambda 애플리케이션 프로젝트 템플릿  

- 프로젝트 디렉토리 생성

```
mkdir cloudfront-signedurl-lambda

cd cloudfront-signedurl-lambda
```

<br>

- Node.Js 프로젝트 초기화

```
npm init -y
```

<br>

- Node.Js 라이브러리를 설치합니다. 참고로 NPM 라이브러리는 [npmjs.com](https://www.npmjs.com/)에서 찾을수 있습니다.  

```
npm install date-fns
npm install @aws-sdk/client-ssm
npm install @aws-sdk/cloudfront-signer
npm install @types/aws-lambda

# 개발용 라이브러리
npm install -D esbuild
npm install -D @types/node
npm install -D @types/aws-lambda
```

<br/>

- package.json 명세   

Node.Js는 버전에 영향을 아주 많이 받으므로 "engines" 속성에 아래와 같이 node 및 npm 버전을 명시하는것이 좋습니다. 

```
{
  "name": "cloudfront-signedurl-lambda",
  "version": "1.0.0",
  "description": "",
  "engines": {
    "node": ">=v20.11.1",
    "npm": ">=10.2.4"
  },  
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@aws-sdk/client-ssm": "^3.540.0",
    "@aws-sdk/cloudfront-signer": "^3.541.0",
    "@types/aws-lambda": "^8.10.136",
    "date-fns": "^3.6.0"
  }
}
```


<br/>

## Lambda 아키텍처 개요  

<p style="text-align: center;">
    <img style="display: block; margin: auto;" src="https://raw.githubusercontent.com/symplesims/symplesims.github.io/main/assets/images/24q1/cfarch02.svg" />
</p>

- Backend API 애플리케이션은 Signed URL 생성을 Lambda 에게 요청하고, 생성된 signed url 을 클라이언트에게 전달 합니다.  
- 클라이언트는 signed url 을 통해 CloudFront에게 요청하고, CloudFront는 인증된 클라이언트가 요청한 Origin 객체를 내려주게 됩니다.  
- Lambda 는 S3, AWS Systems Manager의 Parameter Store, Secrets 복호화를 위한 KMS 관련 액세스 정책이 필요 합니다. 


<br/>

## Lambda 함수 구현

Lambda 함수 구현함에 있어서 아래의 주요 가이드라인을 계획하고 진행하였습니다. 

- Node.Js 기반으로 Signed URL 생성하는 람다 아키텍처에서 `Private Key`는 AWS Systems Manager 의 Parameter Store 의 보안 문자열을 통해 안전하게 관리 합니다.
- 불필요한 구현을 하지 않고 검증된 라이브러리를 활용하여 개발 생산성을 높입니다. 
- 구성 정보와 같은 상수값을 하드 코딩 하지 않고 환경 변수를 통해 참조하도록 함으로써 유연성을 높입니다.
- 하나의 책임 원칙(Single Responsibility Principle)으로 컴포넌트화 재사용성과 유연한 결합을 지향합니다.  
- typescript 기반으로 구현합니다. 

<br/>

### CloudfrontSignedHandler 클래스의 구현

`CloudfrontSignedHandler` 클래스는 `getSignedUrl(s3ObjectPath: string, expireDays?: number)` 메서드를 통해 signed url 을 생성합니다.  
input 파라미터는 `s3ObjectPath` S3버킷 상대경로와, expireDays 만료일입니다. 

[src/utils/cloudfront-signed-handler.ts]
```typescript
import {getSignedCookies, getSignedUrl} from '@aws-sdk/cloudfront-signer';
import {CloudfrontSignedCookiesOutput} from "@aws-sdk/cloudfront-signer/dist-types/sign";
import {format} from "date-fns";

class CloudfrontSignedHandler {
    private static instance: CloudfrontSignedHandler;

    private readonly cfDomainUrl;
    private readonly privateKey;
    private readonly keyPairId;

    private constructor(cfDomainUrl: string, keypairId: string, privateKeyString: string) {
        this.cfDomainUrl = cfDomainUrl;
        this.privateKey = privateKeyString;
        this.keyPairId = keypairId;
    }

    public static getInstance(cfDomainUrl: string, keypairId: string, privateKeyString: string): CloudfrontSignedHandler {
        if (!CloudfrontSignedHandler.instance) {
            CloudfrontSignedHandler.instance = new CloudfrontSignedHandler(cfDomainUrl, keypairId, privateKeyString);
        }
        return CloudfrontSignedHandler.instance;
    }

    private getExpireDay(expireDays?: number): string {
        const expireDate = new Date()
        expireDate.setTime(Date.now() + ((expireDays || 7) * 24 * 60 * 60 * 1000));
        const expireDay = format(expireDate, 'yyyy-MM-dd');
        console.log('expireDate: ', expireDate.getTime())
        console.log('expireDay: ', expireDay)
        return expireDay;
    }

    public getSignedUrl(s3ObjectPath: string, expireDays?: number): { signedUrl: string, dateLessThan: string } {
        const url = `${this.cfDomainUrl}/${s3ObjectPath}`;
        const keyPairId = this.keyPairId;
        const privateKey = this.privateKey;
        const dateLessThan = this.getExpireDay(expireDays);
        const signedUrl = getSignedUrl({
            url,
            privateKey,
            keyPairId,
            dateLessThan,
        });
        return {signedUrl, dateLessThan};
    }

}

export {CloudfrontSignedHandler}
```

<br/>

### SsmHandler 클래스의 구현

`SsmHandler` 클래스는 `SSM Parameter Store` 로부터 Key 경로에 해당하는 암호값을 가져옵니다.  

[src/utils/ssm-handler.ts]
```typescript
import {GetParameterCommand, SSMClient} from '@aws-sdk/client-ssm';

class SsmHandler {
    private static instance: SsmHandler;

    private ssmClient: SSMClient;

    private constructor(region?: string) {
        let options: { region?: string } = {};
        if (region) {
            options = {region: region};
        }
        this.ssmClient = new SSMClient(options);
    }

    public static getInstance(region?: string): SsmHandler {
        if (!SsmHandler.instance) {
            SsmHandler.instance = new SsmHandler(region);
        }
        return SsmHandler.instance;
    }

    async getParameter(name: string, region?: string, withDecryption?: boolean): Promise<string> {
        const requestParam = {
            Name: name,
            WithDecryption: withDecryption ?? true
        }
        const requestCommand = new GetParameterCommand(requestParam)
        return new Promise((resolve, reject) => {
            this.ssmClient.send(requestCommand, function (err: Error, data: any) {
                if (err) {
                    return reject(err)
                }
                if (!data?.Parameter) {
                    return reject(new Error('not found'))
                }
                resolve(data.Parameter.Value)
            })
        })
    }
}

export {SsmHandler}
```

<br/>


### Lambda Endpoint 인 핸들러 구현

AWS 람다가 내부적으로 호출되는 핸들러 입니다. 여기엔 서비스 구성에 필요한 환경 정보를 참조 합니다.

- CLOUDFRONT_DOMAIN: CloudFront 도메인 입니다.
- KEY_PAIR_ID: CloudFront Public Key 에 대한 KEY_PAIR_ID 아이디 입니다.
- SSM_PRIVATE_KEY: Private Key 를 보관하는 SSM 파라미터 스토어 경로입니다.

[index.ts]
```typescript
import {Context, Handler} from 'aws-lambda';
import {SsmHandler} from './src/utils/ssm-handler';
import {CloudfrontSignedHandler} from "./src/utils/cloudfront-signed-handler";

const cfDomainUrl = process.env.CLOUDFRONT_DOMAIN || "";
const keyPairId = process.env.KEY_PAIR_ID || "";
const ssmParameterKey = process.env.SSM_PRIVATE_KEY || "";

const handler: Handler = async (event: any, context: Context) => {
    try {
        const {s3ObjectPath, expireDays} = event;
        const privateKey = await SsmHandler.getInstance().getParameter(ssmParameterKey);
        const {
            signedUrl,
            dateLessThan
        } = CloudfrontSignedHandler.getInstance(cfDomainUrl, keyPairId, privateKey).getSignedUrl(s3ObjectPath, parseInt(expireDays) || 7)

        return {
            statusCode: 200,
            body: JSON.stringify({signedUrl: signedUrl, expireDay: dateLessThan}, null, 2),
        };
    } catch (error) {
        return {
            statusCode: 400,
            body: JSON.stringify({message: error})
        };
    }
};

export {handler};
```

<br/>
<br/>


## Lambda 배포 및 테스트
위와 같이 주요한 Node.Js 클래스와 핸들러 구현이 모두 완료되었고 이제 AWS 클라우드 리소스를 정의하고 배포하여 정상적으로 동작하는지 확인해 보도록 합니다.

### serverless.yml 정의

Node.Js 기반 Lambda 애플리케이션을 AWS 클라우드에 배포하기 위해 `serverless.yml`을 정의합니다. 

`package:`  - 빌드 결과물을 정의 합니다. '!' 는 제외할 파일 또는 디렉토리를 의미합니다.  
`provider:` - AWS 클라우드를 액세스 하는 정보와 클라우드상에 구성할 리소스 정보를 기술합니다. Provider 를 통해 CSP 클라우드를 액세스하고 배포하게 됩니다. 배포를 위한 로컬 컴퓨터는 AWS Profile `dev`를 통해 CLI 로 액세스할 수 있어야 합니다. 
`functions:` - 애플리케이션 (Lambda) 정보를 기술합니다. 람다 이름, Entrypoint, 환경 정보 등을 기술하게 됩니다. 

참고로 "<변수값>" 은 여러분의 클라우드 환경에 맞는 값으로 대체되어야 합니다. 

```yaml
service: cloudfront-signed-lambda
frameworkVersion: '3'

package:
  individually: true
  patterns:
    - 'index.ts'
    - 'src/**/*.ts'
    - 'types/**/*.ts'
    - 'node_modules/node-fetch/**'
    - '!tmp/**'
    - '!dist/**'
    - '!target/**'
    - '!.git/**'

provider:
  name: aws
  stage: dev
  region: ap-northeast-2
  profile: dev
  runtime: nodejs20.x
  architecture: arm64
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - ssm:GetParameter
          Resource: "<your-parameter-store-private-key-arn>"
        - Effect: Allow
          Action:
            - kms:Decrypt
          Resource: "<your-kms-arn>"
        - Effect: Allow
          Action:
            - s3:GetObject
          Resource: "<your-s3-bucket-arn>/*"

functions:
  cloudfrontSignedUrl:
    handler: index.handler
    timeout: 60
    environment:
      CLOUDFRONT_DOMAIN: "<your-cloudfront-domain-url>"
      SSM_PRIVATE_KEY: "<your-parameter-store-private-key-path>"
      KEY_PAIR_ID: "<your-cloudfront-public-key-id>"
      EXPIRATION_DAYS: "5"
```

참고로, serverless.yaml 은 CloudFormation 템플릿으로 변환되어 CloudFormation Stack 으로 배포가 됩니다. 

<br/>

### Lambda 배포 

람다 애플리케이션 배포 및 삭제는 serverless CLI 명령을 통해 즉시 진행될 수 있습니다. 

```
serverless deploy  
```

<br/>

### Lambda 테스트

Lambda가 정상적으로 배포되면 아래와 같은 명령으로 signed-url 을 생성할 수 있습니다.

```
aws lambda invoke --function-name cloudfront-signedurl-lambda-dev-cloudfrontSignedUrl \
  --cli-binary-format raw-in-base64-out \
  --payload '{"s3ObjectPath":"uploads/uploaded-report.pdf", "expireDays":"7"}' \
  output.json
```
위 명령은 S3 버킷에서 `uploads/uploaded-report.pdf` 객체에 대해 sigend-url 을 생성하며 만료일을 현재일 기준 7일 이하로 설정하는 명령입니다.

<br/>

### Lambda 제거  

```
serverless remove  
```

<br>

## 컴퓨팅환경의 무결성을 위한 Dockerize

AWS Lambda 는 이미지 타입으로 배포할 수 있습니다. 이미지 타입은 런타임 환경을 최적화 할 수 있습니다.   
또한 동일한 런타임을 보장하며, 일관된 배포 환경을 유지하고 문제가 발생될 경우 이전 이미지로의 롤백 역시 용이합니다.  

<br>

### Dockerize

아래와 같이 ARM64 기반의 AWS Manged 베이스라인 이미지를 참조하는 `Dockerfile`을 작성합니다. 

[Dockerfile]
```
FROM public.ecr.aws/lambda/nodejs:20.2023.12.06.12-arm64
LABEL author="symplesims@gmail.com"

COPY package.json index.ts src tsconfig.json ./
COPY ./src ./src/

RUN npm install
RUN npm run build

ADD index.ts ${LAMBDA_TASK_ROOT}/
ADD src/ ${LAMBDA_TASK_ROOT}/src/

CMD [ "index.handler" ]
```

<br>

### Troubleshooting 

불행하게도 현재 시점에서 이미지 기반 Lambda 런타임은 typescript 를 지원하지 않고 있습니다.    
esbuild 플러그인을 통해 typescirpt 를 javascript 형식으로 변환하여 배포를 해야만 합니다. 

- `npm run build` 명령을 실행할 경우 esbuild 로 패키징되도록 `package.json` 파일에 아래 코드를 추가 합니다.    

[package.json]
```
  "scripts": {
    "build": "esbuild index.ts --bundle --minify --sourcemap --platform=node --target=es2020 --outfile=dist/index.js"
  },
```

<br/>

- esbuild 컴파일 옵션을 `tsconfig.json` 파일로 정의 합니다.

```
{
  "compilerOptions": {
    "target": "es2016",
    /* Modules */
    "module": "commonjs",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  }
}
```

<br/>

##  Conclusion

이번 CloudFront Signed URL 생성을 전담하는 Lambda 구현을 통해 보안성과 편의성을 모두 높일 수 있게 되었습니다.

S3 오리진 콘텐츠에 대한 액세스 제어를 강화함으로써 데이터 보호 수준을 향상시켰고, 동시에 Lambda 기반의 서버리스 아키텍처로 운영의 복잡성을 줄일 수 있었습니다.

특히 이 공통 모듈을 활용하면 개발팀에서 CloudFront Signed URL 생성 로직을 직접 구현할 필요 없이 모듈을 재사용할 수 있어 생산성 향상에도 기여할 것으로 기대됩니다. 

이러한 방식으로 클라우드 기반 모범 사례를 도입하여 보안, 확장성, 비용 효율성 등 다양한 측면에서 애플리케이션의 품질을 지속적으로 개선해 나가는 한가지 사례로 결론을 맺고자 합니다.

<br/>

## References

- [aws-signedurl-lambda](https://github.com/simplydemo/aws-signedurl-lambda.git)