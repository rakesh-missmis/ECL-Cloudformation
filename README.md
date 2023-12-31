
# ECL Cloudformation Practice Template Documentation.

AWS CloudFormation is an Amazon Web Services (AWS) service that enables users to automate the process of creating and managing AWS infrastructure resources. The mail key components are Templates, Resources, Stacks etc.

Here I have done my task of this Cloudformation  template automates the deployment of an Elastic Beanstalk application behind an Elastic Load Balancer (ELB) and an Auto Scaling Group (ASG). The ASG has scaling policies based on CPU utilization to handle varying loads efficiently.



## Requirements
   1. AWS Account Login.
   2. AWS Management Console
   3. KeyPair 
   4. Security GP
   ```bash
  git clone https://github.com/rakesh-missmis/ECL-Cloudformation.git
```




## Parameters
In the Parameter Stage define only 3 variables with objects i.e VPCid, StackName, Tags.
```bash
  AWSTemplateFormatVersion: 2010-09-09
    Description: AWS CloudFormation Template to deploy Elastic Beanstalk Application with Auto Scaling Group and Elastic Load Balancer
  Parameters:
    VPCId:
      Type: AWS::EC2::VPC::Id
      Description: ID of the existing VPC where resources will be created.
    StackName:
      Type: String
      Description: A unique name for the CloudFormation stack.
    Tags:
      Type: String
      Description: A list of tags in the format 'Name=Application' to apply to the resources created.
```

## Resources
1. Create a Elastic Beanstalk Application:
2. Create Elastic Beanstalk Enviornment (Use Python 3.8 as a platform)
3. Create LoadBalancer
4. Create Targetgroup with Listner port 80 (HTTP) 
5. Create Autoscalling Group With EC2 Instance (Deploy stress that generate High CPU load.
6. Scalling PolicyIn, Scalling PolicyOut, CpuUtilization (used for Instances are Scale In & Scale Out as per Load in application) 


```bash
Resources:
  MyElasticBeanstalkApplication:
    Type: AWS::ElasticBeanstalk::Environment
    Properties:
      ApplicationName: !Ref StackName
      EnvironmentName: !Ref StackName
      Platform: Python 3.8  # You can change this to your desired platform
      SolutionStackName: "64bit Amazon Linux 2 v5.5.0 running Python 3.8"
      OptionSettings:
        - Namespace: aws:elasticbeanstalk:environment
          OptionName: EnvironmentType
          Value: LoadBalanced
        - Namespace: aws:elasticbeanstalk:environment
          OptionName: ServiceRole
          Value: aws-elasticbeanstalk-service-role
        - Namespace: aws:autoscaling:launchconfiguration
          OptionName: EC2KeyName
          Value: ecldemo-1  # Replace with your desired EC2 key pair
      Tags:
        - Key: Name
          Value: !Ref StackName
        - Key: Environment
          Value: !Ref StackName

  MyLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: MyLoadBalancer
      Subnets: <subnet-06c95ba1eb4083925,subnet-03061aa0afaa8f0d8>
      SecurityGroups: <sg-07f20eca5b6b5131f,sg-0f461be130aed64cf>
      Scheme: internet-facing
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: "60"
      Tags: Tags
      Listeners:
        - Port: 80
          Protocol: HTTP
          DefaultActions:
            - Type: fixed-response
              FixedResponseConfig:
                StatusCode: 200
                ContentType: text/plain
                Content: OK
  MyTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: MyTargetGroup
      Port: 80
      Protocol: HTTP
      TargetType: instance
      VpcId: VPCId
      HealthCheckIntervalSeconds: 30
      HealthCheckPath: /stress
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2
  EBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: fixed-response
          FixedResponseConfig:
            StatusCode: 200
            ContentType: text/plain
            Content: OK
      LoadBalancerArn: MyLoadBalancer
      Port: 80
      Protocol: HTTP
  LaunchConfig:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      LaunchConfigurationName: MyLaunchConfiguration
      ImageId: ami-09318275b68e42715
      InstanceType: t2.micro
      SecurityGroups: <sg-07f20eca5b6b5131f,sg-0f461be130aed64cf>
      KeyName: ecldemo-1
  MyAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: MyAutoScalingGroup
      LaunchConfigurationName: LaunchConfig
      MinSize: 2
      MaxSize: 4
      DesiredCapacity: 2
      VPCZoneIdentifier: <subnet-06c95ba1eb4083925,subnet-03061aa0afaa8f0d8>
      Tags:
        - Key: Name
          Value: MyAutoScalingGroup
          PropagateAtLaunch: "true"
      HealthCheckType: EC2
      HealthCheckGracePeriod: 300
      TargetGroupARNs:
        - MyTargetGroup
      MetricsCollection:
        - Granularity: 1Minute
      NotificationConfigurations:
        - TopicARN: <SNS Topic ARN>
          NotificationTypes:
            - autoscaling:EC2_INSTANCE_LAUNCH
            - autoscaling:EC2_INSTANCE_TERMINATE
  MyScaleOutPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      Cooldown: 300
      ScalingAdjustment: 1
      AutoScalingGroupName: MyAutoScalingGroup
  MyScaleInPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      Cooldown: 600
      ScalingAdjustment: -1
      AutoScalingGroupName: MyAutoScalingGroup
      EstimatedInstanceWarmup: 600
  CPUUtilizationAlarmHigh:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Scale-out if CPU utilization is high
      Namespace: AWS/EC2
      MetricName: CPUUtilization
      Dimensions:
        - Name: AutoScalingGroupName
          Value: MyAutoScalingGroup
      Statistic: Average
      Period: "300"
      Threshold: "80"
      ComparisonOperator: GreaterThanThreshold
      EvaluationPeriods: "1"
      AlarmActions:
        - MyScaleOutPolicy
  CPUUtilizationAlarmLow:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Scale-in if CPU utilization is low
      Namespace: AWS/EC2
      MetricName: CPUUtilization
      Dimensions:
        - Name: AutoScalingGroupName
          Value: MyAutoScalingGroup
      Statistic: Average
      Period: "600"
      Threshold: "30"
      ComparisonOperator: LessThanThreshold
      EvaluationPeriods: "2"
      AlarmActions:
        - MyScaleInPolicy
 ```
## Output
  The Output is Show the EndpointURL
```bash
Outputs:
  EndpointURL:
    Description: URL of the deployed Elastic Beanstalk application.
    Value: !GetAtt MyElasticBeanstalkApplication.EndpointURL
    Export:
      Name: !Sub ${StackName}EndpointURL
```
## Appendix

Any additional information goes here

