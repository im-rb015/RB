
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

def get_last_commit_id(repo_id, branch):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/git/repositories/{repo_id}/refs?filter=heads/{branch}&api-version={API_VERSION}"
    response = requests.get(url, headers=HEADERS)
    response.raise_for_status()
    refs = response.json().get("value", [])
    if not refs:
        raise Exception("Branch not found or has no commits")
    return refs[0]["objectId"]

def create_or_update_yaml(repo_id, branch, yaml_path, content):
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

def create_pipeline(client_name, yaml_path):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines?api-version={API_VERSION}"
    payload = {
        "name": f"pipeline-{client_name}",
        "configuration": {
            "type": "yaml",
            "path": yaml_path,
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

def run_pipeline(pipeline_id):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines/{pipeline_id}/runs?api-version={API_VERSION}"
    
    # Include parameters in the run request
    payload = {
        "resources": {
            "repositories": {
                "self": {
                    "refName": "refs/heads/main"
                }
            }
        }
    }
    
    response = requests.post(url, headers=HEADERS, json=payload)
    response.raise_for_status()
    return response.json()

def manage_client_pipeline(client_name, yaml_content, branch="main"):
    yaml_path = f"azure-pipelines-{client_name}.yml"
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
    run_response = run_pipeline(pipeline_id)
    return run_response
def main():
    st.title("Azure Pipeline Deployment")

    # Form for input
    with st.form("pipeline_form"):
        client_name = st.text_input("Client Name")
        environment = st.selectbox("Environment", ["dev", "test", "prod"])
        service_connection = st.text_input("Service Connection Name")
        resource_group = st.text_input("Resource Group")
        storage_account = st.text_input("Storage Account")
        container_name = st.text_input("Container Name")
        backend_key = st.text_input("Backend Key", value="terraform.tfstate")
        terraform_version = st.selectbox("Terraform Version", ["1.6.6", "1.5.7", "1.4.6"])
        vm_image = st.selectbox("VM Image", ["ubuntu-latest", "windows-latest"])
        
        submitted = st.form_submit_button("Deploy Pipeline")
        
        if submitted and client_name and environment and service_connection:
            yaml_content = f"""trigger:
  branches:
    include:
      - main

parameters:
- name: terraform_version
  type: string
  default: '{terraform_version}'
- name: client_directory_name
  type: string
  default: '{client_name}'
- name: environment
  type: string
  default: '{environment}'
- name: service_connection_name
  type: string
  default: '{service_connection}'
- name: resource_group
  type: string
  default: '{resource_group}'
- name: storage_account
  type: string
  default: '{storage_account}'
- name: container_name
  type: string
  default: '{container_name}'
- name: backend_key
  type: string
  default: '{backend_key}'
- name: vm_image
  type: string
  default: '{vm_image}'

pool:
  vmImage: $(vm_image)

variables:
  terraformWorkingDirectory: '$(System.DefaultWorkingDirectory)/$(client_directory_name)/terraform/$(environment)'

steps:
- script: |
    echo "Using parameters:"
    echo "Client Directory: $(client_directory_name)"
    echo "Environment: $(environment)"
    echo "Service Connection: $(service_connection_name)"
    echo "Resource Group: $(resource_group)"
    echo "Storage Account: $(storage_account)"
    echo "Container: $(container_name)"
    echo "Backend Key: $(backend_key)"
    echo "Terraform Version: $(terraform_version)"
    echo "VM Image: $(vm_image)"
  displayName: 'Print Parameters'

- task: TerraformInstaller@0
  displayName: 'Install Terraform'
  inputs:
    terraformVersion: $(terraform_version)

- task: TerraformTaskV4@4
  displayName: 'Terraform Init'
  inputs:
    provider: 'azurerm'
    command: 'init'
    workingDirectory: '$(terraformWorkingDirectory)'
    backendServiceArm: $(service_connection_name)
    backendAzureRmResourceGroupName: $(resource_group)
    backendAzureRmStorageAccountName: $(storage_account)
    backendAzureRmContainerName: $(container_name)
    backendAzureRmKey: $(backend_key)

- task: TerraformTaskV4@4
  displayName: 'Terraform Plan'
  inputs:
    provider: 'azurerm'
    command: 'plan'
    workingDirectory: '$(terraformWorkingDirectory)'
    environmentServiceNameAzureRM: $(service_connection_name)
    commandOptions: '-var-file="environments/$(environment).tfvars"'

- task: TerraformTaskV4@4
  displayName: 'Terraform Apply'
  inputs:
    provider: 'azurerm'
    command: 'apply'
    workingDirectory: '$(terraformWorkingDirectory)'
    environmentServiceNameAzureRM: $(service_connection_name)
    commandOptions: '-var-file="environments/$(environment).tfvars"'"""

            try:
                st.code(yaml_content, language='yaml')
                result = manage_client_pipeline(client_name, yaml_content)
                st.success(f"Pipeline created and triggered! URL: {result['url']}")
            except Exception as e:
                st.error(f"Error: {str(e)}")

if __name__ == "__main__":
    main()

