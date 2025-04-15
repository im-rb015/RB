import requests
import base64
import json
import os
# from devops.pipeline_yaml_creation import customize_pipeline_yaml
from jinja2 import Template
import streamlit as st

from devops.pipeline_yaml_creation_new import get_pipeline_template, render_yaml_template, customize_pipeline_yaml

AZURE_ORG = ""
PROJECT = ""
REPO_NAME = "'
AZURE_DEVOPS_URL = f"https://dev.azure.com/{AZURE_ORG}"
API_VERSION = "7.0"
PAT = "'

HEADERS = {
    "Content-Type": "application/json",
    "Authorization": f"Basic {base64.b64encode(f':{PAT}'.encode()).decode()}"
}

def get_repo_id():
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/git/repositories/{REPO_NAME}?api-version={API_VERSION}"
    response = requests.get(url, headers=HEADERS)
    response.raise_for_status()
    return response.json()["id"]

def get_existing_pipelines():
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines?api-version={API_VERSION}"
    response = requests.get(url, headers=HEADERS)
    response.raise_for_status()
    return {pipe['name']: pipe['id'] for pipe in response.json().get('value', [])}

def create_or_update_yaml(repo_id, branch, yaml_path, content):
    # Check if file exists in branch
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/git/repositories/{repo_id}/items?path={yaml_path}&versionDescriptor.version={branch}&includeContentMetadata=true&api-version={API_VERSION}"
    response = requests.get(url, headers=HEADERS)

    is_update = response.status_code == 200
    commit_action = "edit" if is_update else "add"

    commit_url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/git/repositories/{repo_id}/pushes?api-version={API_VERSION}"
    commit_payload = {
        "refUpdates": [
            {"name": f"refs/heads/{branch}", "oldObjectId": get_last_commit_id(repo_id, branch)}
        ],
        "commits": [
            {
                "comment": f"{'Update' if is_update else 'Add'} YAML for pipeline",
                "changes": [
                    {
                        "changeType": commit_action,
                        "item": {"path": f"/{yaml_path}"},
                        "newContent": {"content": content, "contentType": "rawtext"}
                    }
                ]
            }
        ]
    }
    commit_response = requests.post(commit_url, headers=HEADERS, json=commit_payload)
    commit_response.raise_for_status()
    return commit_response.json()

def get_last_commit_id(repo_id, branch):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/git/repositories/{repo_id}/refs?filter=heads/{branch}&api-version={API_VERSION}"
    response = requests.get(url, headers=HEADERS)
    response.raise_for_status()
    refs = response.json().get("value", [])
    if not refs:
        raise Exception("Branch not found or has no commits")
    return refs[0]["objectId"]

def create_pipeline(client_name, yaml_path):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines?api-version={API_VERSION}"
    payload = {
        "name": f"pipeline-{client_name}",
        "configuration": {
            "type": "yaml",
            "path": f"/{yaml_path}",
            "repository": {
                "id": get_repo_id(),
                "name": REPO_NAME,
                "type": "azureReposGit"
            }
        }
    }
    response = requests.post(url, headers=HEADERS, json=payload)
    response.raise_for_status()
    return response.json()["id"]

def run_pipeline(pipeline_id, parameters):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines/{pipeline_id}/runs?api-version={API_VERSION}"
    payload = {
        "resources": {
            "repositories": {
                "self": {
                    "refName": "refs/heads/main"
                }
            }
        },
        "templateParameters": parameters
    }

    response = requests.post(url, headers=HEADERS, json=payload)
    response.raise_for_status()
    return response.json()

def manage_client_pipeline(client_name, yaml_content, parameters, branch="main"):
    yaml_path = f"clients/{client_name}/azure-pipelines.yml"
    repo_id = get_repo_id()

    # Step 1: Add or Update YAML
    create_or_update_yaml(repo_id, branch, yaml_path, yaml_content)

    # Step 2: Check if pipeline exists
    existing = get_existing_pipelines()
    if f"pipeline-{client_name}" in existing:
        pipeline_id = existing[f"pipeline-{client_name}"]
    else:
        pipeline_id = create_pipeline(client_name, yaml_path)

    # Step 3: Trigger pipeline
    run_response = run_pipeline(pipeline_id, parameters)
    return run_response

# Streamlit UI
if __name__ == "__main__":
    st.title("Trigger Azure DevOps Pipeline")

    # User input fields
    terraform_version = st.text_input("Terraform Version", "1.6.6")
    client_directory_name = st.text_input("Client Directory Name", "client-vinod")
    environment = st.selectbox("Environment", ["dev", "prod"])
    service_connection_name = st.text_input("Service Connection Name", "my-service-connection")
    resource_group = st.text_input("Resource Group", "my-resource-group")
    storage_account = st.text_input("Storage Account", "mystorageaccount")
    container_name = st.text_input("Container Name", "mycontainer")
    backend_key = st.text_input("Backend Key", "my-backend-key")
    vm_image = st.selectbox("VM Image", ["ubuntu-latest", "windows-latest"])
    
    if st.button("Run Azure DevOps Pipeline"):
    
        # Parameters to send
        parameters = {
            "terraform_version": terraform_version,
            "client_directory_name": client_directory_name,
            "environment": environment,
            "service_connection_name": service_connection_name,
            "resource_group": resource_group,
            "storage_account": storage_account,
            "container_name": container_name,
            "backend_key": backend_key,
            "vm_image": vm_image
        }
        client = parameters['client_directory_name']
        yaml_template = """
trigger:
  branches:
    include:
      - none


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
  # terraformWorkingDirectory: '$(System.DefaultWorkingDirectory)/CLIENTS/${{ parameters.client_directory_name }}/terraform/${{ parameters.environment }}'
  terraformWorkingDirectory: '$(System.DefaultWorkingDirectory)/CLIENTS/${{ parameters.client_directory_name }}/terraform'

stages:
  - stage: Terraform
    displayName: 'Terraform Deployment Stage - ${{ parameters.environment }}'
    jobs:
      - job: Deploy
        displayName: 'Deploy Infrastructure'
        pool:
          vmImage: ${{ parameters.vm_image }}
        steps:
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: ${{ parameters.terraform_version }}

          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: '$(terraformWorkingDirectory)'
              backendServiceArm: ${{ parameters.service_connection_name }}
              backendAzureRmResourceGroupName: ${{ parameters.resource_group }}
              backendAzureRmStorageAccountName: ${{ parameters.storage_account }}
              backendAzureRmContainerName: ${{ parameters.container_name }}
              backendAzureRmKey: ${{ parameters.backend_key }}

          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: ${{ parameters.service_connection_name }}
              commandOptions: '-var-file="environments/${{ parameters.environment }}.tfvars"'

          - task: TerraformTaskV4@4
            displayName: 'Terraform Apply'
            inputs:
              provider: 'azurerm'
              command: 'apply'
              workingDirectory: '$(terraformWorkingDirectory)'
              environmentServiceNameAzureRM: ${{ parameters.service_connection_name }}
              commandOptions: '-var-file="environments/${{ parameters.environment }}.tfvars"'


        """
        result = manage_client_pipeline(client, yaml_template, parameters)
        print(f"✅ Pipeline triggered: {result['url']}")
  
