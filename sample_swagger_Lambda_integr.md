# Sample stack DynamoDB Lambda Integration



``` yaml
# IMPORTANT NOTE!!!
# IT IS FORBIDDEN TO USE INTRINSIC FUNCTIONS IN ITS SHORT WAY INSIDE THIS SWAGGER FILE
# THIS IS AN AWS LIMITATION BY THE TIME WE'RE CREATING THIS API, SO PLEASE AVOID USING
# THINGS LIKE !Ref and Instead use Ref
#
swagger: "2.0"
info:
  version: 1.0.0
  title: Contact Manager - API
  description: >
    This API is intented to enable external users to get information for the resource contacts

# Specifies a request validator, by referencing a request_validator_name
x-amazon-apigateway-request-validators:
  basic:
    validateRequestBody: true
    validateRequestParameters: true
  params-only:
    validateRequestBody: false
    validateRequestParameters: true

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

# API can understand the following media types
# Note that consumes only affects operations with a request body, such as POST, PUT and PATCH
# It is ignored for bodiless operations like GET
consumes:
  - application/json
  - application/xml
  - text/plain; charset=utf-8
  - text/html
  - application/pdf
# API can respond with various media types
produces:
  - application/json
  - application/xml
  - image/png
  - image/gif
  - image/jpeg

x-amazon-apigateway-request-validator: basic

paths:
  /invoices:
    get:
      produces:
      - "application/json"
      responses:
        200:
          description: "200 response"
          schema:
            $ref: "#/definitions/Empty"
      x-amazon-apigateway-integration:
       uri:
          Fn::Join:
          - ''
          - - 'arn:aws:apigateway:'
            - Ref: AWS::Region
            - ":lambda:path/2015-03-31/functions/"
            - Fn::GetAtt:
              - LambdaScanDdb
              - Arn
            - "/invocations"
        responses:
          default:
            statusCode: "200"
        passthroughBehavior: "when_no_match"
        httpMethod: "POST"
        contentHandling: "CONVERT_TO_TEXT"
        type: "aws"
    post:
      produces:
      - "application/json"
      responses:
        200:
          description: "200 response"
          schema:
            $ref: "#/definitions/Empty"
      x-amazon-apigateway-integration:
       uri:
          Fn::Join:
          - ''
          - - 'arn:aws:apigateway:'
            - Ref: AWS::Region
            - ":lambda:path/2015-03-31/functions/"
            - Fn::GetAtt:
              - LambdaPutDdb
              - Arn
            - "/invocations"
        responses:
          default:
            statusCode: "200"
        passthroughBehavior: "when_no_match"
        httpMethod: "POST"
        contentHandling: "CONVERT_TO_TEXT"
        type: "aws"
    options:
      consumes:
      - "application/json"
      produces:
      - "application/json"
      responses:
        200:
          description: "200 response"
          schema:
            $ref: "#/definitions/Empty"
          headers:
            Access-Control-Allow-Origin:
              type: "string"
            Access-Control-Allow-Methods:
              type: "string"
            Access-Control-Allow-Headers:
              type: "string"
      x-amazon-apigateway-integration:
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'DELETE,GET,HEAD,OPTIONS,PATCH,POST,PUT'"
              method.response.header.Access-Control-Allow-Headers: "'Content-Type,Authorization,X-Amz-Date,X-Api-Key,X-Amz-Security-Token'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
        requestTemplates:
          application/json: "{\"statusCode\": 200}"
        passthroughBehavior: "when_no_match"
        type: "mock"
  /{InvoiceId}:
    get:
      produces:
      - "application/json"
      parameters:
      - name: "InvoiceId"
        in: "path"
        required: true
        type: "string"
      responses:
        200:
          description: "200 response"
          schema:
            $ref: "#/definitions/Empty"
      x-amazon-apigateway-integration:
       uri:
          Fn::Join:
          - ''
          - - 'arn:aws:apigateway:'
            - Ref: AWS::Region
            - ":lambda:path/2015-03-31/functions/"
            - Fn::GetAtt:
              - LambdaQueryDdb
              - Arn
            - "/invocations"
        responses:
          default:
            statusCode: "200"
        passthroughBehavior: "when_no_match"
        httpMethod: "POST"
        contentHandling: "CONVERT_TO_TEXT"
        type: "aws"
    options:
      consumes:
      - "application/json"
      produces:
      - "application/json"
      responses:
        200:
          description: "200 response"
          schema:
            $ref: "#/definitions/Empty"
          headers:
            Access-Control-Allow-Origin:
              type: "string"
            Access-Control-Allow-Methods:
              type: "string"
            Access-Control-Allow-Headers:
              type: "string"
      x-amazon-apigateway-integration:
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'DELETE,GET,HEAD,OPTIONS,PATCH,POST,PUT'"
              method.response.header.Access-Control-Allow-Headers: "'Content-Type,Authorization,X-Amz-Date,X-Api-Key,X-Amz-Security-Token'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
        requestTemplates:
          application/json: "{\"statusCode\": 200}"
        passthroughBehavior: "when_no_match"
        type: "mock"
definitions:
  Empty:
    type: "object"
    title: "Empty Schema"
```

