# Introduction

# Managed identity & AAD authentication
- Snippets for .NET Core application
- Snippets for deployment pipelines (get token without secret stored - az login)
	- GitHub actions
	- Azure DevOps YAML pipelines

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

# How to integrate key vault

# How to create resources
- PowerShell
- Terraform
- others

# Restrict permissions on Azure DevOps service connection

# How to deploy persistency
- Migrations
- db_user creation
- rolling sql admin login with Terraform

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).
