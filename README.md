# A Scalable ETL Platform

A High Level Architectural design of an ETL platform that can integrate with heterogeneous data sources, extract,
transform and call a target API.

# Problem Statement

You are working in a company called TAXMAN which has an API to calculate TAX for a given order/transaction JSON. Your
team’s responsibility is to make customer jobs easier by writing custom integration logic that can pull data from the
customer ERP system and fed it to TAXMAN API. However, you find it difficult to maintain and scale so many custom
integrations. Hence as an architect, your JOB is to develop a low code Integration platform where developers can quickly
and easily build integrations that helps your business to onboard customers faster.

# Assumptions

* This integration platform would be deployed on cloud (Assumed AWS for this design).
* The source data (customer's ERP) is accessible from cloud (no network constraints).
* For a minority of customers who are heavily constrained to expose their data, a push based mechanism (through a simple
  Data Extract pipeline that runs on the customer premises) could also be an option.
* The data transformation logic will be developed by Integration Platform Engineers through a Drag and Drop UI and not
  by the end customers.
* The customers could change the schedule of their data processing through a UI.
* The customers could view the historical jobs, their statuses and download the results.
* Both a UI and REST APIs are available to customers.

# Design Preamble

* **Multi Tenant** - The platform must support multiple tenants/customers.
* **Low code** - Integrating with a new data source should just be plugging in a connector and configuring it specific
  to the customer's data source.
* **Simplicity of Transformations** - The platform must offer various transformation/data cleansing functions out of the
  box to map and massage the data in-flight with low or no code.
* **Full Load and Incremental loads** - The platform must support both "full loads" and "incremental loads".
* **Simpler onboarding** - Onboarding a new customer or a new target system should be a lean process without having to
  write code.
* **Scalable** - The platform must be highly scalable horizontally to cater to the dynamic data workloads.
* **Pull & Push type data acquisitions** - The platform must support Pull (most use cases) and Push (cases where access
  to the data source is not practical).
* **Managed on cloud** - The platform leverages managed services of AWS wherever possible so that there's no manual
  provisioning/management etc to be done.
* **Support for Edge/On Prem** - Cases where the customers need to push the data into the platform (because of various
  constraints and policies etc), a simple scalable pipeline should be deployed on the customer premises to extract the
  data and push it to cloud.
* **Exception Handling** - At various stages of the ETL, errors are possible. Those errors would have to be clearly
  logged and reported to the end customers.
* **Retries** - The platform must have a configurable mechanism to retry the target API call if the call fails.
* **Secure** - The platform should enforce only authorised users can view the output, create schedules and the data
  stored on disk must be encrypted. Communication between components must be over TLS.
* **Availability** - The system must be highly available for the ETL processes to run.
* **Eventual Consistency** - The platform need not guarantee strong consistency as the result may not be available real
  time. It's okay to have a small delay as this is a batch process.
* **On demand/adhoc processing** - In addition to scheduled runs, the platform must also support on demand runs to be
  triggered by the end users (via Management UI or REST API).
* **Leaner costs** - The platform should consume as minimal as it can when it's idle.
* **Dynamic Scaling in/out** - The platform must be able to cater to scale in/out dynamically based on workloads.
* **Extensibility** - The platform must support custom connectors to be written and deployed alongside the existing
  connectors.
* **Logging** - The platform must have standardised logging to troubleshoot and analyse the processes.
* **Monitoring** - The platform must have monitoring components in place to constantly observe the metrics and define
  rules against thresholds as necessary.

# High Level Architecture

![High Level Architecture](./IntegrationPlatform.png?raw=true "High Level Architecture")

# Description

This platform has majority of it's components managed in the cloud by AWS. I'll describe the main components and their
purposes.

* **AWS Glue** - The Heart of the platform that offers a "studio based approach" to define the ETL in a drag-and-drop
  GUI.
* **AWS SQS** - A messaging Queue managed by AWS that enables decoupling and scalable asynchronous communication between
  various components in the platform.
* **AWS SNS** - A pub/sub managed by AWS that enables decoupling and asynchronous notifications between various
  components in the platform.
* **AWS S3*** - A distributed object store to store the raw data, transformed data, error reports and the final output.
* **MongoDB (AWS DocumentDB)** - A NOSQL Document store to store the output data (after calling the TAXMAN API).
* **Postgres (AWS RDS for ClientDB)** - A postgres DB to store the client details (including their job access limits,
  schedules, audit logs etc).
* **Postgres (AWS RDS for Scheduler DB)** - A postgres DB to store the scheduler details using Quartz (and to lock the
  scheduler to avoid running on multiple instances).
* **AWS Lambda** - A serverless code that reacts to a trigger message in the queue and runs the corresponding pipeline
  specified in the payload of the message.
* **AWS Application Load Balancer** - Loadbalancer to balance the load of incoming requests to the microservices.

# Flow

## Client creation:

* Admin user logs into the UI.
* Creates a new client via the client-operations-service.
* Client is created in the system.

## Client ETL Definition:

* A ETL/Data Engineer logs into AWS account.
* Creates a Data Catalogue for the client's data source and creates a Glue pipeline.
* Logs into the integration-platform GUI and creates a schedule for this pipeline (CRON).
* This is done via clients-operations-service.
* This configuration is immediately placed in SNS_Config_Change_Broadcast topic and subscribed by scheduler microservice
  which creates a schedule using Quartz.
* This configuration is also subscribed by taxcalculator-service for getting the values for retrying policies and output
  notification mechanisms.

## Client ETL Execution (Pull Data Mechanism):

* Scheduler service emits a trigger event into SQS_Glue_Pipeline_TriggerQ at the configured time in cron.
* This event is consumed by a Lambda function (generic function written only once) and has the necessary details in the
  payload to invoke the correct Glue Pipeline.
* Note that the Pipeline can only be triggered through this lambda function (in an automated context). This lambda
  function provides an unified interface to execute the pipeline.
* Pipeline Extracts the data and applies transformation rules and dumps the error records and successfully transformed
  records into S3.
* Every successfully transformed record would be a JSON and will result in an event emitted by S3 to
  SQSDataTransformedEventQ.
* This event is consumed by a spring boot taxcalculator-service which will call the TAXMAN api and save the results in
  MongoDB.
* taxcalculator-service can be built using spring-webflux which is built on the Netty/Reactor stack for better
  throughput. This is mainly because this service is primarily IO bound (calling a API and Saving the results), using a
  non-blocking IO can yield a better throughput than thread-per-request model.
* Additionally, (depending upon the configuration), the result is streamed into SQS_TaxCalculatedEventQ and S3 output
  storage.
* If for any reason the HTTP API call fails, it is placed in a SQS_Retry_RequestsQ and retried until a configured no of
  attempts.
* The TaxResultsDB has read replicas which is then read by tax-output-api-service.
* tax-output-api-service is a springboot based microservice to expose the tax calculated output results to end users in
  various formats.
* For clients who need results in streaming mode, websocket is leveraged.

## Client ETL Execution (Push Data Mechanism):

* In a minority of situations where the AWS Glue pipeline isn't able to access the client's data source, we run the
  extraction process on the client's premises.
* This is done using Apache NiFi (a scalable ETL framework).
* The NiFi template is written by the ETL/Data Engineer and deployed on client's premises on a NiFi cluster.
* The NiFi pipeline extracts the data and pushes directly to the S3 bucket Raw Storage.
* From there, the Glue Pipeline picks it up and executes it as usual.

## Tech Stack:

| Tech/Framework | Purpose                                                         |
|----------------|-----------------------------------------------------------------|
| AWS            | The overall platform on which this platform is deployed as SaaS |
| AWS Glue       | Managed ETL Service                                             |
| AWS S3         | Object storage to store input/output/errors                     | 
| AWS SQS         | Message Queues for decoupling and async communication                   | 
| AWS SNS         | Pub/Sub                   | 
| AWS DocumentDB         | Storing the output data                   | 
| AWS RDS (Postgres)         | Storing the client onboarding metadata                   | 
| AWS Lambda          | Trigger the Pipeline                  | 
| SpringBoot          | Java/Kotlin based microservices framework (reactive and non-reactive)                  | 
| Quartz          | Distributed scheduler                  | 
| EC2 ASG          | EC Autoscaling Group for dynamic scaling based on workloads                  | 
| ReactJS          | For the integration-platform-ui                  | 
| Apache NiFi          | For on-premise/edge data extraction and push to cloud.                  | 







