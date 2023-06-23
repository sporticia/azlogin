# Github Actions login using Azure Service Principal

## Create service principal

You will need to create a service principal, something like this (replace the subscription GUID with a real one).

```bash
az ad sp create-for-rbac --name "App" --role contributor --scopes /subscriptions/e586703f-8ba9-5a0d-bd44-xxxxxxxxxxxx --sdk-auth
```

Grab these values from the json output.

- clientId
- clientSecret
- subscriptionId
- tenantId

```json
{
  "clientId": "008c50b3-1ddf-4c11-bfa2-xxxxxxxxxxxx",
  "clientSecret": "fwQ8Q~toHtwgS6ZIy-xxxxxxxxxxxxxxxxxxxxxx",
  "subscriptionId": "e256703f-3fa4-4a5d-bc44-xxxxxxxxxxxx",
  "tenantId": "4b642344-8834-4175-96d1-xxxxxxxxxxxx",
}
```

## Store service principal values in Github secrets store

Store the json output in GitHub Actions secrets and variables `Settings ==> Actions secrets and variables` as a single secret/entry.

Give it a name (in this example, '**AZURE_CREDS**'), then reference the name in the workflow.

```yaml
on:
  # Show manual 'Run workflow button'
  workflow_dispatch:

name: 'Azure Login Service Principal'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: 'Az CLI login'
    - uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDS }}  <<<===

    - name: 'Run Azure CLI commands'
      uses: azure/CLI@v1
      with:
        azcliversion: 2.30.0
        inlineScript: |
          az account show
          az group list
          pwd
          echo 'Service Principal'
```
# Github Actions login using Azure OpenID

## Create Azure AD app registration

You will need to create an app registration in Azure AD.

```bash
az ad app create --display-name App
```

You will need to grab the `appId` from the json output.

```json
"appId": "d364f96f-fd7a-434d-ae4f-xxxxxxxxxxxx"
```

## Create service principal linked to app.

Create service principal linked to the app via it's appId.

```bash
az ad sp create --id {$App.appId}
```

*NOTE:* This will output a different/new `appId` that you will need to record for later.

## Assign RBAC role to service principal.

Assign an RBAC role assignment (using `contributor` here, but you can refine this down to something smaller).

```bash
az role assignment create --role contributor --subscription $subscriptionId --assignee-object-id  $assigneeObjectId --assignee-principal-type ServicePrincipal --scope /subscriptions/$subscriptionId/resourceGroups/$resourceGroupName
```

*NOTE:* `--assignee-object-id` is the `appId` returned by the `az ad sp create` command, **NOT** the `az ad app create` command!

*NOTE:* We can control these rights to a whole subscription or a resource group with the `--scope` argument.

## Add federated credential

*NOTE:*>  `--id` is the `appId` the returned by the `az ad app create` command **NOT** the `az ad sp create` command.

```bash
az ad app federated-credential create --id $GH-Actions-Repo-App.appId --parameters credentials.json
```

Where the `credentials.json` file contains this.

```json
{
  "name":"{name}",
  "issuer":"https://token.actions.githubusercontent.com",
  "subject":"repo:{org}/{repo}:ref:refs/heads/main",
  "description":"{description}",
  "audiences":[
    "api://AzureADTokenExchange"
  ]
}
```

> NOTE: I had to refresh my browser windows completely to get the federated credential to appear in the list on the browser!?

## Store service principal values in Github secrets store

Store the json outputs in GitHub Actions secrets and variables `Settings ==> Actions secrets and variables`.

You will need to save each of the below values:

| GitHub Secret Name | Azure AD Value |
| ------------- | ---------------------------------- |
| AZURE_CLIENT_ID | Application (client) ID |
| AZURE_TENANT_ID | Directory (tenant) ID |
| AZURE_SUBSCRIPTION_ID | Subscription ID |

and then reference them in the workflow.

```yaml
on:
  # Show manual 'Run workflow button'
  workflow_dispatch:

name: 'Azure Login Service Principal'

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    - name: 'Az CLI login'
    - uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: 'Run Azure CLI commands'
      uses: azure/CLI@v1
      with:
        azcliversion: 2.30.0
        inlineScript: |
          az account show
          az group list
          pwd
          echo 'Service Principal'
```
## Create secrets in Github secrets
| GitHub Secret Name | Azure AD Value |
| ------------- | ---------------------------------- |
| AZURE_CLIENT_ID | Application (client) ID |
| AZURE_TENANT_ID | Directory (tenant) ID |
| AZURE_SUBSCRIPTION_ID | Subscription ID |

