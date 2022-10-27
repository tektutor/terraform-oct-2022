# Day 5 - Terraform CI/CD Pipeline in Azure

## What is DevOps?
- is combination of process and tools 
- to confidently release frequent releases to end customer without any defects
- to speed up faster time to market
- Developers
  - automates the Unit/Integrating testing using Unit Test Frameworks
  - using Junit/TestNg, Mockito - Java
  - Using NUnit/Moq - C#
  - Using GoogleTest/GoogleMock - C/C++
- QA
  - automates the end-2-end functional test, component test, API test, smoke test, regression test, performance, stress and load test,etc
  - using Selenium, BDD Frameworks like Cucumber, Specflow, Jasmine, Karma, etc.,
- Operations Team ( System Administrators, DevOps Engineers )
  - automate Infrastructure provisioning ( Docker, Terraform, etc .,)
  - automate software installation using Ansible

## What is Continuous Integration?
- logically completed code should be integrated several times a day
- developers should add Unit & Integration Test cases as part of the code they are integrating
- each time code is committed into source control repository, Jenkins or similar CI/CD servers will take latest code and build and test and share the build report
- if the code committed let to build failures, it means some test cases failed, which is good. In other words, your automated test cases found a bug

## What is Continuous Deployment?
- each time code is pushed/integrated in Source Control repository, the code is build and tested
- the tested application binaries can be deployed automatically to QA environment for further manual/automated testing

## What is Continuous Delivery?
- the QA tested binaries will be automatically delived to customer's environment
- the customer can review the binary if everything is working as per the requirement
- in certain cases, the binaries will deployed onto live prod environment if the organization's DevOps process is so matured that they are confident to make the product live based the build/test report

## What are Azure DevOps Tools?
- For your organization you can create an Organization within Azure Cloud
- The Digital Origanization created within Azure Cloud supports the below
  - Board


## Use this repository for Azure pipeline
<pre>
https://github.com/tektutor/hello-spring-boot.git
</pre>

## What is an Azure Pipeline?
<pre>
# Starter pipeline
# Start with a minimal pipeline that you can customize to build and deploy your code.
# Add steps that build, run tests, deploy, and more:
# https://aka.ms/yaml

trigger:
- main

pool:
  vmImage: ubuntu-latest

steps:
- script: mvn clean compile
  displayName: 'Build the Spring Boot Microservice application code'

- script: mvn test
  displayName: 'Unit Test application binaries'

- script: mvn package
  displayName: 'Unit Test application binaries'

</pre>


## Lab - Creating an Azure Pipeline that runs Terraform scripts to create azure resources

The first step towards this process involves authenticating using our azure credentials.  As we are about to automate this process, we need to create a service principle i.e Register our application via Azure Active Directory.

```
az ad sp create-for-rbac -n "TekTutor-Terraform" --role="Contributor" --scopes="/subscriptions/<subscription-id>"

```

Expected output
<pre>
(jegan@tektutor.org)<b>az ad sp create-for-rbac -n "TekTutor-Terraform" --role="Contributor" --scopes="/subscriptions/<your-azure-subscription-id>"</b>
Creating 'Contributor' role assignment under scope '/subscriptions/c0a81cd4-e4ff-4f6a-a613-0d70aced5676'
The output includes credentials that you must protect. Be sure that you do not include these credentials in your code or check the credentials into your source control. For more information, see https://aka.ms/azadsp-cli
{
  "appId": "<your-app-id>",
  "displayName": "TekTutor-Terraform",
  "password": "<your-password>",
  "tenant": "<your-tenant-id>"
}
</pre>

Now you need to edit the providers.tf with the below details.  To need to fillup the details below based on above output.
<pre>
terraform {
  required_version = ">=0.12"

  required_providers {
    azurerm = { 
        source = "hashicorp/azurerm"
        version = "~>2.0"
    }   
    random = { 
        source = "hashicorp/random"
        version = "~>3.0"
    }   

    tls = { 
        source = "hashicorp/tls"
        version = "~>4.0"
    }   
 }
}

provider "azurerm" {
  features {}
  subscription_id = "<your-azure-subscription-id-goes-here>"
  client_id = "you-appId-goes-here"
  client_secret = "your-application-password-goes-here"
  tenant_id = "your-application-tenant-id-goes-here"
}
</pre>

## Creating a self-hosted Docker Agent to use in your Azure pipeline

Create a Dockerfile with below content
```
FROM mcr.microsoft.com/windows/servercore:ltsc2019

WORKDIR /azp

COPY start.ps1 .

CMD powershell .\start.ps1
```

The start.ps1 file with below content
```
if (-not (Test-Path Env:AZP_URL)) {
  Write-Error "error: missing AZP_URL environment variable"
  exit 1
}

if (-not (Test-Path Env:AZP_TOKEN_FILE)) {
  if (-not (Test-Path Env:AZP_TOKEN)) {
    Write-Error "error: missing AZP_TOKEN environment variable"
    exit 1
  }

  $Env:AZP_TOKEN_FILE = "\azp\.token"
  $Env:AZP_TOKEN | Out-File -FilePath $Env:AZP_TOKEN_FILE
}

Remove-Item Env:AZP_TOKEN

if ((Test-Path Env:AZP_WORK) -and -not (Test-Path $Env:AZP_WORK)) {
  New-Item $Env:AZP_WORK -ItemType directory | Out-Null
}

New-Item "\azp\agent" -ItemType directory | Out-Null

# Let the agent ignore the token env variables
$Env:VSO_AGENT_IGNORE = "AZP_TOKEN,AZP_TOKEN_FILE"

Set-Location agent

Write-Host "1. Determining matching Azure Pipelines agent..." -ForegroundColor Cyan

$base64AuthInfo = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$(Get-Content ${Env:AZP_TOKEN_FILE})"))
$package = Invoke-RestMethod -Headers @{Authorization=("Basic $base64AuthInfo")} "$(${Env:AZP_URL})/_apis/distributedtask/packages/agent?platform=win-x64&`$top=1"
$packageUrl = $package[0].Value.downloadUrl

Write-Host $packageUrl

Write-Host "2. Downloading and installing Azure Pipelines agent..." -ForegroundColor Cyan

$wc = New-Object System.Net.WebClient
$wc.DownloadFile($packageUrl, "$(Get-Location)\agent.zip")

Expand-Archive -Path "agent.zip" -DestinationPath "\azp\agent"

try
{
  Write-Host "3. Configuring Azure Pipelines agent..." -ForegroundColor Cyan

  .\config.cmd --unattended `
    --agent "$(if (Test-Path Env:AZP_AGENT_NAME) { ${Env:AZP_AGENT_NAME} } else { ${Env:computername} })" `
    --url "$(${Env:AZP_URL})" `
    --auth PAT `
    --token "$(Get-Content ${Env:AZP_TOKEN_FILE})" `
    --pool "$(if (Test-Path Env:AZP_POOL) { ${Env:AZP_POOL} } else { 'Default' })" `
    --work "$(if (Test-Path Env:AZP_WORK) { ${Env:AZP_WORK} } else { '_work' })" `
    --replace

  Write-Host "4. Running Azure Pipelines agent..." -ForegroundColor Cyan

  .\run.cmd
}
finally
{
  Write-Host "Cleanup. Removing Azure Pipelines agent..." -ForegroundColor Cyan

  .\config.cmd remove --unattended `
    --auth PAT `
    --token "$(Get-Content ${Env:AZP_TOKEN_FILE})"
}
```

Create a container registry
```
az group create --name buildagent --location uksouth
az acr create --resource-group buildagent --name alberto --sku Basic
```

Build Docker image
```
az acr build --registry alberto -t buildagent:v1.0 --platform windows .
```

Create a Personal Access Token in Azure DevOps

Enable the Admin User in Azure Container Registry
```
az acr update -n alberto --admin-enabled true
```

Look for server credentials
```
az acr show --name alberto --query loginServer
az acr credential show --name alberto
```

Create a container
```
az container create
  --resource-group buildagent
  --name alberto
  --image alberto.azurecr.io/buildagent:v1
  --restart-policy OnFailure
  --registry-login-server yourregistrylogin
  --registry-username yourusername
  --registry-password yourpassword
  --environment-variables
    'AZP_URL'='https://dev.azure.com/albertodenatale0870'
    'AZP_TOKEN'='yourtoken'
    'AZP_POOL'='DockerAgent'
    'AZP_AGENT_NAME'='DockerAgent1'
 ```


