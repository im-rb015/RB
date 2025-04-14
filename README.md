  C:\Users\rahul244667\AppData\Local\Programs\Python\Python311\Lib\site-packages\strea
  mlit\runtime\scriptrunner\exec_code.py:121 in exec_func_with_error_handling

  C:\Users\rahul244667\AppData\Local\Programs\Python\Python311\Lib\site-packages\strea
  mlit\runtime\scriptrunner\script_runner.py:640 in code_to_exec

  C:\Users\rahul244667\Downloads\MLOps\Xtera\X-Terra\main.py:122 in <module>

    119 
    120 # Example usage:
    121 if __name__ == "__main__":
  ❱ 122 │   rendered_yaml, parameters = customize_pipeline_yaml()
    123 │   print("rendered_yaml:", rendered_yaml)
    124 │   print("parameters:", parameters)
    125 │   client = parameters['client_directory_name']
────────────────────────────────────────────────────────────────────────────────────────
TypeError: cannot unpack non-iterable NoneType object
rendered_yaml: trigger:
  branches:
    include:
      - main
  paths:
    include:
      - client/RB/*

parameters:
  - name: terraform_version
    type: string
    default: '1.6.6'
  - name: client_directory_name
    type: string
  - name: environment
    type: string
  - name: service_connection_name
    type: string
  - name: resource_group
    type: string
  - name: storage_account
    type: string
  - name: container_name
    type: string
  - name: backend_key
    type: string
  - name: vm_image
    type: string
    default: 'ubuntu-latest'

variables:
  terraformWorkingDirectory: '$(System.DefaultWorkingDirectory)/RB/terraform/dev'

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
              backendServiceArm: 'rbrg'
              backendAzureRmResourceGroupName: 'rb'
              backendAzureRmStorageAccountName: 'rbsa'
              backendAzureRmContainerName: 'rbcontainer'
              backendAzureRmKey: 'terraform.tfstate'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: 'rbrg'
              commandOptions: '-var-file="environments/dev.tfvars"'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Apply'
            inputs:
              provider: 'azurerm'
              command: 'apply'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: 'rbrg'
              commandOptions: '-var-file="environments/dev.tfvars"'
parameters: {'client_directory_name': 'RB', 'environment': 'dev', 'service_connection_name': 'rbrg', 'resource_group': 'rb', 'storage_account': 'rbsa', 'container_name': 'rbcontainer', 'backend_key': 'terraform.tfstate', 'terraform_version': '1.11.4', 'vm_image': 'ubuntu-latest'}
client: RB
❌ 400 Client Error: Bad Request for url: https://dev.azure.com/MLOps-COE-ORG/X-Terra/_apis/pipelines/74/runs?api-version=7.0
