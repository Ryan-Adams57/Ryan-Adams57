# Azure Multi-Stage Pipeline Setup Guide

This guide will help you set up and configure the Multi-Stage, Environment-Based CI/CD Pipeline in Azure DevOps.

## Prerequisites

- Azure DevOps organization and project
- Azure subscription for deploying resources
- Repository imported into Azure DevOps Repos or connected via GitHub

## Step 1: Create Azure Environments

In Azure DevOps, you need to create three environments for the pipeline stages:

### 1.1 Navigate to Environments
1. Go to your Azure DevOps project
2. Click on **Pipelines** > **Environments**
3. Click **New environment**

### 1.2 Create Dev Environment
- **Name**: `dev`
- **Description**: Development environment for rapid iteration
- **Resource**: None (or add Azure resources if needed)
- **Approvals**: Not required

### 1.3 Create Staging Environment
- **Name**: `staging`
- **Description**: Pre-production environment for validation
- **Resource**: None (or add Azure resources if needed)
- **Approvals**: Optional - add automated checks if desired

### 1.4 Create Production Environment
- **Name**: `prod`
- **Description**: Production environment
- **Resource**: None (or add Azure resources if needed)
- **Approvals**: **Required** - Configure as follows:
  1. Click on the `prod` environment
  2. Click the three dots (⋮) > **Approvals and checks**
  3. Click **+ Add** > **Approvals**
  4. Add approvers (users or groups who must approve prod deployments)
  5. Set timeout and other options as needed
  6. Click **Create**

## Step 2: Configure Pipeline Variables

### 2.1 Update Variables in azure-pipelines-1.yml

Edit the `azure-pipelines-1.yml` file and update these variables to match your Azure resources:

```yaml
variables:
  # Azure Subscription - Replace with your service connection name
  azureSubscription: 'MyAzureSubscription'
  
  # Resource Groups - Replace with your actual resource group names
  devResourceGroup: 'rg-dev'
  stagingResourceGroup: 'rg-staging'
  prodResourceGroup: 'rg-prod'
  
  # App Service Names - Replace with your actual App Service names
  devAppService: 'app-dev-webapp'
  stagingAppService: 'app-staging-webapp'
  prodAppService: 'app-prod-webapp'
  
  # Container Registry - Replace if using containers
  containerRegistry: 'myregistry.azurecr.io'
  imageName: 'my-webapp'
```

### 2.2 Create Azure Service Connection

1. Go to **Project Settings** > **Service connections**
2. Click **New service connection**
3. Select **Azure Resource Manager**
4. Choose authentication method (Service Principal recommended)
5. Select your Azure subscription
6. Name it to match the `azureSubscription` variable (e.g., 'MyAzureSubscription')
7. Click **Save**

### 2.3 (Optional) Store Sensitive Variables in Library

For sensitive data like API keys or connection strings:

1. Go to **Pipelines** > **Library**
2. Click **+ Variable group**
3. Name the group (e.g., "Pipeline-Secrets")
4. Add variables and mark them as secret
5. Link the variable group to your pipeline

## Step 3: Create Azure Resources

You'll need to create the following Azure resources for each environment:

### For Each Environment (Dev, Staging, Prod):

1. **Resource Group**
   ```bash
   az group create --name rg-dev --location eastus
   az group create --name rg-staging --location eastus
   az group create --name rg-prod --location eastus
   ```

2. **App Service Plan**
   ```bash
   az appservice plan create --name plan-dev --resource-group rg-dev --sku B1
   az appservice plan create --name plan-staging --resource-group rg-staging --sku B2
   az appservice plan create --name plan-prod --resource-group rg-prod --sku S1
   ```

3. **App Service (Web App)**
   ```bash
   # For Node.js
   az webapp create --name app-dev-webapp --resource-group rg-dev --plan plan-dev --runtime "NODE|18-lts"
   
   # For .NET
   az webapp create --name app-dev-webapp --resource-group rg-dev --plan plan-dev --runtime "DOTNET|7.0"
   
   # For Python
   az webapp create --name app-dev-webapp --resource-group rg-dev --plan plan-dev --runtime "PYTHON|3.11"
   ```

4. **(Optional) Container Registry**
   ```bash
   az acr create --name myregistry --resource-group rg-prod --sku Basic
   ```

## Step 4: Create or Import the Pipeline

### 4.1 Create New Pipeline
1. Go to **Pipelines** > **Pipelines**
2. Click **New pipeline**
3. Select your repository source (Azure Repos Git or GitHub)
4. Select your repository
5. Choose **Existing Azure Pipelines YAML file**
6. Select `/azure-pipelines-1.yml`
7. Click **Continue**
8. Click **Run** to execute the pipeline

### 4.2 First Run Permissions
On the first run, you may need to grant permissions:
- Authorize access to environments
- Authorize service connections
- Approve resource access

## Step 5: Test the Pipeline

### 5.1 Trigger the Pipeline
- **Automatic**: Push to the `main` branch
- **Manual**: Click **Run pipeline** in Azure DevOps

### 5.2 Monitor Execution
1. Watch the pipeline execute through stages:
   - ✅ Build & Test (automatic)
   - ✅ Deploy to Dev (automatic)
   - ✅ Deploy to Staging (automatic)
   - ⏸️ Deploy to Production (requires manual approval)

2. When the production stage is ready:
   - You'll receive a notification (if configured)
   - Navigate to the pipeline run
   - Click **Review** on the approval
   - Add comments if needed
   - Click **Approve** or **Reject**

## Step 6: Customization Options

### 6.1 Add Automated Checks to Environments
- **Azure Functions**: Run custom validation logic
- **Azure Monitor**: Check for alerts
- **REST API**: Call external validation services
- **Query Azure Boards**: Check for linked work items

### 6.2 Customize Build Steps
The pipeline detects and builds multiple platforms automatically:
- Modify build commands for your specific project
- Add or remove platform-specific steps
- Customize test commands

### 6.3 Add Deployment Slots (Blue-Green Deployment)
```bash
# Create deployment slot
az webapp deployment slot create --name app-prod-webapp --resource-group rg-prod --slot staging

# Deploy to slot first, then swap
az webapp deployment slot swap --name app-prod-webapp --resource-group rg-prod --slot staging
```

## Pipeline Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         TRIGGER (Push/PR)                       │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     STAGE 1: BUILD & TEST                       │
├─────────────────────────────────────────────────────────────────┤
│  Job 1: Build Application (Node.js/.NET/Python/Docker)          │
│  Job 2: Run Tests (Unit & Integration)                          │
│  Job 3: Security Scan (Vulnerabilities & Quality)               │
└────────────────────────────────┬────────────────────────────────┘
                                 │ (Automatic)
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   STAGE 2: DEPLOY TO DEV                        │
├─────────────────────────────────────────────────────────────────┤
│  Environment: dev                                                │
│  - Deploy Infrastructure                                         │
│  - Deploy Application                                            │
│  - Run Smoke Tests                                               │
└────────────────────────────────┬────────────────────────────────┘
                                 │ (Automatic)
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                 STAGE 3: DEPLOY TO STAGING                      │
├─────────────────────────────────────────────────────────────────┤
│  Environment: staging                                            │
│  Job 1: Deploy to Staging                                        │
│    - Deploy Infrastructure                                       │
│    - Deploy Application                                          │
│    - Quality Gates (Performance/Load/Security)                   │
│  Job 2: Staging Validation                                       │
│    - End-to-end Tests                                            │
│    - API Contract Tests                                          │
│    - Performance Baseline                                        │
└────────────────────────────────┬────────────────────────────────┘
                                 │ (Automatic)
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                STAGE 4: DEPLOY TO PRODUCTION                    │
│                   ⚠️  MANUAL APPROVAL REQUIRED                  │
├─────────────────────────────────────────────────────────────────┤
│  Environment: prod                                               │
│  - Deploy Infrastructure                                         │
│  - Deploy Application (Blue-Green)                               │
│  - Production Smoke Tests                                        │
│  - Deployment Summary                                            │
└─────────────────────────────────────────────────────────────────┘
```

## Troubleshooting

### Pipeline Fails to Start
- Verify service connection is authorized
- Check if environments exist
- Ensure you have permissions on the repository

### Deployment Fails
- Verify Azure resource names match variables
- Check service connection has correct permissions
- Review Azure CLI commands in pipeline logs

### Approval Not Working
- Ensure approvers are added to prod environment
- Check if approvers have correct permissions
- Verify notification settings in Azure DevOps

### Build Fails for Specific Platform
- Check if the build conditions are met (file existence checks)
- Verify tool versions in variables
- Review build logs for specific errors

## Additional Resources

- [Azure Pipelines Documentation](https://docs.microsoft.com/azure/devops/pipelines/)
- [Environments and Approvals](https://docs.microsoft.com/azure/devops/pipelines/process/environments)
- [Azure App Service Deployment](https://docs.microsoft.com/azure/app-service/)
- [YAML Schema Reference](https://docs.microsoft.com/azure/devops/pipelines/yaml-schema)

## Support

For issues or questions:
1. Review the pipeline logs in Azure DevOps
2. Check the [Azure DevOps Documentation](https://docs.microsoft.com/azure/devops/)
3. Open an issue in this repository
4. Contact the repository maintainer

---

**Happy Deploying! 🚀**
