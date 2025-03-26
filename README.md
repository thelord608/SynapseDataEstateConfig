### **Step-by-Step Guide to Automating Config Data Deployment to ADLS Using Azure DevOps**

This guide provides a structured approach to automate the deployment of configuration data to Azure Data Lake Storage (ADLS) through Azure DevOps CI/CD pipelines.

---

## **Step 1: Understanding the Requirements**

The goal is to automate the deployment of configuration data from the **Azure DevOps Repository** (`SynapseDataEstateConfig`) to **Azure Data Lake Storage (ADLS Gen2)** (`usekfdtsynadls01`, `config` container). The pipeline should:

- Trigger on new changes in the **master** branch.
- Copy configuration files to ADLS automatically.
- Have separate release stages for **Dev/Test** and **Prod**.
- Overwrite existing files in ADLS with the latest changes.
- Ignore files in ADLS that do not exist in the repository.

---

## **Step 2: Prerequisites**

Ensure the following are configured:

### **1. Azure DevOps Setup**

- **Repo:** `SynapseDataEstateConfig`
- **Branch:** `master`
- **Service Connection:** `ADLS-Config-ServiceConnection` (Managed Identity authentication)
- **Agent Pool:** `Default`
- **Self-Hosted Agent VM:** `USE-SYNAPSEDT01`

### **2. Azure Data Lake Storage (ADLS)**

- **Storage Account:** `usekfdtsynadls01`
- **Container:** `config`
- **Folder Structure:**
  ```
  config/
  ├── coresignal/
  ├── KFAS/
  ├── CLA/
  ├── DepersonalisationRequests/
  ├── Pay/
  ├── SuccessProfile/
  ├── uat/
  ```

### **3. Required Permissions for Azure DevOps Agent (`USE-SYNAPSEDT01`)**
The agent **must have** the following permissions on the `config` container:
- ✅ **Read (`R`)** - To list files before uploading.
- ✅ **Write (`W`)** - To copy new files and overwrite existing ones.
- ✅ **Execute (`X`)** - To navigate directories.
- ❗ **Ensure the Mask allows Write (`W`)**, otherwise the agent cannot modify files.

**How to Fix Permission Issues?**
1. **Go to Azure Portal → Storage Account → Containers → `config` → Manage ACL.**
2. **Grant `Read (R)`, `Write (W)`, and `Execute (X)` to `USE-SYNAPSEDT01`.**
3. **Ensure the Mask row also includes `Write (W)`.**
4. **Click Save.**

---

## **Step 3: Create the CI Pipeline (Build Pipeline)**

This pipeline detects changes in the **master** branch, packages the configuration files, and publishes them as an artifact.

### **1. Create `ci-pipeline.yml`**

Navigate to **Azure DevOps → Pipelines → New Pipeline**, choose **YAML**, and add:

```yaml
trigger:
  branches:
    include:
      - master

pool:
  name: Default  # Self-hosted agent
  demands:
    - Agent.Name -equals USE-SYNAPSEDT01  # Explicitly specify the agent

steps:
- task: PublishBuildArtifacts@1
  displayName: 'Publish Config Files as Artifact'
  inputs:
    pathToPublish: '$(Build.SourcesDirectory)'  # Publish the entire repo
    artifactName: 'config-artifact'
```

### **2. Save and Run the Pipeline**

- This will detect changes, package the config files, and store them as an artifact.
- Verify that the artifact appears in **Azure DevOps → Pipelines → Artifacts**.

---

## **Step 4: Create the CD Pipeline (Release Pipeline)**

This pipeline will fetch the **config-artifact** and deploy it to ADLS.

### **1. Create `cd-pipeline.yml`**

Navigate to **Azure DevOps → Pipelines → Releases**, choose **New Release Pipeline**, and paste:

```yaml
trigger:
  branches:
    include:
      - master  # Trigger only on master branch commits

pool:
  name: Default  # Using self-hosted Windows agent
  demands:
    - Agent.Name -equals USE-SYNAPSEDT01  # Ensuring it runs on Windows agent

stages:
- stage: Install_Dependencies
  displayName: 'Install Dependencies'
  jobs:
  - job: SetupAgent
    steps:
    - task: PowerShell@2
      displayName: 'Install Required Dependencies'
      inputs:
        targetType: 'inline'
        script: |
          # Install AzCopy
          if (-not (Test-Path "C:\Program Files (x86)\Microsoft SDKs\Azure\AzCopy\azcopy.exe")) {
            Write-Host "Installing AzCopy..."
            Invoke-WebRequest -Uri https://aka.ms/downloadazcopy-v10-windows -OutFile azcopy.zip
            Expand-Archive -Path azcopy.zip -DestinationPath C:\AzCopy
          }
          
          # Install Azure PowerShell Module
          if (-not (Get-Module -ListAvailable -Name Az)) {
            Write-Host "Installing Azure PowerShell Module..."
            Install-Module -Name Az -AllowClobber -Scope CurrentUser -Force
          }
          
          # Verify installations
          azcopy --version
          Get-Module -ListAvailable Az*

- stage: Deploy_Dev
  displayName: 'Deploy to Dev Environment'
  dependsOn: Install_Dependencies  # Ensuring dependencies are installed first
  jobs:
  - job: DeployToADLS
    steps:
    - task: AzureFileCopy@4
      displayName: 'Copy CoreSignal Config Files to ADLS'
      inputs:
        SourcePath: '$(Pipeline.Workspace)/config-artifact/CoreSignal'
        azureSubscription: 'ADLS-Config-ServiceConnection'
        Destination: 'AzureBlob'
        storage: 'usekfdtsynadls01'
        containerName: 'config'
        BlobPrefix: 'coresignal/'  # Ensuring CoreSignal files go under coresignal folder
        Overwrite: true

    - task: AzureFileCopy@4
      displayName: 'Copy KFAS Config Files to ADLS'
      inputs:
        SourcePath: '$(Pipeline.Workspace)/config-artifact/KFAS'
        azureSubscription: 'ADLS-Config-ServiceConnection'
        Destination: 'AzureBlob'
        storage: 'usekfdtsynadls01'
        containerName: 'config'
        BlobPrefix: 'KFAS/'  # Ensuring KFAS files go under KFAS folder
        Overwrite: true
```
### OR
```yaml
trigger:
  branches:
    include:
      - main  # Runs when changes are pushed to the main branch

pool:
  name: Default  # Use self-hosted agent

stages:
- stage: CheckDependencies
  displayName: "Check Required Dependencies on Self-Hosted VM"
  jobs:
  - job: VerifyDependencies
    displayName: "Ensure Azure CLI and Az Modules are Installed"
    steps:
    - task: PowerShell@2
      displayName: "Check Azure CLI Installation"
      inputs:
        targetType: "inline"
        script: |
          if (-not (Get-Command az -ErrorAction SilentlyContinue)) {
            Write-Host "##vso[task.logissue type=error]Azure CLI is not installed. Install it from https://aka.ms/installazurecliwindows"
            exit 1
          } else {
            Write-Host "Azure CLI is installed: $(az --version)"
          }

    - task: PowerShell@2
      displayName: "Check Az PowerShell Modules"
      inputs:
        targetType: "inline"
        script: |
          if (-not (Get-Module -ListAvailable Az)) {
            Write-Host "##vso[task.logissue type=error]Az PowerShell Module is not installed. Install it using 'Install-Module -Name Az -AllowClobber -Scope AllUsers -Force'"
            exit 1
          } else {
            Write-Host "Az PowerShell Module is installed."
          }
- stage: Build
  displayName: "Build Stage"
  jobs:
  - job: PackageFiles
    displayName: "Prepare Files for Deployment"
    steps:
    - task: CopyFiles@2
      displayName: "Copy JSON Files to Staging"
      inputs:
        SourceFolder: "$(Build.SourcesDirectory)"
        Contents: "**/*.json"  # Copy all JSON files while preserving folders
        TargetFolder: "$(Build.ArtifactStagingDirectory)"

    - task: PublishBuildArtifacts@1
      displayName: "Publish JSON Files as Artifacts"
      inputs:
        pathToPublish: "$(Build.ArtifactStagingDirectory)"
        artifactName: "json-files"

- stage: Deploy
  displayName: "Deploy JSON Files to ADLS"
  dependsOn: Build  # Ensures deployment runs only after build completes successfully
  condition: succeeded()  # Ensures deployment only runs if build is successful
  jobs:
  - job: UploadToADLS
    displayName: "Upload JSON Files using AzureBlob File Copy"
    steps:
    - download: current
      artifact: json-files

    - task: AzureFileCopy@4
      displayName: "Copy Files to Azure Blob Storage"
      inputs:
        SourcePath: "$(Pipeline.Workspace)/json-files"
        azureSubscription: "Your-Azure-Service-Connection"
        Destination: "AzureBlob"
        storage: "yourstorageaccount"
        ContainerName: "yourcontainer"
        BlobPrefix: "uploaded-files/"
        Overwrite: true

```
### **2. Save and Deploy**

- This will fetch the artifact and copy the files to ADLS.
- The **Dev stage** runs automatically.
- The **Prod stage** can be triggered manually or automatically based on conditions.

---------------------------------------------------
### **Install Azure CLI and Az PowerShell Modules on Self-Hosted Windows VM**
- Before running the pipeline, install the necessary tools on the self-hosted Windows VM.

- Install Azure CLI on Windows VM
  Run the following PowerShell command as Administrator:
```powershell
# Download and install Azure CLI
Invoke-WebRequest -Uri "https://aka.ms/installazurecliwindows" -OutFile "AzureCLI.msi"
Start-Process -FilePath "msiexec.exe" -ArgumentList "/i AzureCLI.msi /quiet /norestart" -Wait

# Verify installation
az --version
```
- Install Az PowerShell Modules
  Run the following PowerShell command to install the required Az modules:
```powershell
  # Set execution policy
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

# Install Az PowerShell module
Install-Module -Name Az -AllowClobber -Scope AllUsers -Force

# Verify installation
Get-Module -ListAvailable Az*
```
- We allow locally created scripts to run while still blocking scripts from the internet unless signed via execution policy.

### **Validate ADLS Permissions**
- Check ACL Permissions Using Azure CLI
Run the following command on the VM to verify permissions:
```
az storage fs access show --account-name yourstorageaccount --file-system yourcontainer --path CLA/

```
- If you need to grant write permissions to the Managed Identity of the VM:
```
az storage fs access set --account-name yourstorageaccount \
  --file-system yourcontainer \
  --path CLA/ \
  --permissions rwx \
  --group

```
### ** Assign "Storage Blob Data Contributor" to the VM’s Managed Identity**

- The "Storage Blob Data Contributor" role in Azure Role-Based Access Control (RBAC) grants read, write, and delete permissions on Azure Blob Storage to the Managed Identity of your self-hosted agent VM.

- ✅ Why Is It Required?
- Since the Azure DevOps pipeline is using Managed Identity (az login --identity), the VM’s Managed Identity needs explicit permission to:

- Read existing blobs in the storage account.
- Upload and overwrite blobs (write access).
- Delete blobs if needed.
- Without this role, file uploads will fail with an error like:
```
AuthorizationPermissionMismatch: This request is not authorized to perform this operation
```
- How to Assign "Storage Blob Data Contributor" to the VM’s Managed Identity
Run the following Azure CLI command:
```
az role assignment create --assignee <vm-managed-identity-id> \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Storage/storageAccounts/<storage-account>"

```

** Alternative: Assign Role via Azure Portal **
- Go to Azure Portal → Storage Account → IAM (Access Control)
- Click "Add Role Assignment"
- Role: Storage Blob Data Contributor
- Assign to: Your VM’s Managed Identity
- Save the role assignment.
- ✅ Now, the VM can authenticate to Azure Storage using its Managed Identity!

======================================================================================================================================


Build Pipeline (azure-pipelines-build.yml)
```yaml
trigger:
  branches:
    include:
      - main  # Trigger on changes to the main branch

pool:
  name: Default  # Use self-hosted agent

stages:
- stage: CheckDependencies
  displayName: "Check Required Dependencies on Self-Hosted VM"
  jobs:
  - job: VerifyDependencies
    displayName: "Ensure Azure CLI and Az Modules are Installed"
    steps:
    - task: PowerShell@2
      displayName: "Check Azure CLI Installation"
      inputs:
        targetType: "inline"
        script: |
          if (-not (Get-Command az -ErrorAction SilentlyContinue)) {
            Write-Host "##vso[task.logissue type=error]Azure CLI is not installed. Install it from https://aka.ms/installazurecliwindows"
            exit 1
          } else {
            Write-Host "Azure CLI is installed: $(az --version)"
          }

    - task: PowerShell@2
      displayName: "Check Az PowerShell Modules"
      inputs:
        targetType: "inline"
        script: |
          if (-not (Get-Module -ListAvailable Az)) {
            Write-Host "##vso[task.logissue type=error]Az PowerShell Module is not installed. Install it using 'Install-Module -Name Az -AllowClobber -Scope AllUsers -Force'"
            exit 1
          } else {
            Write-Host "Az PowerShell Module is installed."
          }
- stage: Build
  displayName: "Prepare Files"
  jobs:
  - job: PrepareArtifacts
    displayName: "Package JSON Files"
    steps:
    - task: CopyFiles@2
      displayName: "Copy JSON Files to Staging"
      inputs:
        SourceFolder: "$(Build.SourcesDirectory)"
        Contents: "**/*.json"  # Copy all JSON files
        TargetFolder: "$(Build.ArtifactStagingDirectory)"

    - task: PublishBuildArtifacts@1
      displayName: "Publish JSON Files"
      inputs:
        pathToPublish: "$(Build.ArtifactStagingDirectory)"
        artifactName: "json-files"


```
or
```
trigger:
  branches:
    include:
      - feature/pj/172-deployment_config_data_to_ADLS  # Trigger on changes to this specific feature branch

pool:
  name: Default  # Use self-hosted agent pool
  demands:
    - Agent.Name -equals USE-SYNAPSEDT01  # Ensure only USE-SYNAPSEDT01 agent runs this pipeline

stages:
- stage: Build
  displayName: "Prepare Files"
  jobs:
  - job: PrepareArtifacts
    displayName: "Package JSON Files"
    steps:
    - task: CopyFiles@2
      displayName: "Copy JSON Files to Staging"
      inputs:
        SourceFolder: "$(Build.SourcesDirectory)"  # Path where the repo is cloned during pipeline run
        Contents: "**/*.json"  # Copy all JSON files
        TargetFolder: "$(Build.ArtifactStagingDirectory)"  # Temporary folder used to store files before publishing

    - task: PublishBuildArtifacts@1
      displayName: "Publish JSON Files"
      inputs:
        pathToPublish: "$(Build.ArtifactStagingDirectory)"
        artifactName: "json-files"

```

**Deployment Pipeline (azure-pipelines-deploy.yml)**
```yaml
trigger: none  # No trigger, manually runs after build pipeline

pool:
  name: Default  # Use self-hosted agent

variables:
  storageAccount: "yourstorageaccount"
  containerName: "yourcontainer"

stages:
- stage: Deploy
  displayName: "Upload JSON Files to ADLS"
  jobs:
  - job: UploadToADLS
    displayName: "Upload JSON Files"
    steps:
    - download: current
      artifact: json-files

    - task: AzureCLI@2
      displayName: "Authenticate and Upload to ADLS"
      inputs:
        azureSubscription: "Your-Azure-Service-Connection"
        scriptType: "bash"
        scriptLocation: "inlineScript"
        inlineScript: |
          # Login using Managed Identity
          az login --identity

          # Upload JSON files to CLA
          az storage blob upload-batch \
            --account-name $(storageAccount) \
            --destination $(containerName)/CLA/ \
            --source "$(Pipeline.Workspace)/json-files/CLA" \
            --auth-mode "login"

          # Upload JSON files to CoreSignal
          az storage blob upload-batch \
            --account-name $(storageAccount) \
            --destination $(containerName)/CoreSignal/ \
            --source "$(Pipeline.Workspace)/json-files/CoreSignal" \
            --auth-mode "login"

          # Upload JSON files to KFAS
          az storage blob upload-batch \
            --account-name $(storageAccount) \
            --destination $(containerName)/KFAS/ \
            --source "$(Pipeline.Workspace)/json-files/KFAS" \
            --auth-mode "login"

```

=========================================================
**USING cmdline@2**
```yaml
resources:
  pipelines:
    - pipeline: buildOutput
      source: DataConfig-CI  # Name of the build pipeline that publishes json-files artifact
      trigger: none

pool:
  name: Default
  demands:
    - Agent.Name -equals USE-SYNAPSEDT01

variables:
  storageAccount: "usekfdtsynadls01"
  containerName: "config"

stages:
- stage: Deploy
  displayName: "Upload JSON Files to ADLS"
  jobs:
  - job: UploadToADLS
    displayName: "Upload JSON Files"
    steps:
    - download: buildOutput
      artifact: json-files

    - task: CmdLine@2
      displayName: "Upload JSON Files to ADLS using Managed Identity"
      inputs:
        script: |
          az login --identity
          az storage blob upload-batch ^
            --account-name $(storageAccount) ^
            --destination $(containerName)/CapIQ/ ^
            --source "$(Pipeline.Workspace)\json-files\CapIQ" ^
            --auth-mode login
      env:
        AZURE_CONFIG_DIR: $(Agent.TempDirectory)\.azclitask

```
🚀 **This updated guide ensures all permissions, dependencies, and storage paths are correctly set up for successful deployment!** 🚀

