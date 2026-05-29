# Model Context Protocol (MCP) & Vertex AI Agent Engine Deployment Guide

This guide outlines the step-by-step workflow and critical IAM permissions required to deploy a **Cloud Run FastMCP Weather Server** and its corresponding **Vertex AI Agent Engine (Reasoning Engine) Client Agent**.

---

## ⚠️ Critical Dependency
The deployments must be executed in a strict sequential order:
1. **First, deploy the Cloud Run FastMCP Server.**
2. **Obtain the generated Cloud Run Service URI.**
3. **Update the Client Agent's configuration (`agent.py`) with the obtained SSE URL.**
4. **Finally, deploy the Client Agent to Vertex AI Agent Engine.**

The Agent Engine Client *cannot* function or be successfully deployed without the final, live endpoint of the MCP Server.

---

## 🛠️ Prerequisites & API Enablement

Ensure the following Google Cloud APIs are enabled in your project:
- `run.googleapis.com` (Cloud Run)
- `cloudbuild.googleapis.com` (Cloud Build)
- `artifactregistry.googleapis.com` (Artifact Registry)
- `aiplatform.googleapis.com` (Vertex AI / Agent Engine)

You can enable these via the console or using `gcloud`:
```bash
gcloud services enable \
  run.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  aiplatform.googleapis.com \
  --project=YOUR_PROJECT_ID
```

---

## 🔐 Required IAM Permissions

During the deployment process, Cloud Run builds your container from source using Cloud Build. By default, Cloud Build executes using the default Compute Engine service account. In newer Google Cloud projects, this service account has restricted permissions by default, which can cause the build to fail.

You must grant the following IAM roles to the default Compute service account (`<PROJECT_NUMBER>-compute@developer.gserviceaccount.com`):

| Role / Permission | Purpose | Command |
| :--- | :--- | :--- |
| **Storage Object Viewer** (`roles/storage.objectViewer`) | Allows Cloud Build to read the uploaded staging source code from Cloud Storage. | `gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" --role="roles/storage.objectViewer"` |
| **Artifact Registry Writer** (`roles/artifactregistry.writer`) | Allows Cloud Build to push the newly built container image to Artifact Registry. | `gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" --role="roles/artifactregistry.writer"` |
| **Logs Writer** (`roles/logging.logWriter`) | Allows Cloud Build to write execution logs to Cloud Logging. | `gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" --role="roles/logging.logWriter"` |

*Note: Replace `<PROJECT_NUMBER>` with your actual GCP project number.*

---

## 🚀 Step-by-Step Deployment Workflow

### Step 1: Deploy the FastMCP Server to Cloud Run

1. **Clone the FastMCP Server repository** to download the server code locally:
   ```bash
   git clone https://github.com/demichael4520/cr_mcp_weather
   ```

2. **Navigate to the cloned FastMCP Server directory** (referred to as `path/to/cr_mcp_weather` throughout this guide):
   ```bash
   cd cr_mcp_weather
   ```

3. **Deploy to Cloud Run** using the following command:
   ```bash
   gcloud run deploy mcp-weather-server \
     --source . \
     --region=us-central1 \
     --allow-unauthenticated \
     --project=YOUR_PROJECT_ID
   ```

4. **Obtain the Service URL**: Once successfully deployed, copy the **Service URL** from the command output (e.g. `https://mcp-weather-server-xxxxx.us-central1.run.app`).

---

### Step 2: Configure the Client Agent

1. Open the Client Agent's configuration file (`agent.py`).
2. Find the `McpToolset` / `SseConnectionParams` declaration.
3. Update the `url` to reflect your live Cloud Run Service SSE path by appending `/sse` to the Service URL:
   ```python
   # agent.py
   mcp_toolset = McpToolset(
       connection_params=SseConnectionParams(
           url="https://YOUR_CLOUD_RUN_SERVICE_URL/sse",  # Replace with your live URL
       )
   )
   ```
4. Ensure the model is configured to use the latest Gemini version:
   ```python
   root_agent = LlmAgent(
       model='gemini-2.5-flash',
       name='mcp_weather_client',
       # ...
   )
   ```

---

### Step 3: Deploy the Client Agent to Vertex AI Agent Engine

To prevent deployment payload size errors caused by local virtual environments (`venv`), package only the required files into a temporary directory:

1. Create a clean staging directory:
   ```bash
   mkdir -p /tmp/weather_deploy
   ```
2. Copy the necessary agent files and enable Agent Identity:
   ```bash
   cp agent.py requirements.txt __init__.py /tmp/weather_deploy/
   echo '{ "identity_type": "AGENT_IDENTITY" }' > /tmp/weather_deploy/.agent_engine_config.json
   ```
3. Deploy the agent from the staging directory using the ADK CLI:
   ```bash
   adk deploy agent_engine \
     --project=YOUR_PROJECT_ID \
     --region=us-central1 \
     --display_name="MCP Weather Client" \
     /tmp/weather_deploy
   ```
4. Clean up the staging directory:
   ```bash
   rm -rf /tmp/weather_deploy
   ```

---

## 🧪 Verification

To verify that the Reasoning Engine agent is successfully routed to your Cloud Run MCP Server, query the deployed Reasoning Engine. 

Because newer ADK runtimes may expose asynchronous methods that are incompatible with standard synchronous SDK invocation bindings, query the agent using the **low-level streaming API client**:

```python
import vertexai
from vertexai.preview import reasoning_engines
from google.cloud.aiplatform_v1beta1 import types as aip_types
from vertexai.reasoning_engines import _utils

vertexai.init(project="YOUR_PROJECT_ID", location="us-central1")

reasoning_engine = reasoning_engines.ReasoningEngine("YOUR_REASONING_ENGINE_ID")
execution_client = reasoning_engine.execution_api_client

request = aip_types.StreamQueryReasoningEngineRequest(
    name=reasoning_engine.resource_name,
    input={"message": "What is the weather in Tokyo?", "user_id": "test_user"},
    class_method="stream_query"
)

response_stream = execution_client.stream_query_reasoning_engine(request=request)

for chunk in response_stream:
    for parsed in _utils.yield_parsed_json(chunk):
        if parsed and "output" in parsed:
            print(parsed["output"], end="")
```
