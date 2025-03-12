# Introduction 
TODO: Give a short introduction of your project. Let this section explain the objectives or the motivation behind this project. 

# Getting Started
TODO: Guide users through getting your code up and running on their own system. In this section you can talk about:
1.	Installation process
2.	Software dependencies
3.	Latest releases
4.	API references

# Build and Test
TODO: Describe and show how to build your code and run the tests. 

# Contribute
TODO: Explain how other users and developers can contribute to make your code better. 

If you want to learn more about creating good readme files then refer the following [guidelines](https://docs.microsoft.com/en-us/azure/devops/repos/git/create-a-readme?view=azure-devops). You can also seek inspiration from the below readme files:
- [ASP.NET Core](https://github.com/aspnet/Home)
- [Visual Studio Code](https://github.com/Microsoft/vscode)
- [Chakra Core](https://github.com/Microsoft/ChakraCore)

- ----------------------------------------------------------------------------------------------------------

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

### **2. Save and Deploy**

- This will fetch the artifact and copy the files to ADLS.
- The **Dev stage** runs automatically.
- The **Prod stage** can be triggered manually or automatically based on conditions.

---

🚀 **This updated guide ensures all permissions, dependencies, and storage paths are correctly set up for successful deployment!** 🚀

