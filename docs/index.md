# Introduction

# Restrict permissions on Azure DevOps service connection

# Where to store secrets

Azure DevOps secret variables vs. Key Vault

> Secret Variables
> + Easy to create
> - "Lock in" to Azure DevOps
> 
> Key Vault
> + Independent of pipeline
> + Single point of truth
> - A bit more effort needed

# How to integrate key vault in pipeline

# How to create resources
Recommended to not do without review and approval
- PowerShell
- Terraform
- others

# Managed identity & AAD authentication
- Snippets for .NET Core application
- Snippets for deployment pipelines (get token without secret stored - az login)
	- GitHub actions
	- Azure DevOps YAML pipelines

# How to deploy persistency
- Migrations
- db_user creation
- rolling sql admin login with Terraform

# How to integrate SonarCloud


For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).
