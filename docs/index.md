# Introduction

This cheat sheet summarizes information and best practices concerning DevOps security. As DevOps security is a really huge topic the cheat sheet has no claim to completeness but simply covers a few selected areas of DevOps security.

# General

## Supply Chain Security

Update dependencies regularly to the newest version.

## Resource Creation

Recommended to not do without review and approval

- PowerShell
- Terraform
- others

## Deployment of Persistency

- Migrations
- db_user creation
- rolling sql admin login with Terraform

# Azure DevOps

## Access Control for Azure DevOps Service Connections

- Restrict permissions for Azure DevOps service connection.
- Do not add add write permissions. _Get_ and _List_ are enough.

## Secrets Management

### Where to store secrets: Azure DevOps secret variables vs Key Vault

> Secret Variables
>
> - (-) "Lock in" to Azure DevOps
>
> - (+) Easy to create

> Key Vault
>
> - (-) Independent of pipeline
> - (-) Single point of truth
>
> - (+) Requires Azure account
> - (+) A bit more effort needed
> - (+) It comes with an extra cost, though is minimal [0.026 € per 10000 transactions](https://azure.microsoft.com/en-us/pricing/details/key-vault/).

### KeyVault and non AzureDevOps pipelines

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

## Integration of SonarCloud

# Useful Links

- [Code With Engineering Playbook](https://microsoft.github.io/code-with-engineering-playbook/security/)
- [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/)
