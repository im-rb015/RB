can you check how to pass the parameters while running pipelines
 
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - client/client-vinod-three/*
 
# parameters:
#   - name: terraform_version
#     type: string
#     default: '1.6.6'
#   - name: client_directory_name
#     type: string
#   - name: environment
#     type: string
#   - name: service_connection_name
#     type: string
#   - name: resource_group
#     type: string
#   - name: storage_account
#     type: string
#   - name: container_name
#     type: string
#   - name: backend_key
#     type: string
#   - name: vm_image
#     type: string
#     default: 'ubuntu-latest'
 
variables:
  terraformWorkingDirectory: '$(System.DefaultWorkingDirectory)/wewe/terraform/dev'
 
stages:
  - stage: Terraform
    displayName: 'Terraform Deployment Stage - dev'
    jobs:
      - job: Deploy
        displayName: 'Deploy Infrastructure'
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: '1.11.4'
 
          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: '$(terraformWorkingDirectory)'
              backendServiceArm: 'wew'
              backendAzureRmResourceGroupName: 'wewe'
              backendAzureRmStorageAccountName: 'wewe'
              backendAzureRmContainerName: 'wewe'
              backendAzureRmKey: 'terraform.tfstate'
 
          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: 'wew'
              commandOptions: '-var-file="environments/dev.tfvars"'
 
          - task: TerraformTaskV4@4
            displayName: 'Terraform Apply'
            inputs:
              provider: 'azurerm'
              command: 'apply'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: 'wew'
              commandOptions: '-var-file="environments/dev.tfvars"'
 
 
 
 
I am able to do all the things but pipeline is not getting triggered
 
beacuse of these parameters
 
Let me check from my end 
 
def run_pipeline(pipeline_id):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines/{pipeline_id}/runs?api-version={API_VERSION}"
    response = requests.post(url, headers=HEADERS, json={})
    response.raise_for_status()
    return response.json()
 
 
check how can we run this pipeline by passing parameters
 
