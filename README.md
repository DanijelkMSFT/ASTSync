# Attack Simulation Training Sync Function App

The purpose of this function app is to synchronise Attack Simulation Training to Azure Table storage.

## What is stored?

The following API methods are pulled, flattened, and stored in to Azure Table Storage:
* graph.microsoft.com/beta/security/attackSimulation/simulations -> Simulations Table
* graph.microsoft.com/beta/security/attackSimulation/simulations/{id}/reports/simulationUsers -> SimulationsUsers (stores a row for every user in the simulation) and SimulationUserEvents Table (stores all events, such as click/report, etc.)
* graph.microsoft.com/beta/security/attackSimulation/payloads -> Payloads
* graph.microsoft.com/beta/security/attackSimulation/training -> Trainings
* graph.microsoft.com/beta/users -> Users (Only performed for users that have had a simulation ran against them)

## What happens when we sync?

The function app by default runs every hour. If you want to change this, you will need to fork the code and use your fork for deployment. Be wary of hitting any Graph limitations.

On the initial sync, we will pull down every simulation, simulation user, and event. This will take some time to complete. Be patient.

On the iterative sync windows, we will only pull simulations that are in a running state, or completed within the past 7 days.

User details are pulled as well in to the Users table to enable better reporting, but this can be disabled by setting the "SyncEntra" environment variable to false.

For simplicity of deployment, the Function App storage account is used for the Table storage.

## Installation

Whilst the code here could be adjusted to suit any means, it is intended to run in an Azure Function App.

### Create the function app

When creating the Azure Function App in Azure, use the following options
* **Function App Name**: a descriptive name for your function app, and it must be unique across the azure service. As this is not HTTP triggered, the name shouldn't matter.
* **Runtime stack**: .NET
* **Version**: 8 (LTS), isolated worker model
* **Region**: Any region of your choice

![image](https://github.com/cammurray/ASTSync/assets/26195772/cec9948f-b56b-49ca-aaa4-bbddebd9ccaa)

Do not specify any other options at this stage, and press Review + Create, followed by create.

Wait for deployment to complete, and then proceed to setting the deployment method below.

#### Set the deployment method

In the "Deployment Center" for the Azure Function App (Accessible under the Deployment Menu), populate the deployment options:
* **Source** Manual Deployment - External Git
* **Repository** Is this one here: https://github.com/cammurray/ASTSync
* **Branch** master
* **Repository Type** Public

Click Save when populated

<img width="1050" alt="image" src="https://github.com/cammurray/ASTSync/assets/26195772/a23735d2-9070-45bc-9134-9724ddbaf089">


#### Set up the Authentication to Graph API

There are two ways to perform authentication:
* a secure method which uses a managed identity provisioned by the Function App, however requires some PowerShell
* or by using an Azure AD Application where you manage the Client ID & Secret yourself. This is needed if the Function App is in an Azure Subscription that is not associated to the same Entra directory as M365.

##### (Option 1) Recommended - Managed Identity

First, turn on the managed identity in your azure function app by going to Settings -> Identity and turning on the system assigned managed identity for this function app

<img width="1151" alt="image" src="https://github.com/cammurray/ASTSync/assets/26195772/a1a1ae0a-6721-4244-a6b4-53d7910d4397">

When this option saves, you should be given a Object (principal) ID, you must save this

![image](https://github.com/cammurray/ASTSync/assets/26195772/fa130b73-db26-4cbf-b97c-8205ccf9cde7)



For the next steps, you will need PowerShell. Due to AzureAD module compatability, this cannot be ran from a Mac or an ARM processor. Needs to be run on Windows/x86.

The following script will find the Function App Security Principal, and grant it AttackSimulation.Read.All and User.Read.All scopes to Microsoft Graph.

Copy and paste the following in to a PowerShell prompt:

```
$PrincipalID=Read-Host "Enter the Object ID of the Function App Managed Service Principal"

# Install AAD Module and Connect
Install-Module AzureAD -Scope CurrentUser
Connect-AzureAD

# Find the Managed Service Identity and Graph Service Principal
$MSI = (Get-AzureADServicePrincipal -Filter "ObjectId eq '$PrincipalID'")
$GraphServicePrincipal = Get-AzureADServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"

# Add AttackSimulation.Read.All permission
$AppRole = $GraphServicePrincipal.AppRoles | Where-Object {$_.Value -eq "AttackSimulation.Read.All" -and $_.AllowedMemberTypes -contains "Application"}
New-AzureAdServiceAppRoleAssignment -ObjectId $MSI.ObjectId -PrincipalId $MSI.ObjectId -ResourceId $GraphServicePrincipal.ObjectId -Id $AppRole.Id

# Add User.Read.All permission
$AppRole = $GraphServicePrincipal.AppRoles | Where-Object {$_.Value -eq "User.Read.All" -and $_.AllowedMemberTypes -contains "Application"}
New-AzureAdServiceAppRoleAssignment -ObjectId $MSI.ObjectId -PrincipalId $MSI.ObjectId -ResourceId $GraphServicePrincipal.ObjectId -Id $AppRole.Id
```

Finally function app needs access to the storage table to create tables and add the data
Open the storage account resource
Open Access control (IAM)
Choose "Storage Table Data Contributor" as Role
As Member select your managed identity of the function app
Review + create
Done

#### (Option 2) Not Recommended - Azure AD Application (COMING SOON).

#### Deploy the code

Once the deployment method has been set, you should be able to click the "Sync" button. This downloads the latest version and compiles it. Confirm the redeployment.

![image](https://github.com/cammurray/ASTSync/assets/26195772/70516189-0e0c-4b5c-88a1-97ce1665f72f)

After clicking sync, monitor the deployment on the logs page, the "Status" should change when complete. The deployment will take several minutes.

### Connecting to the Azure Table

For simplicity of deployment, the Azure Function app inbuilt storage account is used for deployment. This storage account will be located automatically in the same resource group as the function app.

![image](https://github.com/cammurray/ASTSync/assets/26195772/f623a7c9-dcf3-475b-a61d-210a4223bcd8)




