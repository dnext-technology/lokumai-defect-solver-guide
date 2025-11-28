# Defect Solver Deployment Guide

This guide covers the deployment of the three main components of the Defect Solver system:
1. **Codebase Summarizer** (Knowledge Base Maintenance)
2. **Defect Solver API** (Core Backend)
3. **DNext Coder MCP Server** (Gateway for AI Agents)
4. **Central Storage** (Hugging Face Space)

Refer to the [High Level Diagram](drawio_diagrams/high-level-mcp.png) for a system overview.

---

## 1. Codebase Summarizer

This component runs as a scheduled task on a Hugging Face Space to keep knowledge bases updated.

### Prerequisites
*   **Manual Setup**: Complete the manual steps (cloning, generating project summaries) as described in the [Developer Guide](dev_guide.md) before first deployment.
*   **Config**: Ensure `configurations.yaml` lists all target repositories.
*   **Central Storage**: You must have a Hugging Face Space ready to act as storage (see [Section 4](#4-central-storage)).

### Environment Variables
Set these in your **Hugging Face Space Settings**:

```bash
# LLM Keys
GEMINI_API_KEY=""
OPENAI_API_KEY=""
DEEPSEEK_API_KEY=""
OPENROUTER_API_KEY=""

# Integrations
GITHUB_ACCESS_TOKEN="" # For cloning repos
HF_SPACE_ID=""        # Target Space ID (Central Storage)
HF_ACCESS_TOKEN=""    # Write access token for the Central Storage space

# Scheduler Settings
CODEBASE_SUMMARIZATION_INTERVAL_DAYS=90
MAX_RPM=10
OVERWRITE_EXISTING_FILE_LEVEL_SUMMARIES=True
DEBUG_MODE=True
```

### Deployment
**Do not deploy manually.** This repo uses a GitHub Action (`sync_with_hf.yml`) to sync code to your HF Space.
1. Push changes to `main`.
2. The workflow automatically updates the Space.

---

## 2. Defect Solver API

The backend API that processes bug reports and queries the knowledge base.

### Prerequisites
*   **MongoDB**: Required for logging.
*   **Central Storage**: Read access to the HF Space containing summaries.

### Environment Variables
Configure these in your deployment provider (e.g., Digital Ocean):

```bash
# Security
ADMIN_ACCESS_KEY="lokumai-REPLACE_WITH_SECURE_KEY"

# Central Storage (HF)
HF_SPACE_ID=""
HF_ACCESS_TOKEN=""

# Database
MONGO_URI="mongodb+srv://..."
MONGO_DATABASE=defect_solver
MONGO_COLLECTION=logs

# Models & Logic
FORMATTER_MODEL=gemini-2.5-flash-lite
SPACE_ROUTER_MODEL=gemini-2.5-flash-lite
AGGREGATION_MODEL=gemini-2.5-flash-lite
CONFIG_PATH="configurations.yaml"

# Search Limits
TOP_N_FILES=10
TOP_N_DIRS=3
TOP_N_SPACES=3

# Logic Flags
ADD_COMMON_MODULES_POST_ROUTING=false
USAGE_MAP_PATH=space_router/common-usage-map.yaml
USE_THINKING_IN_BUGFIX=false
DEBUG_MODE=false
```

### Deployment
Uses `deploy.yml` for automated deployment (e.g., to Digital Ocean).
1. Ensure `configurations.yaml` is up to date.
2. Push to `main` to trigger deployment.

---

## 3. DNext Coder MCP Server

The gateway that routes AI Agent requests to the Defect Solver API.

### Environment Variables

```bash
# Backend Connection
DS_API_BASE_URL="https://dnext-coder-api.pia-team.com" # Your API URL
DS_API_MULTIMODULE_ENDPOINT=/mcp_multi_module_bug_localization
DS_API_SINGLEMODULE_ENDPOINT=/mcp_single_module_bug_localization
DS_API_SEARCHSPACE_ENDPOINT=/mcp_search_space_routing
TIMEOUT=120

# Server Config
TRANSPORT_MODE=streamable-http
HOST=0.0.0.0
PORT=8000
LOG_LEVEL=INFO
```

### Deployment
Uses `deploy.yml` for automated deployment.
1. Set environment variables in your cloud provider.
2. Push to `main` to trigger deployment.

---

## 4. Central Storage

The Central Storage is simply a repository in a Hugging Face Space where the Codebase Summarizer pushes the generated knowledge bases.

### Requirements
1.  **Create a Space**: Create a new Space on Hugging Face (e.g., `your-org/ds-storage`).
2.  **Privacy**: Set it to **Private** if your code summaries contain sensitive information.
3.  **Access**: Ensure the `HF_SPACE_ID` used in both the Codebase Summarizer and Defect Solver API environment variables matches this Space's ID.
4.  **Token**: The `HF_ACCESS_TOKEN` used in other components must have write permissions (for Summarizer) and read permissions (for API) to this Space.
