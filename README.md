# Heineken.ATC.SSP.InfraTemplate

Infrastructure templates and reusable Azure DevOps YAML templates for SSP services.

This repository is no longer a single-template Function App deployment. It now contains:

- Multiple ARM templates for different SSP components (API, Function App, Orchestrator, UI)
- Build pipeline templates for backend, Function App, orchestrator, and UI services
- Environment deployment templates for Function App deployments to dev, acc, and prod

## Repository Structure

```text
.
|-- ARM/
|   |-- azuredeploy.backendsvc.json
|   |-- azuredeploy.fapsvc.json
|   |-- azuredeploy.orchsvc.json
|   `-- azuredeploy.uisvc.json
|-- buildbackendsvc.yaml
|-- buildfapsvc.yaml
|-- buildorchsvc.yaml
|-- builduisvc.yaml
|-- deployfaptodev.yaml
|-- deployfaptoacc.yaml
`-- deployfaptoprod.yaml
```

## ARM Templates

### ARM/azuredeploy.backendsvc.json

- Service Bus namespace and queues
- Virtual Network + Subnet + NSG
- Logic Apps for scheduled operations
- AFRA tagging model on resources

### ARM/azuredeploy.fapsvc.json

Deploys Flex Consumption Function App infrastructure:

- Linux Function App (dotnet-isolated; runtime 8.0/9.0/10)
- Flex Consumption plan (FC1)
- Deployment blob container in Storage
- Application Insights wiring
- Key Vault access policy for managed identity
- VNet subnet integration
- AFRA tags

### ARM/azuredeploy.orchsvc.json

Deploys orchestrator service infrastructure, including:

- App Service Web App
- SQL Server + SQL Database
- Storage Account dependencies
- Key Vault and app settings integration
- Logic Apps used by orchestrator workflows
 Store the declared Azure, ARM, Service Bus, storage, Key Vault, AFRA, and SonarQube values as environment or repository secrets. The build workflow requires `SONAR_HOST_URL` and `SONAR_TOKEN`. `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are used with `azure/login` OIDC; no Azure client secret is needed.
### ARM/azuredeploy.uisvc.json

Deploys UI hosting infrastructure:

- Web App for SPA hosting
- Node stack configuration (default `~16`)
- Optional `staging` slot when environment is `acc`

## Azure DevOps YAML Templates

### Build templates

- buildbackendsvc.yaml
   - .NET build/test/publish flow for backend service
   - SonarQube prepare/analyze/publish
   - Publishes WEB artifact and ARM artifact
   - Packs and pushes selected internal NuGet packages

- buildfapsvc.yaml
   - .NET Function projects build/test/publish flow
   - SonarQube integration
   - Publishes WEB and ARM artifacts

- buildorchsvc.yaml
   - .NET orchestrator build/test/publish flow
   - SonarQube integration
   - Publishes WEB and ARM artifacts
   - Packs and pushes orchestrator common NuGet package

- builduisvc.yaml
   - Node build pipeline for SPA
   - PR policy check (master target only allowed from develop)
   - npm cache, install, build, and PR test
   - SonarQube CLI analysis on PRs
   - Publishes WEB and ARM artifacts
   - Includes PR-to-ACC staging deployment job for regression testing

### Function App deployment templates

- deployfaptodev.yaml
   - Stage: Deploy_to_Development
   - Trigger condition: source branch `develop`
   - Uses Azure subscription `HEI-TOOLCHAIN-INNOV-TST`
   - Deploys ARM/azuredeploy.fapsvc.json then deploys function package

- deployfaptoacc.yaml
   - Stage: Deploy_to_Acceptance
   - Trigger condition: source branch `master`
   - Uses Azure subscription `Triple POD HEIWAY`
   - Deploys ARM/azuredeploy.fapsvc.json then deploys function package

- deployfaptoprod.yaml
   - Stage: Deploy_to_Production
   - Trigger condition: source branch `master` (after acceptance stage)
   - Uses Azure subscription `Triple POD HEIWAY`
   - Deploys ARM/azuredeploy.fapsvc.json then deploys function package

## Environment and Branch Mapping

- Development: branch `develop`
- Acceptance: branch `master`
- Production: branch `master` after acceptance

## Required Azure DevOps Setup

Before consuming these templates, ensure your project has:

1. Service connections:
    - `HEI-TOOLCHAIN-INNOV-TST`
    - `Triple POD HEIWAY`

2. Variable groups and variables used in templates, including:
    - Secret groups such as `Secrets Dev`, `Secrets Acc`, `Secrets Prod`
    - ARM naming variables (for example resource group, app insights, key vault, vnet/subnet)
    - AFRA tagging variables
    - Build variables such as `SolutionName`

3. SonarQube service connection:
    - `Heineken SonarQube`

4. Artifact contract expected by deploy templates:
    - `WEB` artifact with deployable package(s)
    - `ARM` artifact with ARM templates

## Example Template Consumption

Use this repository as a template repository in your service pipeline:

```yaml
resources:
   repositories:
      - repository: Heineken.ATC.SSP.InfraTemplate
         type: git
         name: <project>/Heineken.ATC.SSP.InfraTemplate

stages:
   - template: buildfapsvc.yaml@Heineken.ATC.SSP.InfraTemplate
      parameters:
         dotNETSDKVersion: 8.0.x

   - template: deployfaptodev.yaml@Heineken.ATC.SSP.InfraTemplate
      parameters:
         functionsAppName: <your-function-app-name>
```

## ARM Validation (Optional)

Validate templates before merging changes:

```bash
az deployment group validate \
   --resource-group <rg-name> \
   --template-file ARM/azuredeploy.fapsvc.json \
   --parameters @<parameters-file>.json
```

Repeat for backend, orchestrator, and UI templates as needed.

## Notes

- Templates implement AFRA tagging fields across resources for governance.
- Function App deploy templates target Flex Consumption (`isFlexConsumption: true`).
- Some build templates publish ARM artifacts from different paths (`ARM` vs `pipelines`) depending on consumer pipeline layout. Confirm the `PathtoPublish` path matches your repository structure.

## GitHub Actions

The Azure DevOps Function App templates have separate GitHub Actions counterparts in `.github/workflows`:

- `buildfapsvc.yml` builds, tests, publishes, and uploads the `WEB` and `ARM` artifacts.
- `deployfaptodev.yml` deploys the `WEB` and `ARM` artifacts to the `Development` environment.
- `deployfaptoacc.yml` deploys them to `Acceptance`.
- `deployfaptoprod.yml` deploys them to `Production`.

The files use `workflow_call` so a service repository can compose them into one pipeline while retaining separate source files. A caller should run the build first and make the deployment jobs depend on it, for example:

```yaml
name: Function App CI/CD

on:
   push:
      branches: [develop, master]

jobs:
   build:
      uses: emtwentyco/templatefile/.github/workflows/buildfapsvc.yml@main
      with:
         dotnet-sdk-version: 10.0.x
         solution-name: <your-solution-name>
         azureArtifactsFeedUrl: https://pkgs.dev.azure.com/<organization>/<project>/_packaging/<feed>/nuget/v3/index.json

   development:
      if: github.ref == 'refs/heads/develop'
      needs: build
      uses: emtwentyco/templatefile/.github/workflows/deployfaptodev.yml@main
      with:
         functions-app-name: <your-function-app-name>
         solution-name: <your-solution-name>
      secrets: inherit
```

Create matching jobs for Acceptance and Production, with `needs` dependencies, and configure GitHub Environments named `Development`, `Acceptance`, and `Production`. Store the declared Azure, ARM, Service Bus, storage, Key Vault, AFRA, and SonarQube values as environment or repository secrets. The build workflow configures both Azure Artifacts and GitHub Packages as NuGet sources. It requires `AZURE_DEVOPS_PAT` to restore from Azure Artifacts. Set `GITHUB_PACKAGES_TOKEN` with package write permission, or allow the workflow `GITHUB_TOKEN` to write packages. Pass `nugetPackagePath` when an existing `.nupkg` should be pushed to GitHub Packages. `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are used with `azure/login` OIDC; no Azure client secret is needed.

## GitHub Actions

The reusable workflow at `.github/workflows/function-app.yml` replaces the Function App build and deployment templates. Call it from a service repository workflow and pass the required Azure and application values:

```yaml
name: Function App CI/CD

on:
   push:
      branches: [develop, master]

jobs:
   function-app:
      uses: emtwentyco/templatefile/.github/workflows/function-app.yml@main
      with:
         dotnet-version: 10.0.x
         functions-app-name: <your-function-app-name>
         solution-name: <your-solution-name>
      secrets: inherit
```

Configure GitHub Environments named `Development`, `Acceptance`, and `Production`, then add the secrets declared in the workflow. Configure an Entra application with federated credentials for GitHub Actions and grant it the deployment permissions required for the target resource group. The workflow uses OIDC through `azure/login`; no Azure client secret is required.

## Change Log

- 2026-06-20: README rewritten to reflect current multi-template, multi-pipeline repository structure and deployment flow.
