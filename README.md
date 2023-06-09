# ECS Fargate를 이용한 이미지 분류 구현하기

여기서는 [ecs-fargate-sagemaker-based-webservice](https://github.com/hijigoo/ecs-fargate-sagemaker-based-webservice)에서 SageMaker로 이미지 분류와 ECS Fargate로 Inference를 구현하는것을 정리하려고 합니다.

[Amazon ECS](https://aws.amazon.com/ko/ecs/)와 [AWS Fargate](https://aws.amazon.com/ko/fargate/)로 구현된 어플리케이션에서 SageMaker Endpoint를 이용하여 이미지 분류를 수행하는 아키텍처를 구현합니다.

## Architecture

## 구현

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



### SageMaker

[SageMaker Studio Console](https://ap-northeast-2.console.aws.amazon.com/sagemaker/home?region=ap-northeast-2#/studio-landing)에서 [Open Studio]를 선택합니다. Studio를 처음 사용한다면 [Launch Amazon SageMaker Studio](https://docs.aws.amazon.com/sagemaker/latest/dg/studio-launch.html)에 따라 Studio를 생성합니다.



