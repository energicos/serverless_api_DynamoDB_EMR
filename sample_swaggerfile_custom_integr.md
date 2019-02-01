# Sample stack DynamoDB custom -non proxy Integration

```yaml
#
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
    This API is intented to enable external users to get information for the resource addresses

# Specifies a request validator, by referencing a request_validator_name
x-amazon-apigateway-request-validators:
  basic:
    validateRequestBody: true
    validateRequestParameters: true
  params-only:
    validateRequestBody: false
    validateRequestParameters: true
x-amazon-apigateway-request-validator: basic

# Specifies the source to receive an API key
x-amazon-apigateway-api-key-source: HEADER

# since we set up the Cognito Authorizer in the API stack itself we dont need to import a COGNITO_USER_POOLS authorizer with an OpenAPI definition file
securityDefinitions:
  addressesauthorizer:
    type: apiKey # Required and the value must be "apiKey" for an API Gateway API.
    name: Authorization # The name of the header containing the authorization token
    in: header # Required and the value must be "header" for an API Gateway API.
    x-amazon-apigateway-authtype: cognito_user_pools # Specifies the authorization mechanism for the client.
    x-amazon-apigateway-authorizer: # An API Gateway Lambda authorizer definition
      type: cognito_user_pools # Required property and the value must "token"
      providerARNs:
      - Fn::ImportValue: { "Fn::Sub" : "${EnvironmentName}-${Project}-UserPoolARN" }

schemes: [http, https]

paths:
  /invoices:
    post:
      consumes:
      - "application/json"
      produces:
      - "application/json"
      parameters:
      - in: "body"
        name: "Requestbody"
        required: true
        schema:
          $ref: "#/definitions/Requestbody"
      responses:
        200:
          description: "200 response"
          schema:
            $ref: "#/definitions/Empty"
      x-amazon-apigateway-integration:
        credentials: { "Ref" : "DynamoDBRoleARN" }
        uri: {"Fn::Sub" : "arn:aws:apigateway:${AWS::Region}:dynamodb:action/PutItem"}
        responses:
          default:
            statusCode: "200"
        requestTemplates:
          application/json: >-
                #set($inputRoot = $input.path('$'))
                {
                    "TableName": "Invoicelogs",
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
        passthroughBehavior: "when_no_templates"
        httpMethod: "POST"
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
  /invoices/{InvoiceId}:
    get:
      consumes:
      - "application/json"
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
            $ref: "#/definitions/Requestbody"
      x-amazon-apigateway-integration:
        credentials: "arn:aws:iam::300746241447:role/APIDatamappingDynamoDB"
        uri: "arn:aws:apigateway:eu-west-1:dynamodb:action/Query"
        responses:
          default:
            statusCode: "200"
            responseTemplates:
              application/json: >-
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
        requestTemplates:
          application/json: >-
                {
                    "TableName": "Invoicelogs",
                    "IndexName": "InvoiceId-Status-Index",
                    "KeyConditionExpression": "InvoiceId = :v1",
                    "ExpressionAttributeValues": {
                        ":v1": {
                            "S": "$input.params('InvoiceId')"
                        }
                    }
                }
        passthroughBehavior: "when_no_templates"
        httpMethod: "POST"
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
  Requestbody:
    type: "object"
    properties:
      DueDate:
        type: "string"
      Balance:
        type: "number"
      InvoiceId:
        type: "string"
      Status:
        type: "string"
      Line:
        type: "array"
        items:
          type: "object"
          properties:
            Description:
              type: "string"
            Amount:
              type: "integer"
            DetailType:
              type: "string"
            ExpenseDetail:
              type: "object"
              properties:
                Customer:
                  type: "object"
                  properties:
                    value:
                      type: "string"
                    name:
                      type: "string"
                Ref:
                  type: "object"
                  properties:
                    value:
                      type: "string"
                    name:
                      type: "string"
                Account:
                  type: "object"
                  properties:
                    value:
                      type: "string"
                    name:
                      type: "string"
                LineStatus:
                  type: "string"
      Vendor:
        type: "object"
        properties:
          value:
            type: "string"
          name:
            type: "string"
      APRef:
        type: "object"
        properties:
          value:
            type: "string"
          name:
            type: "string"
      TotalAmt:
        type: "number"
    title: "InvoiceInputModel"


```

