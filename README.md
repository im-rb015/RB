trigger:
  branches:
    include:
      - main
  paths:
    include:
      - client/{{ client_directory_name }}/*

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
  terraformWorkingDirectory: '$(System.DefaultWorkingDirectory)/{{ client_directory_name }}/terraform/{{ environment }}'

stages:
  - stage: Terraform
    displayName: 'Terraform Deployment Stage - {{ environment }}'
    jobs:
      - job: Deploy
        displayName: 'Deploy Infrastructure'
        pool:
          vmImage: '{{ vm_image }}'
        steps:
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: '{{ terraform_version }}'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: '$(terraformWorkingDirectory)'
              backendServiceArm: '{{ service_connection_name }}'
              backendAzureRmResourceGroupName: '{{ resource_group }}'
              backendAzureRmStorageAccountName: '{{ storage_account }}'
              backendAzureRmContainerName: '{{ container_name }}'
              backendAzureRmKey: '{{ backend_key }}'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: '{{ service_connection_name }}'
              commandOptions: '-var-file="environments/{{ environment }}.tfvars"'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Apply'
            inputs:
              provider: 'azurerm'
              command: 'apply'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: '{{ service_connection_name }}'
              commandOptions: '-var-file="environments/{{ environment }}.tfvars"'
