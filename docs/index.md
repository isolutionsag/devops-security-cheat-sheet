# Introduction

This cheat sheet summarizes information and best practices concerning DevOps security. As DevOps security is a really huge topic the cheat sheet has no claim to completeness but simply covers a few selected areas of DevOps security.

# General

In context of DevOps security try to always follow the `Need to know` and `least privilege` principles.

## Supply Chain Security

Since the SolarWinds hack, supply chain security has become significantly more important. The following recommendations will help you to increase supply chain security at least a little bit.

- During library/package selection process check out ...
  - ... the transitive dependencies
  - ... the amount of collaborators (the more the better)
  - ... the amount of downloads (the more the better)
  - ... the latest activities (the more actual the better)
- Download libraries/packages only from trusted sources (i.e. NuGet, Maven, npm, ...)
- Update dependencies regularly to the latest version (at least to the latest minor version)

### Dependency Analysis

GitHub provides two interesting and recommendable tools for dependency analysis
- [Dependency Graph](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph)
- [Dependabot Security Updates](https://docs.github.com/en/code-security/supply-chain-security/managing-vulnerabilities-in-your-projects-dependencies/configuring-dependabot-security-updates)

## Resource Creation

Automation of resource creation also known as infrastructure as code (IaC) can increase security too. Besides it allows to reproduce the infrastructure easily, it reduces the risk of manual misconfiguration. Furthermore IaC supports mechanisms to roll secrets on every deployment.
It's a good practice to always review changes to the infrastructure before they get applied.

Possible tools to automate infrastructure creation are:

- [Microsoft PowerShell](https://docs.microsoft.com/en-us/powershell/)
- [HashiCorp Terraform](https://www.terraform.io/)
- [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
- [Chef](https://www.chef.io/products/chef-infra)
- [Puppet](https://puppet.com/)
- [Ansible](https://www.ansible.com/)
- ...

## Source Control Configuration

Recommendations concerning source control configuration:

- Protect branches
  - Restrict access to dev/develop branch to developers
  - Restrict access to master/main branch to release manager and its deputy
  - Allow commits to master/main only through pull requests
- Restrict write access to contributors

# Azure

## Managed identity & AAD authentication
Managed identities provide an identity for applications to use when connecting to resources that support Azure Active Directory (Azure AD) authentication. Using managed identities is the best way to access Azure resources and should be used whenever possible. Should secrets still be required at runtime of the application, they can be stored securely in an Azure Key Vault and read out via Managed Identity.

The following tutorial shows how to connect your ASP.NET Core app to an Azure SQL Database.
- [Tutorial: ASP.NET Core with Azure SQL Database - Azure App Service](https://docs.microsoft.com/en-us/azure/app-service/tutorial-dotnetcore-sqldb-app)

# Azure DevOps

## Overall

- Avoid secrets when is possible (try to use managed identity first)
- If it is not possible, then use key vaults created in Azure to store secrets.
- If it is not possible to use managed identities neither Key Vault (ie, on promise client), then use Azure DevOps secret variables. When creating Azure DevOps variable do not forget to check _Keep this value secret_.
- Do not create the service principal in Azure DevOps (or restrict the permissions afterwards).
- A best practice is to create a separate vault for each deployment environment of each of your applications, such as development, test, and production.
- Always carefully review your code to ensure that your app never writes secrets to any kind of output, including logs, storage, and responses. Never expose secrets.
- Be aware and define who has permission to access artifacts. Do developers of the team really need to access artifacts?

## Secrets Management

### Where to store secrets: Azure DevOps secret variables vs Key Vault

> Secret Variables
>
> - (+) Easy to create
>
> - (-) "Lock in" to Azure DevOps

> Key Vault
>
> - (+) Independent of pipeline
> - (+) Single point of truth
>
> - (-) Requires Azure account
> - (-) A bit more effort needed
> - (-) It comes with an extra cost, though is minimal [0.026 € per 10000 transactions](https://azure.microsoft.com/en-us/pricing/details/key-vault/).

### KeyVault and non Azure DevOps pipelines

- How to integrate it with [GitHub actions](https://docs.microsoft.com/en-us/azure/developer/github/github-key-vault)
- How to integrate it with [Jenkins](https://plugins.jenkins.io/azure-keyvault/)

## Access Control for Azure DevOps Service Connections

- Restrict permissions for Azure DevOps service connection.
  - [Service connections in Azure Pipelines](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml#secure-a-service-connection)
- Do not add add write permissions. _Get_ and _List_ are enough.

## Common pipeline tasks

### Integration of Key Vault in Pipelines

How to integrate Azure key vault in pipeline

1. Create Azure Key Vault.
1. Create a task in your pipeline using _Azure Key Vault_ template.
1. Select and authorize your Azure subscription and choose the key vault created in step 1. to be added in the pipeline. Your pipeline should look like this:

   ```yaml
   - task: AzureKeyVault@2
     inputs:
         azureSubscription: 'Your-Azure-Subscription'
         KeyVaultName: 'Your-Key-Vault-Name'
         SecretsFilter: '*'
         RunAsPreJob: false
   ```

1. Create your [service principal](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal#register-an-application-with-azure-ad-and-create-a-service-principal)
1. Go to your KeyVault, under settings select _Access Policies_, and then _Add Access Policy_. Select your service principal created in previous step. As it is a secret _only assign Get and List permissions_ (important!), add and save.
1. Save and run the pipeline. You should have a secret.txt file as a result.

[Official documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/release/azure-key-vault?view=azure-devops)


### Assign Azure SQL Permissions

The following example show how to assign permissions on an Azure SQL Database to Azure Active Directory Groups in an Azure DevOps Pipeline.

#### Prerequisites
- An Azure SQL Database.
- [Prerequisites of Azure CLI task](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/azure-cli#prerequisites)

#### Task to assign permissions
```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: AZURE_SUBSCRIPTION
    scriptType: pscore
    scriptLocation: inlineScript
    inlineScript: |
      # Install PS Modules
      Install-Module SqlServer -Scope CurrentUser -Force
              
      # Get Access Token
      $accessToken = az account get-access-token --resource=https://database.windows.net | ConvertFrom-Json | select -ExpandProperty accessToken

      # Create SQL Command
      $setGroupPermissionsCommand = @"
          IF NOT EXISTS(SELECT name FROM sys.database_principals WHERE name = 'WRITER_GROUP_NAME')
          BEGIN
          CREATE USER [WRITER_GROUP_NAME] FROM EXTERNAL PROVIDER
          ALTER ROLE db_datawriter ADD MEMBER [WRITER_GROUP_NAME]
          END
      "@

      # Execute SQL Command     
      Invoke-Sqlcmd -ServerInstance DB_SERVER_URL -Database DB_NAME -AccessToken $accessToken -Query $setGroupPermissionsCommand
```

To be replaced:
- AZURE_SUBSCRIPTION
- DB_SERVER_URL
- DB_NAME
- WRITER_GROUP_NAME

#### Related tasks
The same mechanism can be used to perform other database actions, such as applying EF Core migrations.

```powershell
dotnet tool install --global dotnet-ef --version 5.0.9
dotnet ef migrations script --idempotent --project PATH/TO/PROJECT/FOLDER --startup-project PATH/TO/STARTUP/PROJECT/FOLDER --output migrations.sql

$accessToken = az account get-access-token --resource=https://database.windows.net | ConvertFrom-Json | select -ExpandProperty accessToken

# Execute SQL Command
Invoke-Sqlcmd -ServerInstance DB_SERVER_URL -Database DB_NAME -AccessToken $accessToken -InputFile "migrations.sql"
```

### Integration of SonarCloud

To execute SonarCloud analysis for example on each pull request (including each commit on that pull request) you first have to create a new project at [SonarCloud](https://sonarcloud.io/). 

Such a pipeline for a .NET Core Web API with a ReactJS frontend could then look as follows.

```yaml
name: 'PROJECT_NAME SonarCloud'
trigger:
  - develop

variables:
  - name: 'BuildConfiguration'
    value: 'Release'
  - name: 'isTargetDevelopBranch'
    value: $[eq(variables['system.pullRequest.targetBranch'], 'refs/heads/develop')]

pool:
  vmImage: 'windows-latest'
jobs:
  - job: build
    displayName: build
    condition: eq(variables.isTargetDevelopBranch, true)
    steps:
      - task: SonarCloudPrepare@1
        displayName: 'Prepare analysis on SonarCloud'
        inputs:
          SonarCloud: 'SONAR_CLOUD_ENDPOINT_NAME'
          organization: SONAR_CLOUD_ORGANIZATION_NAME
          projectKey: 'SONAR_CLOUD_PROJECT_KEY'
          extraProperties: |
            sonar.exclusions=**/obj/**,**/*.dll
            sonar.cs.opencover.reportsPaths=$(Build.SourcesDirectory)/**/coverage.opencover.xml
            sonar.cs.vstest.reportsPaths=$(Agent.TempDirectory)/*.trx
            sonar.cs.file.suffixes=.cs
            sonar.cs.ignoreHeaderComments=True
            sonar.sources=src/frontend
      - task: DotNetCoreCLI@2
        displayName: Build
        inputs:
          projects: '**/*.sln'
          arguments: '--configuration $(BuildConfiguration)'
      - task: DotNetCoreCLI@2
        displayName: Test
        inputs:
          command: test
          projects: '**/*Tests/*.csproj'
          arguments: '--configuration $(BuildConfiguration) --collect "Code coverage"'
      - task: 'SonarCloudAnalyze@1'
        displayName: 'Run Code Analysis'
      - task: 'SonarCloudPublish@1'
        displayName: 'Publish Quality Gate Result'
      - task: DotNetCoreCLI@2
        displayName: Restore
        inputs:
          command: restore
          projects: '**/*.sln'
```

To be replaced:
- `PROJECT_NAME`
- `SONAR_CLOUD_ENDPOINT_NAME`
- `SONAR_CLOUD_ORGANIZATION_NAME`
- `SONAR_CLOUD_PROJECT_KEY`

### Secrets Detection

Tools for secrets detection offer an additional possibility to enforce secret-free code. Usually secrets detection will be integrated into the build pipeline and let the build fail in case a secret gets detected. `YELP detect-secrets` is such a tool that canbe integrated into build pipelines. A HOWTO for Azure DevOps YAML pipelines can be found [here](https://microsoft.github.io/code-with-engineering-playbook/continuous-integration/dev-sec-ops/secret-management/recipes/detect-secrets-ado/).

In case a secret got detected there are multiple options to resolve the issue after removing the secret from code:

- Merge feature into dev/develop branch using merge type `squash commit`
- Delete feature branch completely and start from scratch

Make sure that in any case the secret gets invalidated by changing the key/password/secret at the corresponding destination system.
Hint: GitHub automatically disables GitHub access tokens that get committed to the GitHub repository they belong to on commit.

# Useful Links

- [Code With Engineering Playbook](https://microsoft.github.io/code-with-engineering-playbook/security/)
- [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/)
