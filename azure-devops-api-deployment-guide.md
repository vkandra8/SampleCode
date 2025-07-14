# Azure DevOps YAML Guide: Exposing APIs through Azure

## Overview

This guide provides comprehensive YAML configurations for deploying and exposing APIs through Azure DevOps pipelines to various Azure services. The examples cover different deployment scenarios, from simple web apps to containerized solutions.

## Table of Contents

1. [Basic API Deployment to Azure App Service](#basic-api-deployment-to-azure-app-service)
2. [Container-based API Deployment](#container-based-api-deployment)
3. [Multi-Stage Deployment with Slots](#multi-stage-deployment-with-slots)
4. [API Management Integration](#api-management-integration)
5. [Environment Variables and Configuration](#environment-variables-and-configuration)
6. [Security and Authentication](#security-and-authentication)
7. [Best Practices](#best-practices)

---

## Basic API Deployment to Azure App Service

### .NET Core API Deployment

```yaml
trigger:
- main
- develop

variables:
  # Build Variables
  buildConfiguration: 'Release'
  
  # Azure Variables
  azureSubscription: 'your-service-connection'
  webAppName: 'your-api-app-name'
  resourceGroupName: 'your-resource-group'
  
  # Agent
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build stage'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Restore packages'
      inputs:
        command: 'restore'
        projects: '**/*.csproj'
    
    - task: DotNetCoreCLI@2
      displayName: 'Build project'
      inputs:
        command: 'build'
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration)'
    
    - task: DotNetCoreCLI@2
      displayName: 'Run tests'
      inputs:
        command: 'test'
        projects: '**/*Tests/*.csproj'
        arguments: '--configuration $(buildConfiguration) --collect "Code coverage"'
    
    - task: DotNetCoreCLI@2
      displayName: 'Publish application'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    
    - upload: $(Build.ArtifactStagingDirectory)
      artifact: drop

- stage: Deploy
  displayName: 'Deploy stage'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy API'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy to Azure Web App'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              runtimeStack: 'DOTNETCORE|8.0'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              appSettings: |
                -ASPNETCORE_ENVIRONMENT "Production"
                -ASPNETCORE_URLS "http://+:8080"
```

### Node.js API Deployment

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  webAppName: 'your-nodejs-api'
  resourceGroupName: 'your-resource-group'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build Node.js API'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: NodeTool@0
      displayName: 'Use Node.js 18.x'
      inputs:
        versionSpec: '18.x'
    
    - script: |
        npm ci
        npm run build --if-present
        npm run test --if-present
      displayName: 'Install dependencies, build, and test'
    
    - task: ArchiveFiles@2
      displayName: 'Archive files'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
        includeRootFolder: false
        archiveType: zip
        archiveFile: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
        replaceExistingArchive: true
    
    - upload: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
      artifact: drop

- stage: Deploy
  displayName: 'Deploy Node.js API'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy to Azure Web App'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy Node.js API'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              runtimeStack: 'NODE|18-lts'
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'
              appSettings: |
                -NODE_ENV "production"
                -PORT "8080"
                -WEBSITE_NODE_DEFAULT_VERSION "18.17.0"
```

### Python FastAPI Deployment

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  webAppName: 'your-python-api'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build Python API'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: UsePythonVersion@0
      displayName: 'Use Python 3.11'
      inputs:
        versionSpec: '3.11'
        addToPath: true
    
    - script: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
      displayName: 'Install dependencies'
    
    - script: |
        pip install pytest pytest-cov
        pytest tests/ --junitxml=junit/test-results.xml --cov=. --cov-report=xml
      displayName: 'Run tests'
    
    - task: PublishTestResults@2
      displayName: 'Publish test results'
      condition: succeededOrFailed()
      inputs:
        testResultsFiles: '**/test-*.xml'
        testRunTitle: 'Publish test results for Python'
    
    - task: ArchiveFiles@2
      displayName: 'Archive application'
      inputs:
        rootFolderOrFile: '$(System.DefaultWorkingDirectory)'
        includeRootFolder: false
        archiveType: zip
        archiveFile: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
        replaceExistingArchive: true
    
    - upload: $(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip
      artifact: drop

- stage: Deploy
  displayName: 'Deploy Python API'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy to Azure Web App'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy Python API'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              runtimeStack: 'PYTHON|3.11'
              package: '$(Pipeline.Workspace)/drop/$(Build.BuildId).zip'
              startUpCommand: 'gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app --bind 0.0.0.0:8000'
              appSettings: |
                -PYTHONPATH "/home/site/wwwroot"
                -PORT "8000"
```

---

## Container-based API Deployment

### Docker Container to Azure Container Registry + App Service

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  containerRegistry: 'your-acr.azurecr.io'
  dockerRegistryServiceConnection: 'your-acr-connection'
  imageRepository: 'your-api'
  webAppName: 'your-containerized-api'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build and Push Container'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: Docker@2
      displayName: 'Build and push container image'
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
          latest

- stage: Deploy
  displayName: 'Deploy Container'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy to Container App Service'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebAppContainer@1
            displayName: 'Deploy container to Azure Web App'
            inputs:
              azureSubscription: $(azureSubscription)
              appName: $(webAppName)
              containers: $(containerRegistry)/$(imageRepository):$(tag)
              appSettings: |
                -DOCKER_REGISTRY_SERVER_URL "https://$(containerRegistry)"
                -DOCKER_REGISTRY_SERVER_USERNAME "$(acrUsername)"
                -DOCKER_REGISTRY_SERVER_PASSWORD "$(acrPassword)"
                -WEBSITES_ENABLE_APP_SERVICE_STORAGE "false"
                -WEBSITES_PORT "8080"
```

### Azure Container Apps Deployment

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  containerRegistry: 'your-acr.azurecr.io'
  imageRepository: 'your-api'
  containerAppName: 'your-container-app'
  resourceGroupName: 'your-resource-group'
  containerAppEnvironment: 'your-container-env'
  tag: '$(Build.BuildId)'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build Container'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: Docker@2
      displayName: 'Build and push image'
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: '$(Build.SourcesDirectory)/Dockerfile'
        containerRegistry: 'your-acr-connection'
        tags: |
          $(tag)

- stage: Deploy
  displayName: 'Deploy to Container Apps'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy Container App'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            displayName: 'Deploy to Azure Container Apps'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az containerapp update \
                  --name $(containerAppName) \
                  --resource-group $(resourceGroupName) \
                  --image $(containerRegistry)/$(imageRepository):$(tag) \
                  --set-env-vars \
                    API_VERSION=v1 \
                    ENVIRONMENT=production
```

---

## Multi-Stage Deployment with Slots

### Blue-Green Deployment with Staging Slots

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  webAppName: 'your-api-app'
  resourceGroupName: 'your-resource-group'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build Application'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Build and publish'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    
    - upload: $(Build.ArtifactStagingDirectory)
      artifact: drop

- stage: DeployToStaging
  displayName: 'Deploy to Staging Slot'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: DeployStaging
    displayName: 'Deploy to Staging'
    environment: 'staging'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy to staging slot'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              deployToSlotOrASE: true
              resourceGroupName: $(resourceGroupName)
              slotName: 'staging'
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              appSettings: |
                -ASPNETCORE_ENVIRONMENT "Staging"
                -ConnectionStrings__DefaultConnection "$(stagingConnectionString)"

- stage: RunSmokeTests
  displayName: 'Run Smoke Tests'
  dependsOn: DeployToStaging
  condition: succeeded()
  
  jobs:
  - job: SmokeTests
    displayName: 'Smoke Tests'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Run API smoke tests'
      inputs:
        command: 'test'
        projects: '**/SmokeTests.csproj'
        arguments: '--configuration Release'
      env:
        API_BASE_URL: 'https://$(webAppName)-staging.azurewebsites.net'

- stage: SwapSlots
  displayName: 'Swap to Production'
  dependsOn: RunSmokeTests
  condition: succeeded()
  
  jobs:
  - deployment: SwapSlots
    displayName: 'Swap Staging to Production'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureAppServiceManage@0
            displayName: 'Swap slots'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              WebAppName: $(webAppName)
              ResourceGroupName: $(resourceGroupName)
              SourceSlot: 'staging'
              SwapWithProduction: true

          - task: AzureCLI@2
            displayName: 'Warm up production'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Warm up the production slot
                curl -f https://$(webAppName).azurewebsites.net/health || exit 1
                echo "Production slot warmed up successfully"
```

---

## API Management Integration

### Deploy API with APIM Integration

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  webAppName: 'your-backend-api'
  apimServiceName: 'your-apim-instance'
  apimResourceGroup: 'your-apim-rg'
  apiName: 'your-api'
  apiPath: 'api/v1'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build API'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Build and publish API'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    
    - upload: $(Build.ArtifactStagingDirectory)
      artifact: drop

- stage: DeployAPI
  displayName: 'Deploy Backend API'
  dependsOn: Build
  condition: succeeded()
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy API'
    environment: 'production'
    pool:
      vmImage: $(vmImageName)
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy backend API'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              package: '$(Pipeline.Workspace)/drop/**/*.zip'

- stage: ConfigureAPIM
  displayName: 'Configure API Management'
  dependsOn: DeployAPI
  condition: succeeded()
  
  jobs:
  - job: ConfigureAPIM
    displayName: 'Configure APIM'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: AzureCLI@2
      displayName: 'Configure API in APIM'
      inputs:
        azureSubscription: $(azureSubscription)
        scriptType: 'bash'
        scriptLocation: 'inlineScript'
        inlineScript: |
          # Create or update API
          az apim api create \
            --resource-group $(apimResourceGroup) \
            --service-name $(apimServiceName) \
            --api-id $(apiName) \
            --path $(apiPath) \
            --display-name "Your API" \
            --service-url "https://$(webAppName).azurewebsites.net" \
            --protocols https \
            --subscription-required true
          
          # Import OpenAPI specification
          az apim api import \
            --resource-group $(apimResourceGroup) \
            --service-name $(apimServiceName) \
            --api-id $(apiName) \
            --path $(apiPath) \
            --specification-url "https://$(webAppName).azurewebsites.net/swagger/v1/swagger.json" \
            --specification-format OpenApi
          
          # Configure policies
          az apim api policy create \
            --resource-group $(apimResourceGroup) \
            --service-name $(apimServiceName) \
            --api-id $(apiName) \
            --xml-content '<policies>
              <inbound>
                <rate-limit-by-key calls="100" renewal-period="60" increment-condition="@(context.Response.StatusCode == 200)" counter-key="@(context.Request.IpAddress)" />
                <cors allow-credentials="false">
                  <allowed-origins>
                    <origin>*</origin>
                  </allowed-origins>
                  <allowed-methods>
                    <method>GET</method>
                    <method>POST</method>
                    <method>PUT</method>
                    <method>DELETE</method>
                  </allowed-methods>
                  <allowed-headers>
                    <header>*</header>
                  </allowed-headers>
                </cors>
              </inbound>
              <backend>
                <forward-request />
              </backend>
              <outbound />
              <on-error />
            </policies>'
```

---

## Environment Variables and Configuration

### Secure Configuration Management

```yaml
trigger:
- main

variables:
- group: 'production-variables'  # Variable group containing secrets
- name: azureSubscription
  value: 'your-service-connection'
- name: webAppName
  value: 'your-api-app'

stages:
- stage: Deploy
  displayName: 'Deploy with Secure Configuration'
  jobs:
  - deployment: Deploy
    displayName: 'Deploy API'
    environment: 'production'
    pool:
      vmImage: 'ubuntu-latest'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureKeyVault@2
            displayName: 'Get secrets from Key Vault'
            inputs:
              azureSubscription: $(azureSubscription)
              KeyVaultName: 'your-key-vault'
              SecretsFilter: 'DatabaseConnectionString,JwtSecret,ExternalApiKey'
              RunAsPreJob: false

          - task: AzureWebApp@1
            displayName: 'Deploy with configuration'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              appSettings: |
                -ASPNETCORE_ENVIRONMENT "Production"
                -ConnectionStrings__DefaultConnection "$(DatabaseConnectionString)"
                -JwtSettings__Secret "$(JwtSecret)"
                -ExternalApi__ApiKey "$(ExternalApiKey)"
                -Logging__LogLevel__Default "Information"
                -HealthChecks__UI__HealthCheckDatabaseConnectionString "$(DatabaseConnectionString)"
                -ApplicationInsights__InstrumentationKey "$(applicationInsightsKey)"

          - task: AzureCLI@2
            displayName: 'Configure custom domain and SSL'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Bind custom domain
                az webapp config hostname add \
                  --webapp-name $(webAppName) \
                  --resource-group $(resourceGroupName) \
                  --hostname api.yourdomain.com
                
                # Configure SSL certificate
                az webapp config ssl bind \
                  --name $(webAppName) \
                  --resource-group $(resourceGroupName) \
                  --certificate-thumbprint $(sslCertThumbprint) \
                  --ssl-type SNI
```

---

## Security and Authentication

### API with Azure AD Integration

```yaml
trigger:
- main

variables:
  azureSubscription: 'your-service-connection'
  webAppName: 'your-secure-api'
  resourceGroupName: 'your-resource-group'
  tenantId: '$(azureTenantId)'
  clientId: '$(azureClientId)'

stages:
- stage: Deploy
  displayName: 'Deploy Secure API'
  jobs:
  - deployment: Deploy
    displayName: 'Deploy with Azure AD'
    environment: 'production'
    pool:
      vmImage: 'ubuntu-latest'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            displayName: 'Deploy API'
            inputs:
              azureSubscription: $(azureSubscription)
              appType: 'webAppLinux'
              appName: $(webAppName)
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              appSettings: |
                -ASPNETCORE_ENVIRONMENT "Production"
                -AzureAd__TenantId "$(tenantId)"
                -AzureAd__ClientId "$(clientId)"
                -AzureAd__Audience "api://$(clientId)"

          - task: AzureCLI@2
            displayName: 'Configure Authentication'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Enable Azure AD authentication
                az webapp auth config-version upgrade \
                  --name $(webAppName) \
                  --resource-group $(resourceGroupName)
                
                az webapp auth microsoft update \
                  --name $(webAppName) \
                  --resource-group $(resourceGroupName) \
                  --client-id $(clientId) \
                  --client-secret-setting-name "MICROSOFT_PROVIDER_AUTHENTICATION_SECRET" \
                  --tenant-id $(tenantId) \
                  --allowed-audiences "api://$(clientId)"
                
                # Configure CORS for API
                az webapp cors add \
                  --name $(webAppName) \
                  --resource-group $(resourceGroupName) \
                  --allowed-origins "https://yourdomain.com" "https://app.yourdomain.com"

          - task: AzureCLI@2
            displayName: 'Configure Application Insights'
            inputs:
              azureSubscription: $(azureSubscription)
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Create Application Insights instance
                APPINSIGHTS_KEY=$(az monitor app-insights component create \
                  --app $(webAppName)-insights \
                  --location "East US" \
                  --resource-group $(resourceGroupName) \
                  --application-type web \
                  --query instrumentationKey \
                  --output tsv)
                
                # Configure App Insights for the web app
                az webapp config appsettings set \
                  --name $(webAppName) \
                  --resource-group $(resourceGroupName) \
                  --settings APPINSIGHTS_INSTRUMENTATIONKEY="$APPINSIGHTS_KEY"
```

---

## Best Practices

### Production-Ready Pipeline Template

```yaml
# Template: api-deployment-template.yml
parameters:
- name: environment
  type: string
- name: azureSubscription
  type: string
- name: webAppName
  type: string
- name: resourceGroupName
  type: string
- name: containerRegistry
  type: string
  default: ''
- name: runIntegrationTests
  type: boolean
  default: false

stages:
- stage: Deploy_${{ parameters.environment }}
  displayName: 'Deploy to ${{ parameters.environment }}'
  
  jobs:
  - deployment: Deploy
    displayName: 'Deploy API to ${{ parameters.environment }}'
    environment: '${{ parameters.environment }}'
    pool:
      vmImage: 'ubuntu-latest'
    
    strategy:
      runOnce:
        deploy:
          steps:
          # Health check before deployment
          - task: AzureCLI@2
            displayName: 'Pre-deployment health check'
            inputs:
              azureSubscription: ${{ parameters.azureSubscription }}
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                # Check if app is currently healthy
                CURRENT_STATUS=$(az webapp show \
                  --name ${{ parameters.webAppName }} \
                  --resource-group ${{ parameters.resourceGroupName }} \
                  --query "state" \
                  --output tsv)
                
                echo "Current app status: $CURRENT_STATUS"
                
                if [ "$CURRENT_STATUS" != "Running" ]; then
                  echo "Warning: App is not in Running state"
                fi

          # Deploy application
          - task: AzureWebApp@1
            displayName: 'Deploy API'
            inputs:
              azureSubscription: ${{ parameters.azureSubscription }}
              appType: 'webAppLinux'
              appName: ${{ parameters.webAppName }}
              package: '$(Pipeline.Workspace)/drop/**/*.zip'
              deploymentMethod: 'auto'

          # Post-deployment verification
          - task: PowerShell@2
            displayName: 'Post-deployment health check'
            inputs:
              targetType: 'inline'
              script: |
                $healthEndpoint = "https://${{ parameters.webAppName }}.azurewebsites.net/health"
                $maxAttempts = 10
                $attempt = 1
                
                Write-Host "Checking health endpoint: $healthEndpoint"
                
                do {
                  try {
                    $response = Invoke-RestMethod -Uri $healthEndpoint -Method Get -TimeoutSec 30
                    Write-Host "Health check attempt $attempt succeeded"
                    Write-Host "Response: $response"
                    break
                  }
                  catch {
                    Write-Host "Health check attempt $attempt failed: $($_.Exception.Message)"
                    if ($attempt -eq $maxAttempts) {
                      Write-Error "Health check failed after $maxAttempts attempts"
                      exit 1
                    }
                    Start-Sleep -Seconds 30
                  }
                  $attempt++
                } while ($attempt -le $maxAttempts)

          # Run integration tests if specified
          - ${{ if eq(parameters.runIntegrationTests, true) }}:
            - task: DotNetCoreCLI@2
              displayName: 'Run integration tests'
              inputs:
                command: 'test'
                projects: '**/IntegrationTests.csproj'
                arguments: '--configuration Release --logger trx --results-directory $(Agent.TempDirectory)'
              env:
                API_BASE_URL: 'https://${{ parameters.webAppName }}.azurewebsites.net'

            - task: PublishTestResults@2
              displayName: 'Publish integration test results'
              condition: succeededOrFailed()
              inputs:
                testResultsFormat: 'VSTest'
                testResultsFiles: '**/*.trx'
                searchFolder: '$(Agent.TempDirectory)'
```

### Multi-Environment Pipeline

```yaml
# Main pipeline using the template
trigger:
- main
- develop

variables:
  buildConfiguration: 'Release'
  vmImageName: 'ubuntu-latest'

stages:
- stage: Build
  displayName: 'Build Application'
  jobs:
  - job: Build
    displayName: 'Build'
    pool:
      vmImage: $(vmImageName)
    
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Restore, build, test, and publish'
      inputs:
        command: 'custom'
        custom: 'run'
        arguments: 'build-and-test'
    
    - task: DotNetCoreCLI@2
      displayName: 'Publish application'
      inputs:
        command: 'publish'
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    
    - upload: $(Build.ArtifactStagingDirectory)
      artifact: drop

# Development Environment
- template: api-deployment-template.yml
  parameters:
    environment: 'development'
    azureSubscription: 'dev-service-connection'
    webAppName: 'your-api-dev'
    resourceGroupName: 'your-api-dev-rg'
    runIntegrationTests: true

# Staging Environment (only for main branch)
- ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
  - template: api-deployment-template.yml
    parameters:
      environment: 'staging'
      azureSubscription: 'staging-service-connection'
      webAppName: 'your-api-staging'
      resourceGroupName: 'your-api-staging-rg'
      runIntegrationTests: true

# Production Environment (manual approval required)
- ${{ if eq(variables['Build.SourceBranchName'], 'main') }}:
  - template: api-deployment-template.yml
    parameters:
      environment: 'production'
      azureSubscription: 'prod-service-connection'
      webAppName: 'your-api-prod'
      resourceGroupName: 'your-api-prod-rg'
      runIntegrationTests: false
```

---

## Common Configuration Patterns

### Database Migration Pipeline

```yaml
- task: AzureCLI@2
  displayName: 'Run database migrations'
  inputs:
    azureSubscription: $(azureSubscription)
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Set environment variables for EF Core migrations
      export ConnectionStrings__DefaultConnection="$(databaseConnectionString)"
      
      # Install EF Core tools
      dotnet tool install --global dotnet-ef
      
      # Run migrations
      dotnet ef database update --startup-project $(Pipeline.Workspace)/drop --verbose
```

### API Documentation Deployment

```yaml
- task: AzureCLI@2
  displayName: 'Deploy API documentation'
  inputs:
    azureSubscription: $(azureSubscription)
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Enable Swagger UI in production (if needed)
      az webapp config appsettings set \
        --name $(webAppName) \
        --resource-group $(resourceGroupName) \
        --settings SwaggerUI__Enabled="true"
      
      # Create API documentation in static web app
      az storage blob upload-batch \
        --destination '$web' \
        --source './docs/api' \
        --account-name $(docsStorageAccount)
```

---

This comprehensive guide covers the most common scenarios for deploying APIs through Azure DevOps. Choose the patterns that best fit your architecture and modify them according to your specific requirements.

### Key Points to Remember:

1. **Always use proper secret management** with Azure Key Vault
2. **Implement health checks** for reliable deployments
3. **Use staging slots** for zero-downtime deployments
4. **Configure monitoring and logging** from the start
5. **Test your deployment pipeline** in non-production environments first
6. **Use templates** to maintain consistency across environments
7. **Implement proper authentication and authorization** for production APIs