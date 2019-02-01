# Sample stack Lambda backed API

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: >
  This template creates an API that uses a lambda function and a dynamo table
  Last Modified: 30.01.2019
  Author: Exec <aerioeus@gmail.com>
Metadata: {}

Parameters:
  Owner:
    Description: Team or Individual Name Responsible for the Stack.
    Type: String
    Default: Andreas Rose

  Project:
    Description: Enter Project Name.
    Type: String
    Default: invoicegenerator

  Subproject:
    Description: Enter Project Name.
    Type: String
    Default: ebs

  EnvironmentName:
    Description: An environment name that will be prefixed to resource names
    Type: String
    Default: Dev
    AllowedValues:
      - Dev
      - Prod
    ConstraintDescription: Specify either Dev or Prod

  ApiKeySourceType:
    Description: Setup for the ApiKeySourceType
    Type: String
    Default: HEADER

  minimumCompressionSize:
    Description: >
      integer that is used to enable compression (with non-negative between 0
      and 10485760 (10M) bytes, inclusive) or disable compression (with a null value) on an API
    Type: String
    Default: 1048576

  EndpointConfiguration:
    Description: A list of the endpoint types of the domain name
    Type: String
    Default: EDGE
    AllowedValues:
      - EDGE
      - REGIONAL
    ConstraintDescription: Specify either EDGE or REGIONAL

  RestApiName:
    Description: Name for the API Gateway RestApi
    Type: String
    Default: Invoiceapi2

  BasePath:
    Description: >
      custom domain name for your API in Amazon API Gateway
      Uppercase letters are not supported
    Type: String
    Default: invoiceapi2

  IdentitySource:
    Description: The source of the identity in an incoming request
    Type: String
    Default: Authorization

  ThrottlingBurst:
    Description: number of burst requests per second that API Gateway permits across all APIs, stages, and methods in an AWS account
    Type: Number
    Default: 200

  ThrottlingRate:
    Description: number of steady-state requests per second that API Gateway permits across all APIs, stages, and methods in an AWS account
    Type: Number
    Default: 500

  Stagename:
    Description: name of the stage, which API Gateway uses as the first path segment in the invoked Uniform Resource Identifier (URI)
    Type: String
    Default: v1

  # this parameter is being provided via masterstack
  APIDomain:
    Description: DomainName for the API
    Type: String

  # this parameter is being provided via masterstack
  APIClientCertificateName:
    Description: client certificate that Amazon API Gateway (API Gateway) uses to configure 
      client-side SSL authentication for sending requests to the integration endpoint
    Type: String

  # this parameter is being provided via masterstack
  LambdaRoleARN:
    Description: allows API Gateway to access Lambda
    Type: String

Resources:
  Api6:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Description: Api for Lambda Ressource for EBS
      Name: !Ref RestApiName
      ApiKeySourceType: !Ref ApiKeySourceType
      Body:
        Fn::Transform:
          Name: AWS::Include
          Parameters:
            Location: !Sub "s3://${AWS::AccountId}-${Project}-swaggerapis/lambda2_swagger.yaml"
      EndpointConfiguration:
        Types:
          - !Ref EndpointConfiguration
      FailOnWarnings: true
      MinimumCompressionSize: !Ref minimumCompressionSize

  # lambda1 API
  SubdomainMapping6:
    Type: AWS::ApiGateway::BasePathMapping
    Properties:
      BasePath: !Ref BasePath
      DomainName: !Ref APIDomain
      RestApiId: !Ref Api6
      Stage: !Ref ApiStage6

  # a "stage" represents a unique identifier for a particular version of a deployed RestApi like "dev", "test", "prod" etc
  # if the stage is assigned the deployment then that is callable by users
  ApiStage6:
    Type: AWS::ApiGateway::Stage
    Properties:
      Description: !Sub ${EnvironmentName}-Stage
      StageName: !Ref Stagename
      RestApiId: !Ref Api6
      DeploymentId: !Ref ApiDeployment6
      ClientCertificateId: !Ref APIClientCertificateName
      AccessLogSetting:
        DestinationArn:
          Fn::ImportValue: !Sub ${EnvironmentName}-ApiLogGroupARN
        Format: $context.error.message
          $context.error.messageString
          $context.identity.user
          $context.requestId
          $context.requestTime
          $context.resourceId
          $context.stage
          $context.responseLatency
          $context.identity.userArn
      MethodSettings:
        - ResourcePath: "/*"
          HttpMethod: "*"
          MetricsEnabled: true
          DataTraceEnabled: true
          LoggingLevel: INFO
      TracingEnabled: true

  # deploys the Api API to a stage specified in the stage resource
  ApiDeployment6:
    Type: AWS::ApiGateway::Deployment
    Properties:
      Description: Deployment API
      RestApiId: !Ref Api6

  # creates a unique key that you can distribute to clients
  ApiKey6:
    Type: AWS::ApiGateway::ApiKey
    DependsOn:
      - ApiStage6
    Properties:
      Name: ApiKey6
      Description: CloudFormation API Key V1
      Enabled: true
      StageKeys:
        - RestApiId: !Ref Api6
          StageName: !Ref ApiStage6

  # creates a usage plan to enforce throttling and quota
  UsagePlan6:
    Type: AWS::ApiGateway::UsagePlan
    Properties:
      Description: Customer usage plan
      ApiStages:
        - ApiId: !Ref Api6
          Stage: !Ref ApiStage6
      Quota:
        Limit: 5000
        Period: MONTH
      Throttle:
        BurstLimit: !Ref ThrottlingBurst
        RateLimit: !Ref ThrottlingRate
      UsagePlanName: Invoiceapp-usageplan

  usagePlanKey6:
    Type: AWS::ApiGateway::UsagePlanKey
    Properties:
      KeyId: !Ref ApiKey6
      KeyType: API_KEY
      UsagePlanId: !Ref UsagePlan6

  LambdaScanDdb:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: LambdaScanDdb
      Runtime: nodejs8.10
      Code:
        S3Bucket:
          Fn::ImportValue: !Sub ${EnvironmentName}-LambdaCodeBucket-Name
        S3Key: lambdascandynamoinvoices/index.zip
      Handler: index.handler
      Role: !Ref LambdaRoleARN
      Timeout: 30
      MemorySize: 128

  # Permissions to use Lambda Function
  LambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:invokeFunction
      FunctionName: !GetAtt LambdaScanDdb.Arn
      Principal: apigateway.amazonaws.com
      SourceArn:
        Fn::Join:
          - ""
          - - "arn:aws:execute-api:"
            - Ref: AWS::Region
            - ":"
            - Ref: AWS::AccountId
            - ":"
            - Ref: Api6
            - "/*"

  LambdaPutDdb:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: LambdaPutDdb
      Runtime: nodejs8.10
      Code:
        S3Bucket:
          Fn::ImportValue: !Sub ${EnvironmentName}-LambdaCodeBucket-Name
        S3Key: lambdaputdynamoinvoices/index.zip
      Handler: index.handler
      Role: !Ref LambdaRoleARN
      Timeout: 30
      MemorySize: 128

  # Permissions to use Lambda Function
  LambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:invokeFunction
      FunctionName: !GetAtt LambdaPutDdb.Arn
      Principal: apigateway.amazonaws.com
      SourceArn:
        Fn::Join:
          - ""
          - - "arn:aws:execute-api:"
            - Ref: AWS::Region
            - ":"
            - Ref: AWS::AccountId
            - ":"
            - Ref: Api6
            - "/*"

  LambdaQueryDdb:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: LambdaQueryDdb
      Runtime: nodejs8.10
      Code:
        S3Bucket:
          Fn::ImportValue: !Sub ${EnvironmentName}-LambdaCodeBucket-Name
        S3Key: lambdaquerydynamoinvoices/index.zip
      Handler: index.handler
      Role: !Ref LambdaRoleARN
      Timeout: 30
      MemorySize: 128

  # Permissions to use Lambda Function
  LambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:invokeFunction
      FunctionName: !GetAtt LambdaQueryDdb.Arn
      Principal: apigateway.amazonaws.com
      SourceArn:
        Fn::Join:
          - ""
          - - "arn:aws:execute-api:"
            - Ref: AWS::Region
            - ":"
            - Ref: AWS::AccountId
            - ":"
            - Ref: Api6
            - "/*"

Outputs:
  StackName:
    Description: The Name of the Stack
    Value: !Ref AWS::StackName

  Owner:
    Description: Team or Individual that Owns this Formation.
    Value: !Ref Owner

  Project:
    Description: The project name
    Value: !Ref Project

  RestApiId:
    Description: returns the RestApi ID
    Value: !Ref Api6

  RootResourceId:
    Description: returns the root resource ID for a RestApi resource
    Value: !GetAtt Api6.RootResourceId

  RootUrl:
    Description: Root URL of the API gateway
    Value:
      Fn::Join:
        - ""
        - - https://
          - Ref: Api6
          - ".execute-api."
          - Ref: AWS::Region
          - ".amazonaws.com"

  ApiDomain:
    Description: Domain of the API gateway
    Value:
      Fn::Join:
        - ""
        - - Ref: Api6
          - ".execute-api."
          - Ref: AWS::Region
          - ".amazonaws.com"

  ApiKeyId:
    Description: returns the API key ID
    Value: !Ref ApiKey6

  ApiStageName:
    Description: returns the name of the stage
    Value: !Ref ApiStage6
```

