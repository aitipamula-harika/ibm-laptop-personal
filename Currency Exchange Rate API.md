**Solution Architecture Product Documentation**

**Product Capability**

**Currency Exchange Rate API**

Currency Exchange Rate API - this  API has several operations ,**ExchangeCurrency** operation retrieves the value of a currency when converted from one currency to another currency ,this operation retrieves the data by screen scraping the Green Screen through demand code: CCV2 ,**ExchangeCurrency3** operation retrieves the value of a currency when converted from one currency to another currency, this operation retrieves the data through demand code: 4C@ and **RetrieveExchangeCurrencyHistoricalRate** operation will allow the Application Channel(s) to retrieve historical (i.e. date of issuance of ticket) BSR currency conversion rate. The formula that is used to compute the historical currency conversion is query based.

**Domain & Capability Mapping**

a. L1 – Corporate Finance

b. L2 - Treasury

c. L3 – Currency Exchange Rate 

**Product Details**

Delta IserverId: A0021556

Air4  IserverId: TBD

\| **Block Code Delta** \| FINTRECERT \|

\| **Block Code Air4** \| AVSACFCERT \|

\| **App Criticality** \| Mission Vital\|

\|**Data Classification** \| Confidential \|

\|**Is PCI** \| False \|

\|**Is AIR4** \| True \|

\---

**Dependencies**

**Upstream Dependencies**

| **Operation Name​**                  | **​Upstream Dependencies**     |  DATA API/PSS Generic API |
|-------------------------------------|---------------------------------|-------------------------|
|ExchangeCurrency                    | TPF-ROF(Demand code -4C@) |/rofCurrencyConversion|
| ExchangeCurrency2                    | TPF-ALC(Demand code -CCV) | /alcExchangeCurrency   |
| ExchangeCurrency3                    | TPF-ROF(Demand code -4C@)    | /rof4CcurrencyConversion |
| RetrieveExchangeCurrencyHistoricalRate| Reference DB -CURR_EXCG_RT , CURRENCY |                  |


**Downstream Dependencies**

NA


**Data Management**

FDM Currency Exchange Rate API is going to call TPF Generic API and Currency Data API .So it's a API to API call , there is no direct DB call .

**Data Schema/Table Design**

NA

**Data Initial State & Replication Awareness**

NA

**Data Access Patterns**

NA

**Data Retention & Archival**

NA

**Process flows**

![image info](./asset/FDM-ToolsV2-API-ProcessFlow.png)


**Business Logic / Rules**
NA

**Targeted users of this capability**.

Below Table provides information about consumers/users for each operation. Currency Exchange Rate API is a Private API. For more details, please follow the Logical Architecture section.

| **Operation Name​**                  | **Consumers​** |
|-------------------------------------|---------------|
| ExchangeCurrency | Delta.com Shop 2, Delta.com Trip, Delta.com Trip - Virgin Atlantic, Delta.com Utilities, Delta.com Utilities (except eDocs) - Virgin Atlantic
   |
| ExchangeCurrency2 | Axis, SNAPP   |
| ExchangeCurrency3 | Online Check-In, Online Check-In - Virgin Atlantic, Preferred Channel Web Services, Preferred Channel Web Services - Virgin Atlantic   |
RetrieveExchangeCurrencyHistoricalRate | Axis, SNAPP   |


### Architectural Decisions To Note

| Decision Number | Decision description                                                                                                                                                                                                                                                                                                          | Impact    |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|
| 1               | CurrencyExchangeRate API is a passthrough API ,Hence considering the less complex business rule and small code amount , AWS lambda will be used as target deployment platform for this API                          | No Impact |
| 2               | **CURRENCY and CURRENCY EXCH RT** table will be accessed  using Data access API available on aws| No Impact |
| 3              |We will use latency based routing in multi region active-active deployment , where Route 53 will direct traffic to the resource that provides the fastest response time. This is determined by measuring the latency between your end users and AWS regions. | No Impact |
| 4            | Route 53's automated health checks work well to monitor endpoints and make DNS routing decisions based on the health.
| 5            |However, there are situations where automated health checks and region failover may not suffice. For instance, subtle issues that affect the performance or functionality of your application, but not the availability or response time, may not be detected by Route 53.S3 as global service is used for manual intervention.|No Impact |




## Current State Architecture 

![image info](./asset/FDM-ToolsV2-Deployment.png)

**Conceptual Architecture**

![image info](./asset/FDM-ToolV2-Logical-Model.png)


**Logical Architecture**

![image info](./asset/FDM-ToolV2-Component-Model.png)

1\. Client submits client ID and secret to get Access Token.​

2\. Client gets Access Token for credentials from PingID.​

3\. The internal consumers (on-cloud / on-prem), invoke Currency Exchange Rate API via R53 private hosted zone.

4\. The private hosted zone resolves to internal ALB, which directs the traffic to Private API Gateway with Application/User Access Token.

5\. API Gateway calls Lambda Authorizer to verify tokens. Lambda Authorizer retrieves JWKS from PingID and validates Access Token.​

6\. API Gateway evaluates policy and invokes lambda.

7\. Lambda (microservice) then make a call to TPF Owned Generic API and Data API get the response back.

8\. Cross Region Check and Manual intervention to automatically failover to shut down a region in the case of active-active.



**Security**

1.  **Integrate with Global Authentication**
    -   Use of Ping-Id integration for app-based (CLIENT_ID) authorization check.
    -   Use of IAM based policies to handle invocation of API for guest user scenario.
2.  **Enforce and Manage Least Privilege for All Users**.
    -   Use of IAM roles and policies-based permissions to control   access of AWS platform resources and services.
    -   Developers in the environment will require the following roles:

        1\. For DEV or SBX: DeveloperPowerUser

        2\. For SI or PROD: Deployer

    -   **Private API Gateway:** The consumers call the Private API gateway using PING Access token. The API gateway calls Lambda authorizer to validate the signature of the PING access token and it also validates the ClientID from the access.yaml file stored in the S3 bucket.
    -   **Lambda Microservice**: The microservice is only exposed through the Private API gateway.
-   Other roles that may be applicable for managers or product owners.
    1.  Read-only: Can only see application data in S3.
    2.  View Only: Restricted to only seeing metadata (no application content)
3.  **Use IaC and DevOps Processes Which Include Security Testing and Scanning.**
    -   Use of CloudFormation Script to set up infra.
    -   Use of CloudFormation Script to set up infra-pipeline and app-pipeline.
    -   Infra-pipeline set up with guard rails to validate the infra set up is as per AWS config defined policies.
    -   App-pipeline set up with Vera-code action for security vulnerability findings.
4.  **Adopt Native Encryption at Rest.**
    -   All data at rest will be encrypted.
5.  **Adopt Native Encryption in Motion for All Traffic.**
    -   All data in motion will be encrypted using TLS1.2
6.  **Structure the Use of Cloud Components into a Segmented Network Model.**
    -   Dedicated AWS Account for Currency Exchange Rate API Mission Vital.
    -   AWS lambda instance provisioning in specific VPC / AZ is handled by AWS platform itself.
    -   Use of security groups to allow access of API from on-cloud consumers VPC CIDR.

**Security Groups**

\> Ports and protocols (could also link to IaC)

Below SG will be attached to specific VPC ID (where Microservice is going to be deployed) to allow the traffic from Delta Network CIDR 10.0.0.0/8 (inbound). Lambda will be configured with in the same VPC.

We need to add necessary inbound rules to this SG.

![image info](./asset/SG.png)

**IAM Roles**

1.**Route53HealthCheckFunctionRole**: Attached to Health check lambda to write error logs to CloudWatch.

2.**CurrencyExchangeRateApiGatewayAuthorizerRole**: Attached to API Gateway to invoke lambda function.

3.**ApiGatewayLoggingRole**: Attached to API Gateway to write the logs to CloudWatch.

4.**AuthorizerLambdaFunctionExecutionRole**: Attached to authorizer lambda to get object from S3 access bucket.

5**delegate-admin-CurrencyExchangeRate-deployer-role**: Service role for deploying to AWS from CICD pipeline.

**Performance & Capacity**

**Delta Environment**

| **Operation**                       | **Minimum Transactions per Minute** | **Maximum Transactions per Minute** | **Response Time (PT) (Seconds)** |
|-------------------------------------|-------------------------------------|-------------------------------------|----------------------------------|
| ExchangeCurrency                    | 11                                  | 130                                 | 0.23 sec                         |
| ExchangeCurrency2                    | 11                                  | 130                                 | 0.23 sec                         |
| ExchangeCurrency3                    | 11                                  | 130                                 | 0.23 sec                                                                           |
| RetrieveExchangeCurrencyHistoricalRate  | 11                                  | 130                                 | 0.23 sec                         |

**Virgin Atlantic Environment**

| **Operation**                       | **Minimum Transactions per Minute** | **Maximum Transactions per Minute** | **Response Time (PT) (Seconds)** |
|-------------------------------------|-------------------------------------|-------------------------------------|----------------------------------|
| ExchangeCurrency                    | 11                                  | 130                                 | 0.23 sec                         |
| ExchangeCurrency2                    | 11                                  | 130                                 | 0.23 sec                         |
| ExchangeCurrency3                    | 11                                  | 130                                 | 0.23 sec                         |
|
| RetrieveExchangeCurrencyHistoricalRate| 11                                | 130                                 | 0.23 sec                         |


**Logging, Monitoring & Alerting**.

-   Using CloudWatch, X-Ray for application & AWS service logging. Operational monitoring dashboard/alarms will be built based on various threshold parameter related to error code, response time, throttling using the cloud watch logs.

- Cloud watch logs captured for following resources –

    - Private API Gateway

    - S3 – for Auth Access file

    - Application log

-   CloudTrail for logging, continuously monitor, and retain account activity related to actions across your AWS resources provisioned for the dedicated account.
-   Logs from Authorization checks are logged in the Lambda authorizer log group. Rules can be configured to stream the Authorization failure events to SIEM.
-   Integrate CloudWatch alerts/alarms with PagerDuty.
-   Application team shall be having necessary IAM Read Only role be able to investigate CloudWatch logs in SI and PROD and run custom queries using Log Insight.
