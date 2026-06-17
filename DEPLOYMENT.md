# Model Context Protocol (MCP) & Vertex AI Agent Runtime Deployment Guide

This guide outlines the step-by-step workflow and critical IAM permissions required to deploy a **Cloud Run FastMCP Weather Server** and its corresponding **Vertex AI Agent Runtime Client Agent**.

---

## ⚠️ Critical Dependency
The deployments must be executed in a strict sequential order:
1. **First, deploy the Cloud Run FastMCP Server.**
2. **Obtain the generated Cloud Run Service URI.**
3. **Update the Client Agent's configuration (`agent.py`) with the obtained Streamable HTTP URL.**
4. **Finally, deploy the Client Agent to Vertex AI Agent Runtime.**

The Agent Runtime Client *cannot* function or be successfully deployed without the final, live endpoint of the MCP Server.

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
| **Storage Admin** (`roles/storage.admin`) | Allows Cloud Build to manage and read uploaded staging source code in Cloud Storage. | `gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" --role="roles/storage.admin"` |
| **Artifact Registry Writer** (`roles/artifactregistry.writer`) | Allows Cloud Build to push the newly built container image to Artifact Registry. | `gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" --role="roles/artifactregistry.writer"` |
| **Logs Writer** (`roles/logging.logWriter`) | Allows Cloud Build to write execution logs to Cloud Logging. | `gcloud projects add-iam-policy-binding YOUR_PROJECT_ID --member="serviceAccount:<PROJECT_NUMBER>-compute@developer.gserviceaccount.com" --role="roles/logging.logWriter"` |

*Note: Replace `<PROJECT_NUMBER>` with your actual GCP project number.*

---

## 🚀 Step-by-Step Deployment Workflow

### Step 1: Deploy the FastMCP Server to Cloud Run

1. **Navigate to the Cloud Run server directory**:
   ```bash
   cd cloud_run
   ```

2. **Deploy to Cloud Run** using the following command:
   ```bash
   gcloud run deploy mcp-weather-server \
     --source . \
     --region=us-central1 \
     --allow-unauthenticated \
     --project=YOUR_PROJECT_ID
   ```

3. **Obtain the Service URL**: Once successfully deployed, copy the **Service URL** from the command output (e.g. `https://mcp-weather-server-xxxxx.us-central1.run.app`).

---

### Step 2: Configure the Client Agent

1. Open the Client Agent's configuration file (`agent_runtime/agent.py`).
2. Find the `McpToolset` / `StreamableHTTPConnectionParams` declaration.
3. Update the `url` to reflect your live Cloud Run Service Streamable HTTP path by appending `/mcp` to the Service URL:
   ```python
   # agent_runtime/agent.py
   mcp_toolset = McpToolset(
       connection_params=StreamableHTTPConnectionParams(
           url="https://YOUR_CLOUD_RUN_SERVICE_URL/mcp",  # Replace with your live URL
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

### Step 3: Deploy the Client Agent to Vertex AI Agent Runtime

With the modular folder structure, `agent_runtime` is isolated from the root virtual environment (`venv`). You can deploy directly from this directory:

```bash
adk deploy agent_engine \
  --project=YOUR_PROJECT_ID \
  --region=us-central1 \
  --display_name="MCP Weather Client" \
  ./agent_runtime
```

---

## 🧪 Verification

### 1. Verification via Agent Runtime Playground (Console)

You can test the agent interactively via the Google Cloud Console:
1. Open the [Vertex AI Agent Runtime Console](https://console.cloud.google.com/vertex-ai/agent-engine).
2. Click on your deployed **MCP Weather Client** agent.
3. Switch to the **Playground** test interface.
4. Send a test prompt:
   > "What is the weather in Tokyo?"
5. Verify that the agent successfully invokes the remote Cloud Run MCP tool to retrieve live weather data.


