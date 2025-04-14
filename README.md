
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

def run_pipeline(pipeline_id):
    url = f"{AZURE_DEVOPS_URL}/{PROJECT}/_apis/pipelines/{pipeline_id}/runs?api-version={API_VERSION}"
    response = requests.post(url, headers=HEADERS, json={})
    response.raise_for_status()
    return response.json()

def manage_client_pipeline(client_name, yaml_content, branch="main"):
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
    run_response = run_pipeline(pipeline_id)
    return run_response

# Example usage:
if __name__ == "__main__":
    client = "client-vinod-rahul2"
    yaml_template = """# Pipeline for client-vinod-latest2
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - client/client-vinod-rahul2/*
jobs:
- job: terraform
  steps:
  - script: |
      cd clients/client-vinod/terraform
      terraform init
      terraform apply -auto-approve
    displayName: 'Run Terraform'"""

    try:
        result = manage_client_pipeline(client, yaml_template)
        print(f"✅ Pipeline triggered: {result['url']}")
    except Exception as e:
        print(f"❌ {e}")

