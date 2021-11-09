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
- Download libraries/packages always ans only from trusted sources (i.e. NuGet, Maven, npm, ...)
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

# Azure DevOps

## Access Control for Azure DevOps Service Connections

- Restrict permissions for Azure DevOps service connection.
- Do not add add write permissions. _Get_ and _List_ are enough.

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

## Overall

- Avoid secrets when is possible (try to use managed identity first)
- If it is not possible, then use key vaults created in Azure to store secrets.
- If it is not possible to use managed identities neither Key Vault (ie, on promise client), then use Azure DevOps secret variables. When creating Azure DevOps variable do not forget to check _Keep this value secret_.
- Do not create the service principal in Azure DevOps (or restrict the permissions afterwards).
- A best practice is to create a separate vault for each deployment environment of each of your applications, such as development, test, and production.
- Always carefully review your code to ensure that your app never writes secrets to any kind of output, including logs, storage, and responses. Never expose secrets.
- Be aware and define who has permission to access artifacts. Do developers of the team really need to access artifacts?

## Integration of Key Vault in Pipelines

How to integrate Azure key vault in pipeline

1. Create Azure Key Vault.
1. Create a task in your pipeline using _Azure Key Vault_ template.
1. Select and authorize your Azure subscription and choose the key vault created in step 1. to be added in the pipeline. Your pipeline should look like this:

   ```
   steps:
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

# Azure

## Managed identity & AAD authentication

- Snippets for .NET Core application
- Snippets for deployment pipelines (get token without secret stored - az login)
  - GitHub actions
  - Azure DevOps YAML pipelines

# Pipelines

## Deployment of Persistency

There are multiple possibilities to manage persistence layer in a deployment pipeline. The following example shows a secret-less approach that does the following:

- Apply EF Core migrations to a SQL database by making use of idempotent script approach
- Create users for Azure Active Directory (AAD) roles in the database to support AAD authentication
- Assgin before created database users for AAD roles to built in database roles

**Root YAML file**
```yaml
name: 'PIPELINE_NAME_HERE'
trigger:
  batch: true
  branches:
    include:
      - main
      - dev
  paths:
    include:
      - src
variables:
  - name: 'isMainBranch'
    value: $[eq(variables['Build.SourceBranch'], 'refs/heads/main')]
  - name: 'isDevBranch'
    value: $[eq(variables['Build.SourceBranch'], 'refs/heads/dev')]
  - name: 'isTargetMainBranch'
    value: $[eq(variables['system.pullRequest.targetBranch'], 'refs/heads/main')]
  - name: 'isReleasePR'
    value: $[startsWith(variables['system.pullRequest.sourceBranch'], 'refs/heads/release')]
  - name: 'isHotfixPR'
    value: $[startsWith(variables['system.pullRequest.sourceBranch'], 'refs/heads/hotfix')]
  - name: 'rootPath'
    value: 'src'
  - name: 'artifactName'
    value: 'APP_NAME_HERE'
  - name: 'DOTNET_CLI_TELEMETRY_OPTOUT'
    value: true
pool:
  vmImage: 'windows-latest'
stages:
  - stage: Build
    displayName: 'Build'
    jobs:
      - template: jobs-build-dotnet.yml
        parameters:
          artifactName: $(artifactName)
          rootPath: $(rootPath)
  - stage: DeployDev
    displayName: 'Deploy to DEV'
    dependsOn: Build
    condition: and(succeeded(), eq(variables.isDevBranch, true))
    jobs:
      - template: jobs-deploy.yml
        parameters:
          artifactName: $(artifactName)
          environment: 'Development'
          environmentCode: 'd1'
          serviceConnection: 'SERVICE_CONNECTION_NAME_HERE'
          webAppName: 'WEB_APP_NAME_HERE'
```

**jobs-deploy.yml**
```yaml
parameters:
  - name: 'artifactName'
    type: string
  - name: 'environment'
    type: string
  - name: 'environmentCode'
    type: string
  - name: 'serviceConnection'
    type: string
  - name: 'webAppName'
    type: string
    
jobs:
  - deployment: 'Deploy'
    displayName: 'Deploy Azure Web App'
    environment: ${{parameters.environment}}
    variables:
      databaseName: customer-${{parameters.environmentCode}}-sqldb-dbname
      databaseServerUrl: customer-${{parameters.environmentCode}}-sql-dbservername.database.windows.net
      readerGroupName: sqldb-dbname-${{parameters.environmentCode}}-dbreader
      writerGroupName: sqldb-dbname-${{parameters.environmentCode}}-dbwriter
      ownerGroupName: sqldb-dbname-${{parameters.environmentCode}}-dbowner
    strategy:
      runOnce:
        deploy:
          steps:
            - checkout: self
            
            - template: steps-prepare-database.yml
              parameters:
                databaseName: $(databaseName)
                databaseServerUrl: $(databaseServerUrl)
                readerGroupName: $(readerGroupName)
                writerGroupName: $(writerGroupName)
                ownerGroupName: $(ownerGroupName)
                serviceConnection: ${{parameters.serviceConnection}}

            - task: AzureRmWebAppDeployment@4
              displayName: Deploy Azure Web App
              inputs:
                ConnectionType: 'AzureRM'
                azureSubscription: ${{parameters.serviceConnection}}
                appType: 'webApp'
                WebAppName: ${{parameters.webAppName}}
                packageForLinux: '$(Agent.BuildDirectory)/${{parameters.artifactName}}/*.zip'
```

**steps-prepare-database.yml**
```yaml
parameters:
  databaseName: ""
  databaseServerUrl: ""
  readerGroupName: ""
  writerGroupName: ""
  ownerGroupName: ""
  serviceConnection: ""

steps:
  - task: AzureCLI@2
    displayName: Database ${{parameters.databaseName}} - Set permissions
    inputs:
      azureSubscription: ${{parameters.serviceConnection}}
      scriptType: pscore
      scriptLocation: inlineScript
      inlineScript: |
        $accessToken = az account get-access-token --resource=https://database.windows.net | ConvertFrom-Json | select -ExpandProperty accessToken
        $setGroupPermissionsQuery = @"
            IF NOT EXISTS(SELECT name FROM sys.database_principals WHERE name = '${{parameters.readerGroupName}}')
            BEGIN
            CREATE USER [${{parameters.readerGroupName}}] FROM EXTERNAL PROVIDER
            ALTER ROLE db_datareader ADD MEMBER [${{parameters.readerGroupName}}] 
            END
            IF NOT EXISTS(SELECT name FROM sys.database_principals WHERE name = '${{parameters.writerGroupName}}')
            BEGIN
            CREATE USER [${{parameters.writerGroupName}}] FROM EXTERNAL PROVIDER
            ALTER ROLE db_datawriter ADD MEMBER [${{parameters.writerGroupName}}]
            END
            IF NOT EXISTS(SELECT name FROM sys.database_principals WHERE name = '${{parameters.ownerGroupName}}')
            BEGIN
            CREATE USER [${{parameters.ownerGroupName}}] FROM EXTERNAL PROVIDER
            ALTER ROLE db_owner ADD MEMBER [${{parameters.ownerGroupName}}]
            END
        "@
        Install-Module SqlServer -Scope CurrentUser -Force
        Invoke-Sqlcmd -ServerInstance ${{parameters.databaseServerUrl}} -Database ${{parameters.databaseName}} -AccessToken $accessToken -Query $setGroupPermissionsQuery

  - task: AzureCLI@2
    displayName: Database ${{parameters.databaseName}} - Apply migrations
    inputs:
      azureSubscription: ${{parameters.serviceConnection}}
      scriptType: pscore
      scriptLocation: inlineScript
      inlineScript: |
        dotnet tool install --global dotnet-ef --version 5.0.9
        dotnet ef migrations script --idempotent --project src/Server.Data --startup-project src/Server --output migrations.sql
        
        $accessToken = az account get-access-token --resource=https://database.windows.net | ConvertFrom-Json | select -ExpandProperty accessToken
        
        Install-Module SqlServer -Scope CurrentUser -Force
        Invoke-Sqlcmd -ServerInstance ${{parameters.databaseServerUrl}} -Database ${{parameters.databaseName}} -AccessToken $accessToken -InputFile "migrations.sql"

```

## Integration of SonarCloud

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

## Secrets Detection

Tools for secrets detection offer an additional possibility to enforce secret-free code. Usually secrets detection will be integrated into the build pipeline and let the build fail in case a secret gets detected. `YELP detect-secrets` is such a tool that canbe integrated into build pipelines. A HOWTO for Azure DevOps YAML pipelines can be found [here](https://microsoft.github.io/code-with-engineering-playbook/continuous-integration/dev-sec-ops/secret-management/recipes/detect-secrets-ado/).

In case a secret got detected there are multiple options to resolve the issue after removing the secret from code:

- Merge feature into dev/develop branch using merge type `squash commit`
- Delete feature branch completely and start from scratch

Make sure that in any case the secret gets invalidated by changing the key/password/secret at the corresponding destination system.
Hint: GitHub automatically disables GitHub access tokens that get committed to the GitHub repository they belong to on commit.

# Useful Links

- [Code With Engineering Playbook](https://microsoft.github.io/code-with-engineering-playbook/security/)
- [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/)
