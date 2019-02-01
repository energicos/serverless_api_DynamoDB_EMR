# multi-API Gateway-Lambda-NoSQL DynamoDB Setup

[TOC]



## 1. Introduction

### 1.1 General Info



- for this assignment we have created an RDMS- backed **“Access-Miniapp ”** model created in **Office365 Access** showing 2 main area of our work
  - Customer Management (Contacts, contacts)
  - Machinery and Components (Powerplants and its components)
- we want to convert this integrated “miniapp” into a Webapp running serverless with the following main parts
  - Frontend (just functional) - no fancy design
  - a backend consisting of 2 DynamoDB tables
    - one table: Contacts/ Contracts
    - one table: Machinery/ Components
    - one table: read live data from machines (API will be provided)
  - later we will have an Integration of these tables with AWS EMR for Analysis
  - APIs with Lambda integration (proxy) to write, read, query,…data from the backend
  - API with custom integration to the DynamoDB backend for write and read without Business logic involved



### 1.2 Environment

- we are developing this microservice-based WebApp running solely on AWS
- the stacks must be created as `yaml` files
- everything shall be implemented in a **serverless** manner
- BUT we don’t use the **serverless framework** but work straight with AWS Cloudformation
- all our deployments are Cloudformation based
- we don't deploy Resources into Dev/ Prod Environment via Console
- Console can be used only for testing purposes
- the stacks are being launched either singly or as masterstacks containing multiple nested stacks using AWSCLI like that:

```shell
$ aws cloudformation create-stack \
--stack-name api-master-stack \
--template-body file://api_masterstack.yaml \
--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM
```



### 1.2 API Setup - Rules

- the API-Gateway configuration is being delivered by a Swagger (OpenAPI) file which is being uploaded onto an s3 bucket and used during stack deployment as source for the API definition using

```yaml
      Body:
        Fn::Transform:
          Name: AWS::Include
          Parameters:
            Location: !Sub "s3://${AWS::AccountId}-${Project}-swaggerapis/lambda2_swagger.yaml"
```

- as backend we use solely DynamoDB (which is later integrated with AWS EMR)
- we implement a “multi-API-Gateway under one domain” - approach involving **basepathmapping** in our Cloudformation stacks  (see sample stacks)
- the API access is being secured by a **Cognito based Authorizer** which uses a Cognito pool (Cognito stack)



## 2. Scope of delivery

| __Results__              | __Description__                                              |
| ------------------------ | ------------------------------------------------------------ |
| Documentation            | A documentation describing the resources and the approach as well as the elaborations on the deployed  AWS extensions. Please follow the [documentation template](./docu_template.md) |
| Cloudformation templates | Cloudformation templates with the resources working as expected in an automated way (cognito, Api Gateway masterstack, IAM, swaggerfiles, nodejs code for Lambdas,  Lambda Stack) |
| Swagger files            | Swagger files with proxy Lambda integration/ custom DynamoDB integration for any necessary Method using |
| Postman/Curl Test        | provide curl or postman test Results showing that the Integration works with Cognito Token; |



## 3. Details of the assignment

### 3.1 Goal 1 - NoSQL Structure

- the first goal is to split up the provided **RDBMS Backend** of the **“Access-Miniapp ”**(backend and frontend has been modeled in Microsoft Access) and create 2 Dynamo Tables for the content
- the nested structure of the JSON file should look like that:

```yaml
{
  "DueDate": "2013-02-15",
  "Balance": 1990.19,
  "InvoiceId": "ENCP030",
  "Status": "Payable",
  "Line": [
    {
      "Description": "Sample Expense",
      "Amount": 500,
      "DetailType": "ExpenseDetail",
      "ExpenseDetail": {
        "Customer": {
          "value": "ABC123",
          "name": "Sample Customer"
        },
        "Ref": {
          "value": "DEF234",
          "name": "Sample Construction"
        },
        "Account": {
          "value": "EFG345",
          "name": "Fuel"
        },
        "LineStatus": "Billable"
      }
    }
  ],
  "Vendor": {
    "value": "GHI456",
    "name": "Sample Bank"
  },
  "APRef": {
    "value": "HIJ567",
    "name": "Accounts Payable"
  },
  "TotalAmt": 1990.19
}
```

- the content of the **“Access-Miniapp ”** shall be split up into 2 Tables with the following context
  - DynamoDB table1:
    - contains all attributes for Contacts like name, address, etc
    - contains all attributes for Contacts(aka Products): like Contract Number, begin, contract id etc
  - DynamoDB Table2: Machinery/ Components
    - contains all details like machine Id, purchase date etc
    - contains data re components: like type/price/implementation date of boiler, burner, pumps, etc.

- then create a third DynamoDB table for time series data to collect live data related to these machinery like:
  - temperature
  - pressure
  - on/off cycle

- **keep in mind that we need to analyze the data of each table cross-table later on (using EMR)**

- so the Structure shall be created in a way that later on aggregations and cross-table analysis is possible



### 3.2 Goal 2 - APIs

- after the structure has been set up, the APIs must be developed to input and retrieve the data from the backend DynamoDB
- for this we use solely serverless implementation
  - API with Lambda + **proxy Integration** (see sample)
  - API without Lambda using **custom DynamoDB Integration** (see sample)
- the API content will be based on the access patterns we want to use for the data
- make sure the **Cognito based Authorizer** is defined in the Swagger file and working properly
- these access patterns will be provided by us
- pls start with basic Access patterns like inputting data on the frontend by hand and query them using the frontend of the “Access-Miniapp ”



### 3.4 Goal 3 - Creat a simple frontend to mimic the “Miniapp” Frontend

- last step will be to **integrate the APIs,** with a simple frontend (monotlithic) to show data input and output using the APIs
- pls use HTML and CSS-Grid
- no design required, just mimic the fields on the “Access-Miniapp ”



### 4. API Implementation

### 4.1 Swagger Integration

- the API Definition must be done as Swagger /OpenAPI 2.0
- the swagger files shall be integrated as following:

```yaml
  Api:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Description: TBD
      Name: !Ref RestApiName
      ApiKeySourceType: !Ref ApiKeySourceType
      Body:
        Fn::Transform:
          Name: AWS::Include
          Parameters:
            Location: !Sub "s3://bucketname/swagger.yaml"
      EndpointConfiguration:
        Types:
          - !Ref EndpointConfiguration
      FailOnWarnings: true
      MinimumCompressionSize: !Ref minimumCompressionSize
```

- the OpenAPI Operations shall be integrated with AWS Extension like this:

```yaml
swagger: "2.0"
info:
  version: 1.0.0
  title: Contact Manager - API
  description: >
    This API is intented to enable external users to get information for the path.....

# Specifies a request validator, by referencing a request_validator_name
x-amazon-apigateway-request-validators:
.....

# Specifies the source to receive an API key
x-amazon-apigateway-api-key-source: HEADER

# list of binary media types to be supported by API Gateway
x-amazon-apigateway-binary-media-types:
  - application/json
  - application/xml
  - text/plain; charset=utf-8
  - text/html
  - application/pdf
  - image/png
  - image/gif
  - image/jpeg

securityDefinitions:
  Lambdaauthorizer:
    type: apiKey # Required and the value must be "apiKey" for an API Gateway API.
    name: Authorization # The name of the header containing the authorization token
    in: header # Required and the value must be "header" for an API Gateway API.
    x-amazon-apigateway-authtype: cognito_user_pools # Specifies the authorization mechanism for the client.
    x-amazon-apigateway-authorizer: # An API Gateway Lambda authorizer definition
      type: cognito_user_pools # Required property and the value must "token"
      providerARNs:
      - Fn::ImportValue: { "Fn::Sub" : "${EnvironmentName}-${Project}-UserPoolARN" }

schemes: [http, https]
consumes:
  - application/json
  - application/xml
  - text/plain; charset=utf-8
  - text/html
  - application/pdf
produces:
  - application/json
  - application/xml
  - image/png
  - image/gif
  - image/jpeg
paths:
  /users/{userId}:
    get:
      description: TBD
      security:
      - Lambdaauthorizer: []
      x-amazon-apigateway-integration:
        uri:
          Fn::Join:
          - ''
          - - 'arn:aws:apigateway:'
            - Ref: AWS::Region
            - ":lambda:path/2015-03-31/functions/"
            - Fn::GetAtt:
              - LambdaFunctionName
              - Arn
            - "/invocations"
        credentials:
          Fn::ImportValue: {"Fn::Sub" : "${EnvironmentName}-ApiGatewayRoleARN"}
        passthroughBehavior: when_no_match
        httpMethod: POST
        type: aws_proxy
        .....
```



### 4.2 Rules for the API Creation

#### 4.2.1 Content and Structure compliance

- the following rules concerning the API Setup shall apply:
- **info section** in Swagger shall the following components:
  - title
  - summary
  - description
  - termsofservice
  - contact
  - license(type)
  - tags
- use paging capabilites if in parameters when it makes sense
- simplify the data model description by using `$Ref` for
  - Parameter,
  - Responses,
  - Schema Definitions
- Use **same definitions** on read and write operations whenever possible
- Combine multiple definitions to ensure consistency
- create Models to ensure consistent design for reused properties / use the `allOf` JSON schema property to keep the attributes at root level
- Define a required or optional property in a definition used as a parameter
- use default values for a parameter or a default value for a property in a definition
- use enums when possible
- use Read-Only and Write-Only Properties
- define the following security definitions/ authentication as OAuth2 **on the root level** so it applies to all operations and override **on operation-level** if necessary (like in Post)
  - LegacySecurity; type: basic
  - OauthSecurity; type: OAuth2(like for post/user)
  - MediaSecurity; type: OAuth2 (like for post/images)
- apply globally:
  - OauthSecurity:
    - user
  - LegacySecurity: []
- For the **API Documentation** the following Rules apply:
  - add descriptions at almost every level of the OpenAPI specification
    - operations
    - responses
    - response headers
    - tags
  - use reusable descriptions where possible
  - use tags on operation level to categorize operations
    - use single and multipe tags if an operation belongs to different categories
  - schema shall have a title and a description
  - Properties can be described with description
  - Parameter shall have inline descriptions
  - for Operations use
    - operation id's
    - summary
    - description
  - use GFM (Github Flavored Markdown) in descriptions
    - multiline
    - array descriptions
    - descriptions with code
  - provide Examples for atomic or object properties, definitions, and responses
  - Links to external API documentation
    - link to general API Documentation
    - link to specific API Documentation (like Use-cases etc)
    - link to tag documentation (tags description)
- split the Swagger-file into subfiles and reference them via JSON Pointer
- the components which should be held outside the Main swagger file should be:
  - Request body
  - Response body
  - definitions

```yaml
https://azimi.me/2015/07/16/split-swagger-into-smaller-files.html
https://apihandyman.io/writing-openapi-swagger-specification-tutorial-part-8-splitting-specification-file/
```



#### 4.2.2 Examples for Implementation

##### 4.2.2.1 create a Request Body

- please use the modular style and move the schema definitions to the global `definitions` section and refer to them by using `$ref`
- the following sample uses an **object** to transmit the data:

```yaml
paths:
  /users:
    post:
      summary: Creates a new user.
      consumes:
        - application/json
      parameters:
        - in: body
          name: user
          description: The user to create.
          schema:
            $ref: "#/definitions/User"     # <----------
     responses:
         200:
           description: OK
definitions:
  User:           # <----------
    type: object
    required:
      - userName
    properties:
      userName:
        type: string
      firstName:
        type: string
      lastName:
        type: string
```



##### 4.2.2.2 Define a Response body

- define at the root level and referenced via `$ref`/ dont define it inline in the `Operations

```yaml
      responses:
        200:
          description: A User object
          schema:
            $ref: "#/definitions/User"
definitions:
  User:
    type: object
    properties:
      id:
        type: integer
        description: The user ID.
      username:
        type: string
        description: The user name.
```



##### 4.2.2.3 Define a Default Responses

- if an `operation` returns multiple errors with different HTTP status codes, but all of them have the same response structure, set a default response
- use a`default` response to describe errors collectively, not individually
- "Default" means this response is used for all HTTP codes that are not covered individually for this operation

```yaml
      responses:
        200:
          description: Success
          schema:
            $ref: '#/definitions/User'
        # Definition of all error statuses
        default:
          description: Unexpected error
          schema:
            $ref: '#/definitions/Error'
```



##### 4.2.2.4 Reuse Respones

- If multiple operations return the same response (status code and data), you can define it in the global `responses` section and reference that definition via `$ref` at the operation level.

```yaml
paths:
  /users:
    get:
      summary: Gets a list of users.
      response:
        200:
          description: OK
          schema:
            $ref: "#/definitions/ArrayOfUsers"
        401:
          $ref: "#/responses/Unauthorized"   # <-----
  /users/{id}:
    get:
      summary: Gets a user by ID.
      response:
        200:
          description: OK
          schema:
            $ref: "#/definitions/User"
        401:
          $ref: "#/responses/Unauthorized"   # <-----
        404:
          $ref: "#/responses/NotFound"       # <-----
# Descriptions of common responses
responses:
  NotFound:
    description: The specified resource was not found
    schema:
      $ref: "#/definitions/Error"
  Unauthorized:
    description: Unauthorized
    schema:
      $ref: "#/definitions/Error"
definitions:
  # Schema for error response body
  Error:
    type: object
    properties:
      code:
        type: string
      message:
        type: string
    required:
      - code
      - message
```





### 4.3 Further details

#### 4.3.1 General requirements

- the stacks must be created as `yaml` files

- use **CognitoPool  for Request Authorization exclusively**

- enable Payload Compression for the API

- enable Request Validation on API Level (set to ALL)

- create a Usage Plan Resource for the API Gateway and an API Key

- enable Cloudwatch logs

- set Default Responses

- Stage in Swagger is `/v1`

- use API Gateway **Cognito** Authorizers

- enable cross-origin resource sharing (CORS) for selected methods on the resource

- use Client-Side SSL Certificates for Authentication by the Backend/ Configure Backend HTTPS Server to Verify the Client Certificate

- create and Use Usage Plans with API Keys

- we use a custom API domain name

- use the following OpenAPI Extensions and its sub properties to integrate AWS API Gateway and Swagger API

  - `x-amazon-apigateway-authorizer`
  - `x-amazon-apigateway-authtype`
  - `x-amazon-apigateway-request-validator`
  - `x-amazon-apigateway-integration`
  - `x-amazon-apigateway-any-method Object`
  - `x-amazon-apigateway-binary-media-types Property`
  - `x-amazon-apigateway-gateway-responses`
  - `x-amazon-apigateway-api-key-source`

- dont use

  - `x-amazon-apigateway-any-method`



#### 4.3.2 Design Operations

- the API Responses shall cover successful responses and any *known* errors
- By "known errors" we mean, for example, a 404 Not Found response for an operation that returns a resource by ID, or a 400 Bad Request response in case of invalid operation parameters
- define `schema` for Response Bodies at the root level and reference it via `$ref`
- implement Request Validation
  - use the  `x-amazon-apigateway-request-validators`  Object/ no handcoded Validation implementations are allowed
    - for  **payloads** against the specified `schema`
    - as  **basic validation** of required HTTP request parameters in
      - the URI,
      - query string, and
      - headers
  - Note on Content-Type:
    - Request body validation is performed according to the configured request Model which is selected by the value of the request ‘Content-Type’ header
    - In order to enforce validation and restrict requests to explicitly-defined content types, it’s a good idea to use strict request passthrough behavior (‘”passthroughBehavior”: “never”‘), so that unsupported content types fail with 415 “Unsupported Media Type” response
- don't use `{proxy+}` Resources at all



#### 4.3.3 Operation Integrations with Backends

- create the proper `IntegrationRequest  `and `IntegrationResponses `  for any non-proxy API Gateway Integration with DynamoDB like that:
- **IntegrationRequest - sample**

```java
#set($inputRoot = $input.path('$'))
{
    "TableName": "Invoicelog",
	"ConditionExpression": "attribute_not_exists(InvoiceId)",
    "Item": {
        "DueDate": {
            "S": "$input.path('$.DueDate')"
        },
        "Balance": {
            "N": "$input.path('$.Balance')"
        },
        "UUID": {
            "S": "$context.requestId"
        },
        "InvoiceId": {
            "S": "$input.path('$.InvoiceId')"
        },
        "Status": {
            "S": "$input.path('$.Status')"
        },
        "Line": {
            "L": [
            #foreach($elem in $inputRoot.Line)
                {
                    "M": {
                        "Description": {
                            "S": "$elem.Description"
                        },
                        "Amount": {
                            "N": "$elem.Amount"
                        },
                        "DetailType": {
                            "S": "$elem.DetailType"
                        },
                        "ExpenseDetail": {
                                    "M": {
                                    "Customer": {
                                    "M": {
                                        "value": {
                                            "S": "$elem.ExpenseDetail.Customer.value"
                                        },
                                        "name": {
                                            "S": "$elem.ExpenseDetail.Customer.name"
                                        }
                                        }
                          },
                                    "Ref": {
                                    "M": {
                                        "value": {
                                            "S": "$elem.ExpenseDetail.Ref.value"
                                        },
                                        "name": {
                                            "S": "$elem.ExpenseDetail.Ref.name"
                                        }
                        }
                        },
                                  "Account": {
                                    "M": {
                                        "value": {
                                            "S": "$elem.ExpenseDetail.Account.value"
                                        },
                                        "name": {
                                            "S": "$elem.ExpenseDetail.Account.name"
                                        }
                                        }
                                      },
                                  "LineStatus": {
                                      "S": "$elem.ExpenseDetail.LineStatus"
                                      }
                        }
                    }
                  }
                }#if($foreach.hasNext),#end
            #end
            ]
        },
        "Vendor": {
                "M": {
                    "value": {
                        "S": "$input.path('$.Vendor.value')"
                    },
                    "name": {
                        "S": "$input.path('$.Vendor.name')"
                    }
                }
            },
        "APRef": {
              "M": {
                  "value": {
                      "S": "$input.path('$.APRef.value')"
                  },
                  "name": {
                      "S": "$input.path('$.APRef.name')"
                  }
              }
          },
        "TotalAmt": {
            "N": "$input.path('$.TotalAmt')"
        }
    }
}
```

- **Integration Response Sample**

```java
#set($inputRoot = $input.path('$'))
{
    "Invoices": [
        #foreach($elem in $inputRoot.Items)
        {
         "InvoiceNumber": "$elem.InvoiceId.S",
          "Balance": "$elem.Balance.N",
          "Status": "$elem.Status.S",
          "TotalAmt": "$elem.TotalAmt.N",
          "Line": [
              #foreach($item in $elem.Line.L)
              {
                "Description": "$item.M["Description"].S",
                "DetailType": "$item.M["DetailType"].S",
                "Amount": "$item.M["Amount"].N",
                "CustomerValue": "$item.M["ExpenseDetail"].M["Customer"].M["value"].S",
                "CustomerName": "$item.M["ExpenseDetail"].M["Customer"].M["name"].S",
                "RefValue": "$item.M["ExpenseDetail"].M["Ref"].M["value"].S",
                "RefName": "$item.M["ExpenseDetail"].M["Ref"].M["name"].S",
                "AccountValue": "$item.M["ExpenseDetail"].M["Account"].M["value"].S",
                "AccountName": "$item.M["ExpenseDetail"].M["Account"].M["name"].S",
                "InvoiceStatus": "$item.LineStatus.S",
              }#if($foreach.hasNext),#end
              #end
            ],
          "VendorValue": "$elem.Vendor.M["value"].S",
          "VendorName": "$elem.Vendor.M["name"].S",
          "APRefValue": "$elem.APRef.M["value"].S",
          "APRefName": "$elem.APRef.M["name"].S"
        }#if($foreach.hasNext).#end
        #end
    ]
}
```



- Reuse Responses by defining them in the global `responses` section and reference that definition via `$ref` at the operation level
- Lambda Integrations
  - **dont use custom Lambda** (`non-proxy`) and custom HTTP (non-proxy) Integration type
- HTTP-Integrations
  - **dont use custom HTTP** (`non-proxy`) Integration type
- set Gateway Responses for the integrations using  [x-amazon-apigateway-gateway-responses.gatewayResponse](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-gateway-responses.gatewayResponse.html) extension



#### 4.3.4 Headers

- dont use custom headers, only the official headers (no `X-` prefix or otherwise)
- use the following request headers
  - Accept
  - Accept-Language
  - Cache-Control
  - Content-Type
  - Connection: Keep-Alive
  - Date: Thu, 11 Aug 2016 15:23:13 GMT
  - Keep-Alive: timeout=5, max=1000
  - Last-Modified: Mon, 18 Jul 2016 02:36:04 GMT
- use the following response headers
  - Access-Control-Allow-Origin: *
  - Connection: Keep-Alive
  - Content-Type: text/html; charset=utf-8
  - Date: Mon, 18 Jul 2016 16:06:00 GMT
  - Keep-Alive: timeout=5, max=1000
  - Last-Modified: Mon, 18 Jul 2016 02:36:04 GMT



## 5. Technology Stack

- The following technologies are mandatory and cannot be replaced by any other technology similar or not without our explicit approval:
  - AWS API Gateway as the API layer
  - AWS Cognito
  - AWS Lambda
  - AWS DynamoDB
  - Node.js running in Lambda for the code layer of this API
  - OpenAPI definition/ Swagger
  - postman to check the API
  - HTML
  - CSS-Grid

- the following technologies are not allowed

  - Serverless framework
  - Flexbox
  - CSS-preprocessor
  - Html preprocessor
  - HAML



## 6. Code repository

- we provide a repository named "serverless_api_DynamoDB_EMR"
- Fork the repository and work inside your own account
- Give access to `jprivillaso@gmail.com` and `aerioeus@gmail.com` so we can track your work
- Create a Pull Request and we will review the code
- arrange your code like this

```shell
--api-masterstack
  |__api_contacts.yaml
  |__api_invoice.yaml
  |__ ....
--nodejs
  |__Lambda1
      |__index.js
      |__index.zip
  |__Lambda2
      |__index.js
      |__index.zip
  |__Lambda3
      |__....
--swaggerapis
  |__Invoices_swagger # folder holding all the parts of the swagger file
      |__swagger_Requestbody.yaml
      |__swagger_Responsebody.yaml
      |__swagger_Definitions.yaml
      |__swagger_invoice_main.yaml
  |__.....

--cognito-stack
  |__cognito.yaml
  |__cognito_lambda.yaml
  |__ ccr-nodejs-1.0.0.zip
--DynamoDB-stack
  |__DynamoDB_Pay_Per_use.yaml
--VPC-masterstack
  |__VPC_core.yaml
  |__securitygroups.yaml
--IAM-masterstack
  |__policies.yaml
  |__roles.yaml
  |__Instancegroups.yaml
  |__User.yaml

```



## 7. Provided by us

- Documentation template as reference
- Swagger (OpenAPI) file containing all Pathes for the entire module
- Github Repository to pull from
- further we provide the following Sample stacks:
  - vpc Masterstack
  - IAM Masterstack
  - Cognito stack
  - DynamoDB stack
  - API Masterstack



## 8. Timeline

- Project Start: 01.02.2019
- Project finish:  21.02.2019
- Payment: 3.000 USD upon completing the project according to the conditions laid out in this document
