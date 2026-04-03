# Azure CI/CD Guide (Intermediate to Senior)

This guide is designed for engineers preparing for Azure CI/CD interviews and real project implementation.
Each section includes:
- Concept explanation
- Real-time scenario
- Practical YAML example with comments

---

## 1) What is CI/CD in Azure DevOps?

Short answer:
CI (Continuous Integration) validates code changes quickly. CD (Continuous Delivery/Deployment) automates release to environments.

Real-time scenario:
A fintech team pushes code daily. CI runs tests and security checks, CD deploys to dev and staging automatically, and production after approval.

Explanation:
Azure DevOps Pipelines helps standardize build and deployment across repos and teams.

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main

pool:
  vmImage: ubuntu-latest

stages:
  - stage: BuildAndTest
    jobs:
      - job: Build
        steps:
          - checkout: self
          - script: echo "Run build and tests here"
            displayName: Build and unit tests
```

---

## 2) CI pipeline for .NET API (build, test, publish)

Real-time scenario:
A team maintains a .NET Web API. Every push to main should compile, run tests, and publish build artifacts.

```yaml
trigger:
  branches:
    include:
      - main
      - feature/*

pool:
  vmImage: ubuntu-latest

variables:
  buildConfiguration: Release

stages:
  - stage: CI
    jobs:
      - job: DotNetBuildTest
        steps:
          - task: UseDotNet@2
            inputs:
              packageType: sdk
              version: 8.0.x

          - script: dotnet restore
            displayName: Restore packages

          - script: dotnet build --configuration $(buildConfiguration) --no-restore
            displayName: Build solution

          - script: dotnet test --configuration $(buildConfiguration) --no-build --collect:"XPlat Code Coverage"
            displayName: Run tests with coverage

          - script: dotnet publish src/MyApi/MyApi.csproj --configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/app
            displayName: Publish app

          - task: PublishBuildArtifacts@1
            inputs:
              pathToPublish: $(Build.ArtifactStagingDirectory)/app
              artifactName: api-drop
              publishLocation: Container
```

---

## 3) Multi-stage CD pipeline with approvals

Real-time scenario:
Retail platform deploys automatically to dev and qa, but production needs manual approval.

```yaml
stages:
  - stage: Build
    jobs:
      - job: BuildJob
        steps:
          - script: echo "Build done"

  - stage: Deploy_Dev
    dependsOn: Build
    jobs:
      - deployment: DeployDev
        environment: myapp-dev
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to Dev"

  - stage: Deploy_QA
    dependsOn: Deploy_Dev
    jobs:
      - deployment: DeployQA
        environment: myapp-qa
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to QA"

  - stage: Deploy_Prod
    dependsOn: Deploy_QA
    jobs:
      - deployment: DeployProd
        environment: myapp-prod # configure approvals/checks in Azure DevOps Environment
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Deploying to Production"
```

---

## 4) How do you manage secrets securely?

Short answer:
Use Azure Key Vault and variable groups. Never hardcode secrets in YAML.

Real-time scenario:
Production DB password and API keys are fetched from Key Vault at runtime.

```yaml
variables:
  - group: app-shared-vars

steps:
  - task: AzureKeyVault@2
    inputs:
      azureSubscription: my-service-connection
      KeyVaultName: kv-myapp-prod
      SecretsFilter: DbConnectionString,JwtSigningKey
      RunAsPreJob: true

  - script: |
      echo "Secrets loaded from Key Vault"
      # Use $(DbConnectionString) in tasks; do not print secret values
    displayName: Load secrets
```

---

## 5) Infrastructure as Code deployment with Bicep

Real-time scenario:
Before app deployment, pipeline provisions App Service, App Insights, and SQL firewall rules.

```yaml
steps:
  - task: AzureCLI@2
    displayName: Deploy infrastructure with Bicep
    inputs:
      azureSubscription: my-service-connection
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        az deployment group create \
          --resource-group rg-myapp-dev \
          --template-file infra/main.bicep \
          --parameters environment=dev appName=myapp
```

---

## 6) Deploy ASP.NET Core API to Azure App Service

Real-time scenario:
After successful tests, API package is deployed to Linux App Service.

```yaml
steps:
  - task: AzureWebApp@1
    displayName: Deploy API to App Service
    inputs:
      azureSubscription: my-service-connection
      appType: webAppLinux
      appName: myapp-api-dev
      package: $(Pipeline.Workspace)/api-drop/**/*.zip
```

---

## 7) Deploy containerized app to Azure Container Apps or AKS

Real-time scenario:
Microservice image is built, pushed to ACR, then deployed to Container Apps.

```yaml
steps:
  - task: Docker@2
    displayName: Build and push image
    inputs:
      containerRegistry: my-acr-service-connection
      repository: myapp/orders-api
      command: buildAndPush
      Dockerfile: src/OrdersApi/Dockerfile
      tags: |
        $(Build.BuildId)
        latest

  - task: AzureCLI@2
    displayName: Deploy to Container Apps
    inputs:
      azureSubscription: my-service-connection
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        az containerapp update \
          --name ca-orders-api \
          --resource-group rg-myapp-dev \
          --image myacr.azurecr.io/myapp/orders-api:$(Build.BuildId)
```

---

## 8) How do you implement branch strategy in CI/CD?

Short answer:
Use branch policies + PR validation + protected main branch.

Real-time scenario:
Only reviewed and validated pull requests can merge to main.

```yaml
pr:
  branches:
    include:
      - main

trigger:
  branches:
    include:
      - main
      - release/*

steps:
  - script: echo "PR validation pipeline"
```

Explanation:
Combine this with Azure Repos branch policies: required reviewers, successful build, and no direct push to main.

---

## 9) How do you speed up pipelines?

Short answer:
Use caching, parallel jobs, incremental builds, and selective triggers.

Real-time scenario:
Large monorepo build reduced from 25 minutes to 10 minutes.

```yaml
steps:
  - task: Cache@2
    inputs:
      key: 'nuget | "$(Agent.OS)" | **/*.csproj'
      restoreKeys: |
        nuget | "$(Agent.OS)"
      path: ~/.nuget/packages
    displayName: Cache NuGet packages

  - script: dotnet restore
  - script: dotnet build --no-restore
```

---

## 10) How do you add quality gates (tests, coverage, security)?

Real-time scenario:
Build should fail if test coverage below threshold or critical vulnerabilities are found.

```yaml
steps:
  - script: dotnet test --collect:"XPlat Code Coverage"
    displayName: Unit tests with coverage

  - script: |
      dotnet tool install --global dotnet-reportgenerator-globaltool
      reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
    displayName: Generate coverage report

  - script: dotnet list package --vulnerable --include-transitive
    displayName: Dependency vulnerability check
```

---

## 11) How do you implement deployment slots and zero-downtime release?

Real-time scenario:
Deploy to staging slot, run smoke tests, then swap to production.

```yaml
steps:
  - task: AzureWebApp@1
    displayName: Deploy to staging slot
    inputs:
      azureSubscription: my-service-connection
      appType: webAppLinux
      appName: myapp-api-prod
      deployToSlotOrASE: true
      resourceGroupName: rg-myapp-prod
      slotName: staging
      package: $(Pipeline.Workspace)/api-drop/**/*.zip

  - script: echo "Run smoke tests against staging slot URL"
    displayName: Smoke tests

  - task: AzureAppServiceManage@0
    displayName: Swap staging to production
    inputs:
      azureSubscription: my-service-connection
      Action: Swap Slots
      WebAppName: myapp-api-prod
      ResourceGroupName: rg-myapp-prod
      SourceSlot: staging
```

---

## 12) How do you support rollback strategy?

Short answer:
Keep previous artifact versions, use slot swap rollback, or redeploy last known good image/tag.

Real-time scenario:
Production issue found after release; rollback in under 5 minutes.

```yaml
steps:
  - script: |
      echo "Rollback by redeploying previous image tag"
      echo "Example: myacr.azurecr.io/myapp/orders-api:12345"
    displayName: Rollback strategy reference
```

---

## 13) How do service connections work in Azure DevOps?

Short answer:
Service connections securely allow pipelines to access Azure subscriptions, ACR, Key Vault, etc.

Real-time scenario:
One pipeline deploys dev resources with dev service connection and prod with restricted prod connection.

```yaml
# Example task using service connection
- task: AzureCLI@2
  inputs:
    azureSubscription: prod-service-connection
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: az account show
```

---

## 14) How do you structure reusable templates in YAML?

Real-time scenario:
10 microservices share same build steps to avoid duplicated YAML.

```yaml
# main pipeline
stages:
  - template: templates/build-stage.yml
    parameters:
      projectPath: src/OrdersApi/OrdersApi.csproj

# templates/build-stage.yml
# parameters:
# - name: projectPath
#   type: string
# stages:
# - stage: Build
#   jobs:
#   - job: Build
#     steps:
#     - script: dotnet build ${{ parameters.projectPath }}
```

---

## 15) End-to-end Azure CI/CD best-practice flow (senior-level)

Recommended flow:
1. PR validation (lint, test, SAST)
2. Build and version artifact/image
3. Publish artifact
4. Deploy infra changes (Bicep/Terraform)
5. Deploy app to dev automatically
6. Integration and smoke tests
7. Promote to qa
8. Manual approval for prod
9. Slot-based prod deployment
10. Monitoring and automatic rollback trigger

Senior interview tip:
Discuss trade-offs: speed vs safety, deployment frequency vs stability, centralized templates vs team autonomy.

---

## Rapid Fire Interview Questions

1. Difference between Continuous Delivery and Continuous Deployment?
2. Why keep infra deployment separate from app deployment sometimes?
3. How do you secure production secrets in pipelines?
4. How do you prevent accidental production deployment from feature branches?
5. How do you implement blue-green vs canary in Azure?
6. How do you trace a deployment to a specific commit and work item?
7. What metrics define CI/CD success?
8. How do you reduce MTTR during failed release?

---

## Practice Tasks

1. Create a multi-stage YAML pipeline for a .NET API.
2. Add Key Vault secret retrieval and remove hardcoded config.
3. Add deployment to staging slot and production swap.
4. Add rollback stage using previous artifact/image.
5. Add branch-based conditions for dev, qa, and prod.
