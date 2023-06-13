# ECS Fargate를 이용한 이미지 분류 구현하기

여기서는 [ecs-fargate-sagemaker-based-webservice](https://github.com/hijigoo/ecs-fargate-sagemaker-based-webservice)에서 SageMaker로 이미지 분류와 ECS Fargate로 Inference를 구현하는것을 정리하려고 합니다.

## Architecture

[Amazon ECS](https://aws.amazon.com/ko/ecs/)와 [AWS Fargate](https://aws.amazon.com/ko/fargate/)로 구현된 어플리케이션에서 SageMaker Endpoint를 이용하여 이미지 분류를 수행하는 아키텍처를 구현합니다. 샹세한 Architecture는 [ecs-fargate-sagemaker-based-webservice](https://github.com/hijigoo/ecs-fargate-sagemaker-based-webservice)를 참조하여 아래와 같이 간단히 그렸습니다.

![image](https://github.com/kyopark2014/image-classification-using-aws-fargate-for-ecs/assets/52392004/bf2c14ec-3b81-41e5-9b3e-cf34ad1e51f1)


## Training

## 구현

### ECS Terminalogy

[AWS EC2 on ECS vs Fargate](https://www.youtube.com/watch?v=DVrGXjjkpig)와 같이 ECS는 AWS ECS on ECS와 AWS Fargate for ECS로 구성할 수 있습니다. 여기서는 ECS Terminalogy와 관련된 CDK 코드를 설명합니다. 일반적으로 EC2로 ECS를 구성하면 서비스에 대한 컨트롤에 유리하므로 예측되는 Job을 High utilization 할 수 있으며, Fargate는 구성(Setup) 및 유지보수(Maintenance)가 유리하고 Adhoc Job들을 유연(Flexibility)하게 관리할 수 있습니다. 

- Task : ECS의 Runtime instance로서 ECS를 구성하는 가장 기본이 되는 block입니다.
- [Task Definitions](./cdk-image-classification/README.md#task-definition-web) : Task를 위한 Template로 Docker 이미지를 포함한 Memory/CPU 요구사항을 설정합니다.
- Container(EC2 only) : Task를 구동하는 가상의 인스턴스입니다.
- [Cluster](./cdk-image-classification/README.md#cluster) : Task들을 실행하는 컨테이너(EC2) 또는 Task(Fargate)의 그룹(Group)입니다.
- [Service](./cdk-image-classification/README.md#task-definition-web) : Task를 실행하고 관리하는 시스템입니다.


아래 그림은 ECS를 EC2와 Fargate로 구성하였을때의 차이점을 보여줍니다. EC2로 ECS를 구성시에는 컨테이너 1개에 여러개의 Task를 가질수 있고, 이것으로 Autoscaling group으로 묶여진 Cluster를 구성하는데, Fargate는 Task들로만 구성된 Cluster를 구현할 수 있습니다.

<img src="https://github.com/kyopark2014/image-classification-using-ecs-fargate/assets/52392004/623b9cc2-0760-42dc-a9ef-3e8b534beb48" width="600">


### AWS Cloud9 환경 준비

배포의 편의를 위하여 [AWS Cloud](https://aws.amazon.com/ko/cloud9/)을 이용하여 설치를 진행합니다.

[Cloud9 Console](https://ap-northeast-2.console.aws.amazon.com/cloud9control/home?region=ap-northeast-2#/create)에 접속하여 [Create environment] 이름으로 “AIWebApplication”를 입력하고, EC2 instance는 편의상 “m5.large”를 선택합니다. 나머지는 기본값을 유지하고, 하단으로 스크롤하여 [Create]를 선택합니다.

![noname](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/85933efa-3e9e-458b-a9cc-a1ca0ba5bfa9)

[Environment]에서 “AIWebApplication”를 [Open]한 후에 아래와 같이 터미널을 실행합니다.

![image](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/272281b0-a99d-42ff-b771-2e69ba986a4f)

Cloud9 용량을 확장합니다.

```java
wget https://raw.githubusercontent.com/kyopark2014/technical-summary/main/resize.sh
chmod a+rx resize.sh
./resize.sh 100
```

### CDK로 설치하기

소스를 다운로드 합니다. (아래 경로 변경 필요함)

```java
git clone https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice
```

관련된 라이브러리를 설치합니다.

```java
cd ecs-fargate-sagemaker-based-webservice/cdk-ai-wep-application && npm install
```

Account ID를 확인합니다.

```java
aws sts get-caller-identity --query Account --output text
```

아래와 같이 bootstrap을 수행합니다. 여기서 “account-id”는 상기 명령어로 확인한 12자리의 Account ID입니다. bootstrap 1회만 수행하면 되므로, 기존에 cdk를 사용하고 있었다면 bootstrap은 건너뛰어도 됩니다.

```java
cdk bootstrap aws://account-id/ap-northeast-2
```

아래와 같이 설치합니다.
 
```java
cdk deploy
```

셜치과 완료가 되면 아래와 같이 CDK의 Output에서 "WebPageURL"와 "WasAlbUrl"을 알 수 있습니다.

![image](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/9c308c78-618e-488e-8060-c98f684ce121)

아래와 같이 Cloud9의 왼쪽 탐색기에서 "image-classification-using-ecs-fargate/web/app/router.js" 파일을 아래와 같이 BASE_URL을 WasAlbUrl로 업데이트합니다.

![noname](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/b8af281b-5f7e-4235-a17e-aeb5d102cdc4)

저장후에 Cloud9의 터미널로 가서 아래와 같이 재설치를 합니다.

```java
cdk deploy
```

완료가 되면, "WebPageURL"로 접속하여 아래와 같은 화면이 나오는지 확인합니다. 

![image](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/f4c0f8e4-a0fb-48c5-8d6b-8c2055cd64b6)

이때 좌측 상단의 [WAS 접속 확인 페이지]를 선택하여, 아래와 같이 [WAS 접속 확인]을 선택합니다.
<img src="https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/2d82ee90-d8d8-41d8-93d1-a22244485f60" width="600">

정상적으로 설치가 되어서 WAS와 잘 연결되었다면 아래와 같은 메시지가 나옵니다.

![image](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/assets/52392004/f2f0b5db-6038-4f3d-91cd-8cab823beea8)




### SageMaker

[SageMaker Studio Console](https://ap-northeast-2.console.aws.amazon.com/sagemaker/home?region=ap-northeast-2#/studio-landing)에서 [Open Studio]를 선택합니다. Studio를 처음 사용한다면 [Launch Amazon SageMaker Studio](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-launch.html)에 따라 Studio를 생성합니다.


## 리소스 정리하기

Cloud9의 터미널에 접속하여 아래와 같이 설치한 인프라들을 삭제합니다.

```java
cdk destroy
```


## Reference

[AWS EC2 on ECS vs Fargate](https://www.youtube.com/watch?v=DVrGXjjkpig)

[AWS Fargate와 Amazon ECS를 활용한 CI/CD 모범사례](https://www.youtube.com/watch?v=ZRPdVY5lFBM)
