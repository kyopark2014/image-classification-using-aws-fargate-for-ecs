# CDK를 이용한 이미지 분류 인프라 생성

아래와 같이 [cdk-image-classification-stack.ts](./lib/cdk-image-classification-stack.ts)에 대해 설명합니다. 

### VPC / Subnet

2개의 AZ로 이중화 될 수 있도록 VPC와 Subnet을 설치합니다. VPC의 CIDR은 "10.0.0.0/16"이며, 2개의 public subnet과 2개의 private subnet을 선언하였습니다.

```typescript
const vpc = new ec2.Vpc(this, 'my-app-vpc', {
    maxAzs: 2, 
    natGateways: 2,
    ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
    vpcName: 'app-vpc',
    subnetConfiguration: [
        {
            subnetType: ec2.SubnetType.PUBLIC, // PUBLIC, PRIVATE_ISOLATED, PRIVATE_WITH_EGRESS
            cidrMask: 20,
            name: 'public'
        },
        {
            subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
            cidrMask: 20,
            name: 'private'
        },
    ]
});
```  

### Cluster

ECS Cluster를 선업합니다.
```typescript
const cluster = new ecs.Cluster(this, "AppEcsCluster", {
    vpc: vpc,
    clusterName: "AppEcsCluster"
}); 
``` 

### Security Group (WEB)

WEB의 Load Balancer와 WEB을 위한 Security Group을 선언합니다. WEB ALB를 위한 Security Group는 HTTP로 들어오는 모든 트래픽을 허용하고, WEB을 위한 Security Group은 WEB ALB의 Security Group과 8000 포트에 대해 인바운드 트래픽을 허용합니다.

```typescript
// Security Group - app-web-alb-sg
const sg_WebAlb = new ec2.SecurityGroup(this, "AppWebAlbSg", {
    vpc: vpc,
    allowAllOutbound: true,
    description: 'security group',
    securityGroupName: "app-web-alb-sg",
});
sg_WebAlb.addIngressRule(
    ec2.Peer.anyIpv4(),
    ec2.Port.tcp(80),
    'allow HTTP traffic from anywhere',
);

// Security Group - app-web-sg
const sg_Web = new ec2.SecurityGroup(this, "AppWebSg", {
    vpc: vpc,
    allowAllOutbound: true,
    description: 'security group',
    securityGroupName: "app-web-sg",
});
sg_Web.addIngressRule(
    ec2.Peer.securityGroupId(sg_WebAlb.securityGroupId),
    ec2.Port.tcp(8000),
    'allow TCP traffic from Web',
);  
``` 

### Docker 이미지 준비 

"/web"에 있는 Dockerfile을 이용하여 도커 컨테이너 이미지를 빌드하고 ECR에 업로드합니다.

```typescript
const webImage = ecs.ContainerImage.fromAsset('../web');
``` 

### Task Definition (WEB)

ECR의 WEB 서비스를 위한 Task를 아래와 같이 정의합니다. 이때 WEB 컨테이너는 8000번 포트로 HTTP를 이용합니다. 

```typescript
const taskDefinition_Web = new ecs.FargateTaskDefinition(this, 'ServiceTaskForWeb', {
    family: 'app-web-td'
});
taskDefinition_Web.addContainer('app-web', {
    image: webImage,
    portMappings: [{
        containerPort: 8000,
        protocol: ecs.Protocol.TCP,
        name: "app-web-8000-tcp",
        appProtocol: ecs.AppProtocol.http,
    }],
    logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'AppWeb',
        logRetention: logs.RetentionDays.ONE_WEEK,
    }),
    containerName: "app-web"
});
``` 

### Service (WEB)

WEB는 Fargate에 서비스 형태로 올라갑니다. 아래와 같이 최대 4개, 최소 2개이고 기본은 2개로 선언합니다. 오토스케일링은 CPU 부하가 70%를 기준으로 하고 300초 Cool Up/Down 시간을 가지도록 선언하였습니다.

```typescript
const fargateService_Web = new ecs.FargateService(this, 'ServiceForWeb', {
    cluster: cluster,
    taskDefinition: taskDefinition_Web,
    serviceName: "app-web-service",
    desiredCount: 2,
    assignPublicIp: false,
    securityGroups: [sg_Web],
});
// Setup AutoScaling policy
const scalingWeb = fargateService_Web.autoScaleTaskCount({
    minCapacity: 2,
    maxCapacity: 4
});
scalingWeb.scaleOnCpuUtilization('CpuScalingWeb', {
    policyName: "app-web-asg-policy",
    targetUtilizationPercent: 70,
    scaleInCooldown: cdk.Duration.seconds(300),
    scaleOutCooldown: cdk.Duration.seconds(300)
});
``` 

### Load Balancer (WEB)

ALB를 이용하여 WEB에 접속할 수 있도록 아래와 같이 Load Balancer를 설정합니다. 80포트로 들어오는 트래픽을 분배하며, "/health"를 이용해 Health 상태를 확인합니다. Load Balancer가 연결되는 Target Group은 Fargate에 서비스로 올라가는 WEB입니다. 

```typescript
// load balancer for Web
const alb_web = new elbv2.ApplicationLoadBalancer(this, 'alb', {
    loadBalancerName: "app-web-alb",
    internetFacing: true,
    ipAddressType: elbv2.IpAddressType.IPV4,
    vpc: vpc,
    securityGroup: sg_WebAlb,
});
const listener = alb_web.addListener('Listener', {
    port: 80,
    open: true,
});
listener.addTargets('targetServiceWeb', {
    targets: [fargateService_Web],
    healthCheck: {
        enabled: true,
        path: '/health',
    },
    targetGroupName: "app-web-alb-tg",
    protocol: elbv2.ApplicationProtocol.HTTP,
    port: 8000,
    protocolVersion: elbv2.ApplicationProtocolVersion.HTTP1,
}); 
``` 

웹어플리케이션의 진입 URL은 아래와 같이 WEB Load Balancer의 DNS 이름입니다.

```typescript
new cdk.CfnOutput(this, 'WebPageURL', {
    value: "http://" + alb_web.loadBalancerDnsName,
    description: 'Url of webpage',
}); 
``` 

### Security Group (WAS)

WAS 서버를 위한 Security Group을 선언합니다. WAS ALB의 Security Group은 WEB Security Group의 80 트래픽을 허용합니다. WAS의 Security Group은 WAS ALB로 부터 전달되는 트래픽을 허용합니다.

```typescript
// Security Group - app-was-alb-sg
const sg_WasAlb = new ec2.SecurityGroup(this, "AppWasAlbSg", {
    vpc: vpc,
    allowAllOutbound: true,
    description: 'security group of WAS ALB',
    securityGroupName: "app-was-alb-sg",
});
sg_WasAlb.addIngressRule(
    ec2.Peer.securityGroupId(sg_Web.securityGroupId),
    ec2.Port.tcp(80),
    'allow HTTP traffic from WEB',
);

// Security Group - app-was-sg
const sg_Was = new ec2.SecurityGroup(this, "AppWasSg", {
    vpc: vpc,
    allowAllOutbound: true,
    description: 'security group of WAS',
    securityGroupName: "app-was-sg",
});
sg_Was.addIngressRule(
    ec2.Peer.securityGroupId(sg_WasAlb.securityGroupId),
    ec2.Port.tcp(8081),
    'allow tcp traffic from WAS ALB',
);  
``` 

### Docker 이미지 준비 (WAS) 

"/was"에 있는 Dockerfile로 WEB용 도커 컨테이너 이미지를 빌드하고 ECR에 저장합니다.

```typescript
const wasImage = ecs.ContainerImage.fromAsset('../was'); 
``` 

WAS가 SageMaker에 대한 권한을 가지도록 IAM Policy와 Role을 생성합니다.

```typescript
const wasTaskRole = new iam.Role(this, "EcsWasTaskRole", {
    roleName: "ecsWasTaskRole",
    assumedBy: new iam.ServicePrincipal("ecs-tasks.amazonaws.com")
});
wasTaskRole.attachInlinePolicy(new iam.Policy(this, 'EcsWasTaskPolicy', {
    policyName: "InvokeSageMakerEndpoint",
    statements: [new iam.PolicyStatement({
        actions: ['sagemaker:InvokeEndpoint'],
        resources: ['*'],
    })],
}));
``` 

### Task Definition (WAS)

WAS를 위한 Fargrate용 Task를 선언합니다. cpu와 memory 조건을 설정할 수 있으며 8081번 포트로 사용하도록 설정하였습니다.

```typescript
// Fargate task definition for Was
const taskDefinition_Was = new ecs.FargateTaskDefinition(this, 'ServiceTaskForWas', {
    family: 'app-was-td',
    cpu: 1024, // 1024 (1 vCPU) 
    memoryLimitMiB: 3072, // 3 GB,
    taskRole: wasTaskRole,
    // runtimePlatform: {cpuArchitecture: ecs.CpuArchitecture.ARM64}  // X86_64 or ARM64
});
taskDefinition_Was.addContainer('app-was', {
    image: wasImage,
    portMappings: [{
        containerPort: 8081,
        protocol: ecs.Protocol.TCP,
        name: "app-was-8081-tcp",
        appProtocol: ecs.AppProtocol.http,
    }],
    logging: ecs.LogDrivers.awsLogs({
        streamPrefix: 'AppWas',
        logRetention: logs.RetentionDays.ONE_WEEK,
    }),
    containerName: "app-was",
}); 
``` 

### Service (WAS)

Fargate의 서비스로 동작하는 WAS를 정의합니다. 최소 2개, 최대 4개의 Task를 사용할 수 있도록 선언하였으며, 오토스케일링 조건도 정의하였습니다.

```typescript
// Fargate Service for WAS
const fargateService_Was = new ecs.FargateService(this, 'ServiceForWas', {
    cluster: cluster,
    taskDefinition: taskDefinition_Was,
    serviceName: "app-was-service",
    desiredCount: 2,
    assignPublicIp: false,
    securityGroups: [sg_Was],
});
// Setup AutoScaling policy
const scalingWas = fargateService_Was.autoScaleTaskCount({
    minCapacity: 2,
    maxCapacity: 4
});
scalingWas.scaleOnCpuUtilization('CpuScalingWas', {
    policyName: "app-was-asg-policy",
    targetUtilizationPercent: 70,
    scaleInCooldown: cdk.Duration.seconds(300),
    scaleOutCooldown: cdk.Duration.seconds(300)
});
``` 

### Load Balancer (WAS)

WAS를 위한 Load Balancer를 설정하였습니다. WAS는 외부로 연결되지 않고 WEB에서 80번 포트로 들어온 트래픽만 허용합니다. WAS와는 8081번 포트로 연결됩니다.

```typescript
// load balancer for WAS
const alb_was = new elbv2.ApplicationLoadBalancer(this, 'AlbWas', {
    loadBalancerName: "app-was-alb",
    internetFacing: false,  // internal
    ipAddressType: elbv2.IpAddressType.IPV4,
    vpc: vpc,
    securityGroup: sg_WasAlb,
});
const listener_was = alb_was.addListener('Listener', {
    port: 80,
    open: true,
});
listener_was.addTargets('targetServiceForWAS', {
    targets: [fargateService_Was],
    healthCheck: {
        enabled: true,
        path: '/health',
    },
    targetGroupName: "app-was-alb-tg",
    protocol: elbv2.ApplicationProtocol.HTTP,
    port: 8081,
    protocolVersion: elbv2.ApplicationProtocolVersion.HTTP1,
});  
``` 

[router.js](https://github.com/kyopark2014/ecs-fargate-sagemaker-based-webservice/blob/main/web/app/router.js)에서 "BASE_URL"로 WEB ALB의 URL입력하므로 아래와 같이 CDK OUTPUT을 출력합니다.

```typescript
new cdk.CfnOutput(this, 'Was-Alb-Url', {
    value: "http://" + alb_was.loadBalancerDnsName,
    description: 'Address of WAS ALB URL',
}); 
``` 

## Interface Endpoint

[Interface Endpoint](https://github.com/amcquistan/aws-vpc-endpoints-with-cdk/blob/main/lib/vpc-endpoints-stack.ts)



## Reference

[class Vpc (construct)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.Vpc.html)

[Creating an AWS Fargate service using the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/ecs_example.html)

[Introduction to the Cloud Development Kit (CDK)](https://catalog.us-east-1.prod.workshops.aws/workshops/5962a836-b214-4fbf-9462-fedba7edcc9b/en-US)

[VPC: enum SubnetType](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.SubnetType.html)

[CDK Construct library for higher-level ECS Constructs](https://docs.aws.amazon.com/cdk/api/v1/docs/aws-ecs-patterns-readme.html)

[class ApplicationLoadBalancedFargateService (construct)](https://docs.aws.amazon.com/cdk/api/v1/docs/@aws-cdk_aws-ecs-patterns.ApplicationLoadBalancedFargateService.html)

[Deploying applications to ECS Fargate with AWS CDK](https://www.gravitywell.co.uk/insights/deploying-applications-to-ecs-fargate-with-aws-cdk/)

[How to setup AutoScale for ECS service using CDK](https://medium.com/@alexander.sirenko/setup-autoscale-for-ecs-service-using-cdk-39f03cc3f046)

[class SecurityGroup (construct)](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.SecurityGroup.html)

[Security Group Examples in AWS CDK - Complete Guide](https://bobbyhadz.com/blog/aws-cdk-security-group-example)

[Creating an Application Load Balancer in AWS CDK](https://bobbyhadz.com/blog/aws-cdk-application-load-balancer)

[class ApplicationTargetGroup (construct)](https://docs.aws.amazon.com/cdk/api/v1/docs/@aws-cdk_aws-elasticloadbalancingv2.ApplicationTargetGroup.html)

[class InterfaceVpcEndpointAwsService](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.InterfaceVpcEndpointAwsService.html)
