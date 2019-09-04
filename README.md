# Elastic DevOps agent lifecycle management

The Elastic Agent Lifecycle Management for DevOps Server is intended to create or remove agents in a dynamic way. Currently, DevOps Server only supports static agent pool management, i.e., users have to provision Azure resources, install agent package, configure agents, unregister agents, deprovision Azure resources manually. In production environment, there may be many dev teams, internal and external ones, working together. It might be necessary to isolate the build/release jobs of different teams into different agent pools. An automation tool is needed to orchestrate the creating and removing DevOps agents dynamically, which is how Elastic Agent Lifecycle Management for DevOps Server can help.

## Mechanism
The Elastic Agent Lifecycle Management currently supports three APIs:

* API for creating agents

By invoking http://<application_url>/createagents/, you can trigger the application to create agents according to the  agent quota specified for each agent pool.

The application will create agents in a parallel and asynchronous manner.

* API for reminding the application an agent is cleaned

Each agent is executed with the --once option, which ensures that the agent will be unregistered after running one job. Afterwards, the agent_run.sh shall send a request to http://<application_url>/agentcleaned?agentName=<agentName>&poolName=<poolName> to clean up the agent profile. Note that the related Azure resources won't be deprovisioned at this time.

* API for cleaning Azure resources

Users can send a request to http://cleanresources trigger this API to clean up Azure resources associated with the cleaned agents. The application will clean up the Azure resources in an asynchronous manner.

## Prerequisites

### Installing Java and Maven
1. Download and install Java 8.0
2. Set JAVA_HOME environment variable
3. Download Apache Maven 3.6.x and install Maven
4. Add the path to java and mvn in your environment variable

### Provisioning Azure Database for MySQL
1.	Provision Azure Database for MySQL in your subscription. It’s recommended to provision MySQL of version 5.7, not 8.0+.
2.	Configure connection security of MySQL service
    a.	Press “Connection security” 
    b.	Press “Add client IP” button on the top and edit the start IP address and end IP address to allow your IP addresses to access MySQL service

### Creating database and tables in MySQL
1.	Download mysqlsh from https://dev.mysql.com/downloads/shell/
2.	Start mysqlsh in local console
3.	In mysqlsh command, input following commands:
    a.	\connect <userName>%40<serverName>:<password>@<serverName>.mysql.database.chinacloudapi.cn
    b.	\sql
    c.	create database <dbName>;
    d.	use <dbName>;
    e.	create agents table with following command:

```sql
create table agents(agentName varchar(50) not null primary key, poolName varchar(50) not null, resourceGroupName varchar(50) not null, vmName varchar(50) not null);
```

    f.	create cleanrequests table
```sql
create table cleanrequests(agentName varchar(50) not null primary key, poolName varchar(50) not null, vmName varchar(50) not null, resourceGroupName varchar(50) not null, requestTime varchar(35) not null);
```
    g.	show tables;

### Uploading scripts and DevOps agent package
1. Ensure the following shell scripts are uploaded to an Azure storage account:
    - agent_install.sh
    - agent_run.sh
2. Ensure the DevOps agent package is uploaded to Azure storage account:
3. Generate Blob SAS token and URL for agent_install.sh, agent_run.sh, and agent package
4. Edit the ARM template.json file by changing the following parameter default value to the Blob SAS URL copied from above:
    -	Changing “customScriptUrl” default value to the SAS URL of agent_install.sh
    -	Changing “agentRunScriptUrl” default value to the SAS URL of agent_run.sh
    -	Changing “agentPackageUrl” default value to the SAS URL of agent package

### Preparing ARM templates
1. Editing template.json according to your Azure environment and uploading it to Azure storage account
2. Generating Blob SAS token and URL for template.json and note it
3. Creating and editing parameters.json for each Agent pool, for example, linux_parameters.json for Agent pool “Linux”. Uploading the parameters.json files to Azure storage account.
4. Generating Blob SAS token and URL for the parameters.json and note them

### Creating service principle
1. Browse Azure portal and open Azure Active Directory
2. Create a service principle in AAD portal and note its
    - Client Id
    - Tenant Id
    - Client secret

## Deployment (local mode)
1. Edit agent_pool_config.json
   - Update “armTemplateUrl” property with the Blob SAS URL of ARM template
   - Edit “subscriptionId” to that of your subscription
   - Edit “clientId” to that of your service principle
   - Edit “tenantId” to that of your service principle
   - Edit “clientKey” to that of your client secret
   - For each agent pool
     - Change the pool name
     - Change the agent quota for the pool, which is the number of active agents to be deployed
     - Change “parameterUrl” to the Blob SAS URL of the parameters.json of the agent pool
2. You can place agent_pool_config.json to any directory in your local computer
3. Edit application.properties file 
   - Edit “logging.path” to a directory to contain your application log and create the directory
   - Edit “spring.datasource.url” to jdbc:mysql://<serverName>.mysql.database.chinacloudapi.cn:3306/<dbName>?serverTimezone=UTC&useSSL=true&requireSSL=false&verifyServerCertificate=true
   - Edit “spring.datasource.username” to <userName>@<serverName>
   - Edit “spring.datasource.password” to your password of MySQL
   - Edit “config.agentpool” to the file path of agent_pool_config.json where it’s placed
   - Edit “devops.server.url” to the URL of DevOps server cluster
   - Edit “devops.server.port” to 80 for HTTP and 443 for HTTPS
   - Edit “devops.server.protocol” to http or https
   - Edit “devops.admin.name” to the administrator username of DevOps Server
   - Edit “devops.admin.password” to the administrator password of DevOps Server
4. Use Maven to compile and package the application into jar

## Accessing API
1. Open browser and navigate to http://localhost:8080/createagents to trigger application to deploy Azure resources and agents
2. Navigate to the target resource group in Azure portal to make sure all the Azure resources are provisioned
3. Navigate to http://localhost:8080/agentcleaned?agentName=<agentName>&poolName=<poolName> to trigger application to record an agent to be undeployed
4. Navigate to http://localhost:8080/cleanresources to trigger application to clean up Azure resources
