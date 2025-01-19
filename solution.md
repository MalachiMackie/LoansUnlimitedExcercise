# Loans Unlimited

Loans Unlimited is a company where customers can apply for loans though a web page. the loan applications get reviewed and accepted or rejected by a loan provider agent after a credit check has been performed. The loan provider agent reviews applications though a separate web interface

## Solution Requirements
- customer sumbits applications through web page
- credit check through ACME Credit Check Ltd API
  - failed credit fails loan application and notifies via email
- customer can submit multiple loan applications
- customer can view current and previous loan applications through web page
- loan provider can view all loan applications
- loan provider approve loans that have successful credit check
- loan provider must use a web interface that is not directly accessible over the internet
- approved loan notify customer via email
- each day at 9pm a file containing all *loan credit payments* for approved loans will be generated and placed on the SFTP server of ACME Bank ltd
- AUDIT HISTORY

## Interactions

### Customer Interactions
- submit application though web ui
- view loan applications

### Loan Agent Interactions
- view all loan applications 
- approve loan application (application must have had successful credit check)
  - triggers email to customer

### System Interactions
- export all loan credit payments for approved loans to ACME Bank Ltd SFTP server

## Data Model

See `./LoansUnlimited-DataModel.png` for data model 

## Components

See `./LoansUnlimited-ComponentDiagram.png` for how components interact

### Customer Web Interface

The customer web interface has two main pieces of functionality. It will have a form for applying for a loan. For a customer's first loan with Loans Unlimited, this will also serve as a sign up form.
This form will be linked to from a main static company website. Once someone is a customer of Loans Unlimited, they will have login credentials to login to the customer portal.
Here they will be able to view their current loans (loan balance, application status, etc).

#### Technology

The customer portal will use Server Side Rendering (SSR) with mvc along with client side angular for the limited amount of interactivity.
Using SSR lets a dotnet team build quickly with a language they know. It limits the download bundle size of the site, ensuring fast loading for customers.
Using client side angular for interactivity keeps open the option of expanding functionality in the future. Using purely SSR would limit the interactivity of the site in the future

##### Tradeoffs
- Using mvc and angular requires either having devs that know both .Net and Js/Ts/Angular, or having both frontend and backend devs.
- Not using an angular SPA (Single Page Application) means there is a level of complexity with the boundry between SSR And Client Side Rendering.
- All of the initial requirements are possible with purely SSR, and so would have reduced the complexity of the solution at the cost of reduced expandability.

### Customer Web API

The customer web api needs to be able to accept and process a customer's loan application, register a customer with login credentials, and serve a customer's list of loans

#### Technology

The customer web api will be built using .Net 8 as it is the latest LTS release of .Net.
It will use Entity Framework (EF) Core to connect to the database as it provides a fast and easy developer experience and the complexity of database queries will be low considering the domain.
The code will be architected using a very simple clean code architecture, keeping database concerns out of the application layer.

##### Tradeoffs

See "Choosing Monolith vs separate api's" for tradeoffs of separate services between customer api and loan provider agent api
Keeping to the LTS release of .Net means slower uptake on each release's performance and language improvements, but it allows Microsoft to iron out any teething issues before upgrading versions
Using EF Core incures some performance penalties at the cost of developer experience. For a simple domain, EF Core is super simple to use and setup, but as a domain becomes more complex, EF Core configuration can become very complex as well.

### Credit Check Response Lambda

.Net 8 lambda for the credit check service to send credit check response to. This will be exposed through an API Gateway endpoint

### Database

The database will be an RDS managed MySql instance. MySql provides a good balance between cost, functionality and number of developers with MySql experience.
Each table will have change tracking triggers that will copy values to an audit table. this will provide an audit history of all changes done to any table.
Database will be configured to encrypt at rest so if a database backup is leaked or someone gets access to the file system, no information will be exposed.
SSL will also be required for any applications to connect to the database, ensuring data is encrypted as its sent between the database any any applications

### Loan Provider Agent Web Interface 

The loan provider agent web interface needs to authenticate agents so they can view and approve loan applications. It needs to not be accessible over the internet

#### Technology

This web interface will be an angular SPA. Loan provider agents will load it through a private AWS [S3 Gateway Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)
so it is not available publically.

##### Tradeoffs

SPA's are generally slower to load. This is fine as this is an internal application not for public use, and will be cached.
Alternative would be building something like an Electron app. To do this, you either have to have the api accessible to the internet and rely on authentication, or have the private network link anyway. It also makes deployment/installation management more complicated.

### Loan Provider Agent API

This api needs to access customers' loan application and let loan provider agents approve them.

#### Technology

It will also be a .Net 8 api, but will be a separate application from the customer web api to ensure the ability to approve loans is not at all accessible over the internet.
It will be hosted has an ecs container within an AWS VPC.
It needs to access the same database that the customer web application does, so a version column will be used in each table to ensure the most up to date data is being viewed/acted on

##### Tradeoffs

See "Choosing Monolith vs separate api's" for tradeoffs of separate services between customer api and loan provider agent api
Hosting this separately from the customer web api means maintaining another application and deployment

### Queues

AWS SQS queues will be used to handle any asynchronous actions within the applications

### Email Queue Handler

This will be a .Net 8 lambda registered as an sqs queue handler so that the success or failure of an email to send does not impact the rest of the request or application.
Any failed email queue messages will also be forwarded to a separate email failure queue for easy monitoring

### Bank File Exporter

Every evening at 9pm, we need to load all newly accepted loan applications into a file to upload to an SFPT server at ACME Bank Ltd

#### Technology

This will be a timed .Net 8 lambda set to trigger at 9pm and 9:30pm. If the first run succeeds, the second trigger won't do anything. But the second trigger allows us to retry the upload. If the retry fails, then the failure will be logged so it can be alerted on

### Other AWS Components

#### Networking

All the aws components will be hosted within a VPC, with only the customer web api being accessible publically
Uses AWS Direct Connect to connect on prem network to aws vpc to ensure loan provider agent application is not available publically

#### Logging

Cloudwatch will be configured with a dashboard to monitor application status as well as logging. These metrics can then have alerts configured so if a critical error occurs, someone can be notified

## Testing

Each of the .net components will have unit tests and Integration tests. We can ensure test coverage through automated tools like sonarqube or codacy. High level end to end tests will also be written to ensure integration of the whole system. These tests can be run against containers with a real database to ensure we are testing against components as close to production as possible

## Deployment plan

All changes will trigger github action pipelines to build applications, run tests and deploy infrastructure and applications.

### Infrastructure

The aws infrastructure will be managed through Infrastructure as code using AWS CDK. This allows infrastructure to be version managed and deployed automatically

### Loan Provider Agent Web Interface

This will be deployed to the s3 bucket that is configured behind the vpc s3 endpoint

### .Net applications

All the .Net applications will be built into docker containers and deployed to either lambda or ecs depending on the application

## Assumptions
- Network loan provider agent is connecting from is secured properly. Does not allow work from home for loan provider agents unless secure vpn into network is used.
- There is no "Task Assignment" aspect for the loan provider agents. It's just a list of loans where each agent picks one and does it. Possible future enhancement
- Credit check service is asynchronous. They accept a webhook configuration to respond to us.
- There will be a "super user" that can register loan provider agents to access the system
- Integration with bank to update loan balance is out of scope

### Choosing Monolith vs separate api's
two distinct parts to the system: customer api and loan provider agent api
separate ui's
want to minimize possibility of security vulnrability allowing loan approvals from customer ui/api

#### Monolith
pros:
- only one system interacting with the database (no chance of db model getting out of sync/date) 
- simplicity

cons:
- if customer web service needs scaling up, then the loan provider agent service must also be scaled up
- more dev discipline needed to now allow bleeding of one sub system into the other
- potential for security vulnrability allowing a loan to be approved through the customer api (open to the internet)

#### Separate services
pros:
- easier scaling
- loan provider agent system can remain not exposed to the internet
- cleaner line between systems

cons:
- multiple systems interacting with the database (potential of data overwrites or db model out of sync)
- more complex ci/cd

thoughts:
could still use mono repo
