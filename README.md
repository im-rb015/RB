import requests
import base64
import json
import os
# from devops.pipeline_yaml_creation import customize_pipeline_yaml
from jinja2 import Template
import streamlit as st

from devops.pipeline_yaml_creation_new import get_pipeline_template, render_yaml_template, customize_pipeline_yaml

AZURE_ORG = "MLOps-COE-ORG"
PROJECT = "X-Terra"
REPO_NAME = "X-Terra"
AZURE_DEVOPS_URL = f"https://dev.azure.com/{AZURE_ORG}"
API_VERSION = "7.0"
PAT = "EyMYnXRnPvUr6oRlmxoj7NTwHsARbxY5TXWnSGNdRLehZQ0tDAllJQQJ99BDACAAAAAsfKyNAAASAZDO1ejW"

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

# Example usage:
if __name__ == "__main__":
    rendered_yaml, parameters = customize_pipeline_yaml()
    print("rendered_yaml:", rendered_yaml)
    print("parameters:", parameters)
    client = parameters['client_directory_name']
    print("client:", client)

    st.title("Azure Terraform Deployment UI")
#     rendered_yaml = customize_pipeline_yaml()
#     print("rendered_yaml:", rendered_yaml)
#     print("Type:", type(rendered_yaml))

#     yaml_template = f"""# Pipeline for client-vinod-latestone

# {rendered_yaml}
# """

#     print(" yaml template " ,yaml_template)
    yaml_template = """
    # Pipeline for client-vinodm
    {rendered_yaml}
    """

    try:
        result = manage_client_pipeline(client, yaml_template, parameters)
        print(f"✅ Pipeline triggered: {result['url']}")
    except Exception as e:
        print(f"❌ {e}")

# rendered_yaml = customize_pipeline_yaml()
# if rendered_yaml is not None:
#     print(" redered yaml is", rendered_yaml)










import os
import streamlit as st
from jinja2 import Template
from pathlib import Path


def get_pipeline_template():
    """Read pipeline YAML template from file."""
    try:
        template_path = os.path.join(os.path.dirname(__file__), 'yaml_template', 'iac_pipeline.yml')
        with open(template_path, 'r') as f:
            template_content = f.read()
        # Check if the file is empty
        if not template_content.strip():
            st.error(" Template file is empty.")
            return None
        # print(f"Template file read successfully from {template_path}")
        # Print the content of the template for debugging purposes
        # print("Template content:", template_content)
        return template_content
    except Exception as e:
        st.error(f" Error reading template file: {str(e)}")
        return None

def customize_pipeline_yaml():
    """Replace placeholders in YAML template."""
    try:
        submitted = False
        with st.form("pipeline_yaml_form"):
            st.subheader("Customize YAML Template")
            st.write("Fill in the details below to customize the YAML template.")
            
            client_directory_name = st.text_input("Client Name", placeholder="Client Name")
            environment = st.selectbox("Environment", ["dev", "test", "prod"])
            service_connection_name =st.text_input("Azure Service Connection", placeholder="Azure Service Connection Name")
            resource_group = st.text_input("Resource Group", placeholder="Resource Group Name")
            storage_account = st.text_input("Storage Account", placeholder="Storage Account Name")
            container_name = st.text_input("Container Name", placeholder="Storage Account Container Name")
            backend_key= st.text_input("Backend Key", "terraform.tfstate")
            terraform_version= st.selectbox("Terraform Version", ["1.11.4", "1.10.5", "1.9.8", "1.8.5"])
            vm_image= st.selectbox("VM Image", ["ubuntu-latest", "windows-latest"])
                    
            submit_button  = st.form_submit_button("Generate YAML")
            if submit_button:
                submitted = True
                # st.write("Form submitted!")
        if submitted:
            if not client_directory_name or not environment or not service_connection_name or not resource_group or not storage_account or not container_name or not backend_key or not terraform_version or not vm_image:
                st.error(" Please fill in all fields.")
            else:
                context = {
                    'client_directory_name': client_directory_name,
                    'environment': environment,
                    'service_connection_name': service_connection_name,
                    'resource_group': resource_group,
                    'storage_account': storage_account,
                    'container_name': container_name,
                    'backend_key': backend_key,
                    'terraform_version': terraform_version,
                    'vm_image': vm_image

                }
                rendered_yaml = render_yaml_template(context)
                if rendered_yaml is not None:
                    # st.write(rendered_yaml)
                    return rendered_yaml, context
                        
    except Exception as e:
        st.error(f" Error customizing template: {str(e)}")
    return None

def render_yaml_template(context):
    template_yaml = get_pipeline_template()
    if template_yaml is None:
        return None
    template = Template(template_yaml)
    rendered_yaml = template.render(context)
    return rendered_yaml

# yaml, context = customize_pipeline_yaml()
# print("rendered_yaml:", yaml)
# print("context:", context)
