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
* For a minority of customers who are heavily constrained to expose their data a push based mechanism (through a simple Data Extract pipeline that runs on the customer premises) could also be an option.
* The data transformation logic will be developed by Integration Platform Engineers through a Drag and Drop UI and not by the end customers.
* The customers could change the schedule of their data processing through a UI.
* The customers could view the historical jobs, their statuses and download the results.
* Both a UI and REST APIs are available to customers.

# Design Preamble
* **Low code** - Integrating with a new data source should just be plugging in a connector and configuring it specific to the customer's data source.
* **Simplicity of Transformations** - The platform must offer various transformation/data cleansing functions out of the box to map and massage the data in-flight with low or no code.
* **Full Load and Incremental loads** - The platform must support both "full loads" and "incremental loads".
* **Simpler onboarding** - Onboarding a new customer or a new target system should be a lean process without having to write code.
* **Scalable** - The platform must be highly scalable horizontally to cater to the dynamic data workloads.
* **Pull & Push type data acquisitions** - The platform must support Pull (most use cases) and Push (cases where access to the data source is not practical).
* **Managed on cloud** - The platform leverages managed services of AWS wherever possible so that there's no manual provisioning/management etc to be done.
* **Support for Edge/On Prem** - Cases where the customers need to push the data into the platform (because of various constraints and policies etc), a simple scalable pipeline should be deployed on the customer premises to extract the data and push it to cloud.
* **Exception Handling** - At various stages of the ETL, errors are possible. Those errors would have to be clearly logged and reported to the end customers.
* **Secure** - The platform should enforce only authorised users can view the output, create schedules and the data stored on disk must be encrypted. Communication between components must be over TLS.
* **Availability** - The system must be highly available for the ETL processes to run.
* **Eventual Consistency** - The platform need not guarantee strong consistency as the result may not be available real time. It's okay to have a small delay as this is a batch process.
* **On demand/adhoc processing** - In addition to scheduled runs, the platform must also support on demand runs to be triggered by the end users (via Management UI or REST API).
* **Leaner costs** - The platform should consume as minimal as it can when it's idle.
* **Dynamic Scaling in/out** - The platform must be able to cater to scale in/out dynamically based on workloads.
* **Extensibility** - The platform must support custom connectors to be written and deployed alongside the existing connectors.

# High Level Architecture
![High Level Architecture](./IntegrationPlatform.png?raw=true "High Level Architecture")










