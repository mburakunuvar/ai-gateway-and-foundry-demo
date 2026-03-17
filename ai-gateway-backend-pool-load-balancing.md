# APIM ❤️ AI Foundry — Backend Pool Load Balancing

**Source lab:** [Azure-Samples/AI-Gateway — backend-pool-load-balancing](https://github.com/Azure-Samples/AI-Gateway/blob/main/labs/backend-pool-load-balancing/backend-pool-load-balancing.ipynb)

---

## Overview

This guide walks through the **Jupyter notebook–based** deployment of Azure API Management (APIM) as an AI Gateway in front of multiple Azure AI Foundry (Azure OpenAI–compatible) endpoints using APIM's **built-in backend pool** feature with **priority-based routing**.

Every step below maps directly to a cell in the source notebook (`backend-pool-load-balancing.ipynb`). Run each code block as a notebook cell in VS Code.

### How it works

- APIM exposes a single inference endpoint to callers.
- A backend pool groups **four** AI Foundry accounts across different regions at **three priority levels**.
- The **Priority 1** backend (East US) receives all traffic under normal conditions.
- When it returns **HTTP 429** (rate-limited), APIM's `retry` policy transparently retries against the pool — the two **Priority 2** backends (Sweden Central + West US) each receive ~50% of the overflowed traffic, balanced by `weight`.
- If all Priority 2 backends are also exhausted, traffic falls to the **Priority 3** backend (UK South) as a last resort.
- Only if **all backends are unavailable** does APIM return **HTTP 503** to the caller.
- No 429 errors are ever surfaced to the caller.

### Architecture

```
Caller (Python SDK / HTTP client)
        │
        ▼
  Azure API Management (Basicv2 tier)
  ┌──────────────────────────────────────────────────────────┐
  │  Inference API  /inference/openai/...                    │
  │  Policy:                                                 │
  │    • set-backend-service → backend pool                  │
  │    • retry on 429 / 503 (count=2, tries all 3 backends)  │
  └───────┬──────────────────────────────────────────────────┘
          │
          ▼
   ┌─────────────────────────────────────────────┐
   │  Backend Pool (priority + weighted routing) │
   │                                             │
   │  ┌─────────────────────────────────────┐   │
   │  │ Priority 1                          │   │  ← served first
   │  │ foundry1 — East US                  │   │
   │  └─────────────────────────────────────┘   │
   │                                             │
   │  ┌──────────────────┐ ┌──────────────────┐ │
   │  │ Priority 2 w=50  │ │ Priority 2 w=50  │ │  ← 50/50 split on P1 failover
   │  │ foundry2         │ │ foundry3         │ │
   │  │ Sweden Central   │ │ West US          │ │
   │  └──────────────────┘ └──────────────────┘ │
   │                                             │
   │  ┌─────────────────────────────────────┐   │
   │  │ Priority 3                          │   │  ← last resort
   │  │ foundry4 — UK South                 │   │
   │  └─────────────────────────────────────┘   │
   └─────────────────────────────────────────────┘
```

---

## Prerequisites

### Tools

| Tool | Version | Install |
|------|---------|---------|
| Python | 3.12+ | [python.org](https://www.python.org/downloads/) |
| VS Code | Latest | [code.visualstudio.com](https://code.visualstudio.com/) |
| VS Code Jupyter extension | Latest | [Marketplace](https://marketplace.visualstudio.com/items?itemName=ms-toolsai.jupyter) |
| Azure CLI | Latest | `curl -sL https://aka.ms/InstallAzureCLIDeb \| sudo bash` |
| Git | Any | `sudo apt install git` |

> **Azure CLI must be signed in** before running the notebook cells. Run `az login` and verify with `az account show` (see Step 4).

### Azure permissions

Your account needs **both** on the target subscription or resource group:

- `Contributor` — to create resources
- `RBAC Administrator` (or `Owner`) — to assign RBAC roles to the APIM managed identity

> **Why?** The APIM system-assigned managed identity must be granted the **Cognitive Services OpenAI User** role on each AI Foundry account so it can obtain Entra ID tokens for authentication (no API keys are used or stored).

### Azure regions with model availability

All four regions must support the model you intend to deploy. Verify availability at:
[Azure AI services — Products by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/?cdn=disable&products=cognitive-services,api-management)

Regions used in this lab: `eastus`, `swedencentral`, `westus`, `uksouth`.

Verify model/version availability per region:
[Azure OpenAI models](https://learn.microsoft.com/azure/ai-services/openai/concepts/models)

---

## Step 1 — Clone the repo, create venv, install dependencies

The lab's `main.bicep` references shared Bicep modules at relative paths and a shared `utils` Python module. You must clone the full repo.

Expected layout (`AI-Gateway` cloned inside the demo repo):
```
/workspaces/
└── ai-gateway-and-foundry-demo/   ← your repo
    ├── AI-Gateway/                ← cloned here
    └── .venv/                     ← virtual environment at repo root
```

> **Note:** `AI-Gateway/` and `.venv/` are both listed in `.gitignore` so they won't be tracked by git.

```bash
# From your demo repo root (e.g. /workspaces/ai-gateway-and-foundry-demo/)
git clone https://github.com/Azure-Samples/AI-Gateway.git
python -m venv .venv
source .venv/bin/activate          # Linux/Mac
# .\.venv\Scripts\Activate.ps1    # Windows PowerShell
pip install -r AI-Gateway/requirements.txt
pip install ipykernel
python -m ipykernel install --user --name ai-gateway-venv --display-name "Python (ai-gateway-venv)"
```

> **Why `ipykernel`?** VS Code's Jupyter extension cannot use a venv as a kernel unless it is explicitly registered. The last command above creates the kernel spec so VS Code can discover it under the name **"Python (ai-gateway-venv)"**.

---

## Step 2 — Open the notebook in VS Code

```bash
# From your demo repo root
code AI-Gateway/labs/backend-pool-load-balancing/backend-pool-load-balancing.ipynb
```

Select the **"Python (ai-gateway-venv)"** kernel when prompted by VS Code. If it does not appear in the list, click **Refresh** in the kernel picker or reload the VS Code window (`Ctrl+Shift+P` → *Developer: Reload Window*).

All subsequent steps are **notebook cells** — run them sequentially inside the notebook.

---

## Step 3 — Cell 0️⃣ Initialize notebook variables

This cell imports the shared `utils` helper module (from `../../shared/`) and defines all configuration variables used by subsequent cells.

```python
import os, sys, json
sys.path.insert(1, '../../shared')  # add the shared directory to the Python path
import utils

deployment_name = os.path.basename(os.path.dirname(globals()['__vsc_ipynb_file__']))
resource_group_name = f"lab-{deployment_name}" # change the name to match your naming style
resource_group_location = "eastus"

aiservices_config = [{"name": "foundry1", "location": "eastus", "priority": 1},
                    {"name": "foundry2", "location": "swedencentral", "priority": 2, "weight": 50},
                    {"name": "foundry3", "location": "westus", "priority": 2, "weight": 50},
                    {"name": "foundry4", "location": "uksouth", "priority": 3}]

models_config = [{"name": "gpt-4o-mini", "publisher": "OpenAI", "version": "2024-07-18", "sku": "GlobalStandard", "capacity": 1}]

apim_sku = 'Basicv2'
apim_subscriptions_config = [{"name": "subscription1", "displayName": "Subscription 1"}]

inference_api_path = "inference"  # path to the inference API in the APIM service
inference_api_type = "AzureOpenAI"  # options: AzureOpenAI, AzureAI, OpenAI, PassThrough
inference_api_version = "2025-03-01-preview"
foundry_project_name = deployment_name

utils.print_ok('Notebook initialized')
```

### Key variable notes

| Variable | Value | Why |
|----------|-------|-----|
| `deployment_name` | Auto-derived from directory name (`backend-pool-load-balancing`) | Keeps naming consistent with the lab folder |
| `resource_group_name` | `lab-backend-pool-load-balancing` | Groups all resources for easy clean-up |
| `aiservices_config` priorities | P1 → P2 (×2, 50/50) → P3 | Priority-based failover with weighted distribution |
| `models_config[0].capacity` | `1` (~1k TPM) | Intentionally low to trigger retry/failover quickly |
| `apim_sku` | `Basicv2` | Fast provisioning; sufficient for this demo |
| `inference_api_type` | `AzureOpenAI` | Exposes the standard Azure OpenAI REST API path under `/openai/...` |

---

## Step 4 — Cell 1️⃣ Verify the Azure CLI and connected subscription

```python
output = utils.run("az account show", "Retrieved az account", "Failed to get the current az account")
if output.success and output.json_data:
    current_user = output.json_data['user']['name']
    tenant_id = output.json_data['tenantId']
    subscription_id = output.json_data['id']

    utils.print_info(f"Current user: {current_user}")
    utils.print_info(f"Tenant ID: {tenant_id}")
    utils.print_info(f"Subscription ID: {subscription_id}")
```

> Run `az login` in a terminal before running this cell. Use `az account set --subscription "<id>"` to switch subscriptions if needed. The login session persists across terminal restarts in Codespaces.

---

## Step 5 — Cell 2️⃣ Create deployment using Bicep

> **Skip this cell if your resources are already deployed.** Proceed directly to Cell 9 (Get deployment outputs) to retrieve the existing deployment's outputs.

This cell:
1. Creates the resource group via `utils.create_resource_group()`.
2. Writes `params.json` programmatically from the notebook variables.
3. Runs `az deployment group create` via the `utils.run()` helper.

```python
# Create the resource group if doesn't exist
utils.create_resource_group(resource_group_name, resource_group_location)

# Define the Bicep parameters
bicep_parameters = {
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "apimSku": { "value": apim_sku },
        "aiServicesConfig": { "value": aiservices_config },
        "modelsConfig": { "value": models_config },
        "apimSubscriptionsConfig": { "value": apim_subscriptions_config },
        "inferenceAPIPath": { "value": inference_api_path },
        "inferenceAPIType": { "value": inference_api_type },
        "foundryProjectName": { "value": foundry_project_name }
    }
}

# Write the parameters to the params.json file
with open('params.json', 'w') as bicep_parameters_file:
    bicep_parameters_file.write(json.dumps(bicep_parameters))

# Run the deployment
output = utils.run(f"az deployment group create --name {deployment_name} --resource-group {resource_group_name} --template-file main.bicep --parameters params.json",
    f"Deployment '{deployment_name}' succeeded", f"Deployment '{deployment_name}' failed")
```

### What gets deployed

The `main.bicep` template provisions (via shared modules at `../../modules/`):

1. **APIM instance** — Basicv2 tier, system-assigned managed identity enabled
2. **Four AI Foundry accounts** across four regions, each with an AI Foundry project and a `gpt-4o-mini` model deployment (capacity: 1k TPM)
3. **RBAC role assignments** — APIM's managed identity gets `Cognitive Services OpenAI User` on all four accounts
4. **APIM inference API** — path `/inference`, with the backend pool, retry policy, and circuit breaker attached

### Shared Bicep modules used by `main.bicep`

| Module | Path | Purpose |
|--------|------|---------|
| `apim.bicep` | `../../modules/apim/v2/apim.bicep` | APIM instance + subscriptions |
| `foundry.bicep` | `../../modules/cognitive-services/v3/foundry.bicep` | AI Foundry accounts + model deployments + RBAC |
| `inference-api.bicep` | `../../modules/apim/v2/inference-api.bicep` | APIM API definition + backend pool + policy |

---

## Step 6 — Cell 3️⃣ Get the deployment outputs

Retrieves the APIM gateway URL and subscription key(s) from the Bicep deployment outputs.

```python
# Obtain all of the outputs from the deployment
output = utils.run(f"az deployment group show --name {deployment_name} -g {resource_group_name}",
    f"Retrieved deployment: {deployment_name}", f"Failed to retrieve deployment: {deployment_name}")

if output.success and output.json_data:
    apim_service_id = utils.get_deployment_output(output, 'apimServiceId', 'APIM Service Id')
    apim_resource_gateway_url = utils.get_deployment_output(output, 'apimResourceGatewayURL', 'APIM API Gateway URL')
    apim_subscriptions = json.loads(utils.get_deployment_output(output, 'apimSubscriptions').replace("\'", "\""))
    for subscription in apim_subscriptions:
        subscription_name = subscription['name']
        subscription_key = subscription['key']
        utils.print_info(f"Subscription Name: {subscription_name}")
        utils.print_info(f"Subscription Key: ****{subscription_key[-4:]}")
    api_key = apim_subscriptions[0].get("key") # default api key to the first subscription key
```

After this cell, the variables `apim_resource_gateway_url` and `api_key` are available in the notebook context for the test cells below.

---

## Step 7 — Cell 🧪 Test the API using direct HTTP calls

Sends 20 requests via the `requests` library and prints the `x-ms-region` response header to show which Azure region served each request.

> **Wait 1–2 minutes after deployment** before running this cell to allow model deployments to become fully available.

> **Tip:** Use the [tracing tool](https://github.com/Azure-Samples/AI-Gateway/blob/main/tools/tracing.ipynb) to track the behavior of the backend pool.

```python
import requests, time

runs = 20
sleep_time_ms = 100
url = f"{apim_resource_gateway_url}/{inference_api_path}/openai/deployments/{models_config[0]['name']}/chat/completions?api-version={inference_api_version}"
messages = {"messages": [
    {"role": "system", "content": "You are a sarcastic, unhelpful assistant."},
    {"role": "user", "content": "Can you tell me the time, please?"}
]}
api_runs = []

# Initialize a session for connection pooling and set any default headers
session = requests.Session()
session.headers.update({'api-key': api_key})

try:
    for i in range(runs):
        print(f"▶️ Run {i+1}/{runs}:")

        start_time = time.time()
        response = session.post(url, json = messages)
        response_time = time.time() - start_time
        print(f"⌚ {response_time:.2f} seconds")

        utils.print_response_code(response)

        if "x-ms-region" in response.headers:
            print(f"x-ms-region: \x1b[1;32m{response.headers.get('x-ms-region')}\x1b[0m")
            api_runs.append((response_time, response.headers.get("x-ms-region")))

        if (response.status_code == 200):
            data = json.loads(response.text)
            print(f"Token usage: {json.dumps(dict(data.get('usage')), indent = 4)}\n")
            print(f"💬 {data.get('choices')[0].get('message').get('content')}\n")
        else:
            print(f"{response.text}\n")

        time.sleep(sleep_time_ms/1000)
finally:
    # Close the session to release the connection
    session.close()
```

### Expected output — normal operation

All requests routed to East US (priority 1) before quota is exhausted:

```
x-ms-region: East US   (20/20 requests)
```

### Expected output — after priority-1 TPM exhaustion

With `capacity: 1` (~1k TPM), East US exhausts quickly. APIM retries transparently and distributes to priority-2 backends:

```
East US: ~3 requests
Sweden Central: ~9 requests
West US: ~8 requests
```

If both priority-2 backends are also exhausted, some requests will show `UK South`. No HTTP 429 errors are visible to the caller.

---

## Step 8 — Cell 🔍 Analyze Load Balancing results

Visualizes the response times colour-coded by region. The priority 1 backend is used until TPM exhaustion, then distribution occurs near-equally across the two priority 2 backends (50/50 weights).

> The first request may take longer than usual and should be discounted in terms of duration.

```python
import pandas as pd
import matplotlib.pyplot as plt
from matplotlib.patches import Rectangle as pltRectangle
import matplotlib as mpl

mpl.rcParams['figure.figsize'] = [15, 7]
df = pd.DataFrame(api_runs, columns = ['Response Time', 'Region'])
df['Run'] = range(1, len(df) + 1)

# Define a color map for each region
color_map = {'East US': 'lightpink', 'Sweden Central': 'lightyellow', 'West US': 'lightblue'}

# Plot the dataframe with colored bars
ax = df.plot(kind = 'bar', x = 'Run', y = 'Response Time',
    color = [color_map.get(region, 'gray') for region in df['Region']], legend = False)

# Add legend
legend_labels = [pltRectangle((0, 0), 1, 1, color = color_map.get(region, 'gray')) for region in df['Region'].unique()]
ax.legend(legend_labels, df['Region'].unique())

plt.title('Load Balancing results')
plt.xlabel('Run #')
plt.ylabel('Response Time')
plt.xticks(rotation = 0)

average = df['Response Time'].mean()
plt.axhline(y = average, color = 'r', linestyle = '--', label = f'Average: {average:.2f}')

plt.show()
```

---

## Step 9 — Cell 🧪 Test the API using the Azure OpenAI Python SDK

Repeats the same test using the Python SDK to ensure compatibility. Note that we use `with_raw_response` to access the `x-ms-region` header.

> **Wait 1–2 minutes** before re-running this cell if you just ran the HTTP test.

```python
import time
from openai import AzureOpenAI

runs = 20
sleep_time_ms = 100

client = AzureOpenAI(
    azure_endpoint = f"{apim_resource_gateway_url}/{inference_api_path}",
    api_key = api_key,
    api_version = inference_api_version
)

for i in range(runs):
    print(f"▶️ Run {i+1}/{runs}:")

    start_time = time.time()
    raw_response = client.chat.completions.with_raw_response.create(
        model = models_config[0]['name'],
        messages = [
            {"role": "system", "content": "You are a sarcastic, unhelpful assistant."},
            {"role": "user", "content": "Can you tell me the time, please?"}
        ])
    response_time = time.time() - start_time

    print(f"⌚ {response_time:.2f} seconds")
    print(f"x-ms-region: \x1b[1;32m{raw_response.headers.get('x-ms-region')}\x1b[0m")

    response = raw_response.parse()

    if response.usage:
        print(f"Token usage:\n   Total tokens: {response.usage.total_tokens}\n   Prompt tokens: {response.usage.prompt_tokens}\n   Completion tokens: {response.usage.completion_tokens}\n")

    print(f"💬 {response.choices[0].message.content}\n")

    time.sleep(sleep_time_ms/1000)
```

---

## Step 10 — Understand the policy

The `policy.xml` file in the lab directory is loaded by `main.bicep` (via `loadTextContent('policy.xml')`) and applied to the inference API. The `{backend-id}` placeholder is replaced at deployment time by the Bicep template with the actual backend pool resource ID.

```xml
<policies>
    <inbound>
        <base />
        <set-backend-service backend-id="{backend-id}" />
    </inbound>
    <backend>
        <!--Set count to one less than the number of backends in the pool to try
            all backends until the backend pool is temporarily unavailable.-->
        <retry count="2" interval="0" first-fast-retry="true"
               condition="@(context.Response.StatusCode == 429 ||
                            context.Response.StatusCode == 503)">
            <!--Switch back to same backend pool which will have automatically
                removed the faulty backend -->
            <set-backend-service backend-id="{backend-id}" />
            <forward-request buffer-request-body="true" />
        </retry>
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
        <choose>
            <!--Return a generic error that does not reveal backend pool details.-->
            <when condition="@(context.Response.StatusCode == 503)">
                <return-response>
                    <set-status code="503" reason="Service Unavailable" />
                </return-response>
            </when>
        </choose>
    </on-error>
</policies>
```

### Policy behaviour breakdown

| Policy element | What it does |
|----------------|-------------|
| `set-backend-service` (inbound) | Directs the request to the named backend pool instead of any individual backend |
| `retry` (backend, count=2) | On HTTP 429 or 503, retries up to twice; APIM picks the next available backend by priority each time |
| `buffer-request-body="true"` | Buffers the request body so it can be replayed to the next backend |
| `on-error choose` | Returns a sanitised 503 — the caller never sees internal backend pool names or URLs |

---

## Step 11 — Cell 🗑️ Clean up resources

When finished with the lab, remove all deployed resources to avoid charges. Use the [clean-up-resources.ipynb](https://github.com/Azure-Samples/AI-Gateway/blob/main/labs/backend-pool-load-balancing/clean-up-resources.ipynb) notebook in the same lab directory.

The clean-up notebook calls `utils.cleanup_resources(deployment_name, resource_group_name)` which:
1. Deletes AI Foundry projects
2. Deletes and purges Cognitive Services accounts (purge is required to free the soft-deleted resource name)
3. Deletes the resource group

---

## Troubleshooting

### Deployment stuck / times out

- Basicv2 APIM typically completes in ~2 minutes. If stuck beyond 10 minutes, check the Azure Portal under **Resource group → Deployments** for live status.
- If the deployment fails at the APIM step, check that your subscription has capacity for the `Basicv2` SKU in your region.

### Model deployment errors

- Verify `gpt-4o-mini` (version `2024-07-18`) is available in **all four regions** (`eastus`, `swedencentral`, `westus`, `uksouth`) at the [availability page](https://learn.microsoft.com/azure/ai-services/openai/concepts/models).
- `GlobalStandard` SKU requires quota in each region. Check per-region quota:
  ```bash
  for loc in eastus swedencentral westus uksouth; do
    echo "=== $loc ==="
    az cognitiveservices usage list --location $loc -o table 2>/dev/null | grep -i gpt
  done
  ```

### HTTP 401 on test calls

- The APIM subscription key is passed as the `api-key` header. Confirm the key hasn't been regenerated.
- Double-check `apim_resource_gateway_url` doesn't end with a trailing slash.

### HTTP 503 on test calls

- All four backends are exhausted simultaneously. With `capacity: 1` (~1k TPM each), rapid bursts can drain the entire pool.
- Increase `sleep_time_ms` (e.g., to `500`) or wait a minute for quota to reset, then re-run the cell.
- Increase `capacity` in `models_config` and re-run the deployment cell.

### Kernel failed to start / "Python Environment no longer available"

This happens when the `.venv` was deleted, recreated, or the kernel spec was never registered.

```bash
# From your demo repo root (e.g. /workspaces/ai-gateway-and-foundry-demo/)
source .venv/bin/activate
pip install ipykernel
python -m ipykernel install --user --name ai-gateway-venv --display-name "Python (ai-gateway-venv)"
```

Then in VS Code: open the notebook, click the kernel selector (top-right), choose **"Python (ai-gateway-venv)"**. If it is still missing, reload the window (`Ctrl+Shift+P` → *Developer: Reload Window*) to force VS Code to re-scan kernel specs.

### x-ms-region header missing

- This header is set by each Azure OpenAI backend. If it's missing, the request may have hit a backend that stripped it. Use the [tracing tool](https://github.com/Azure-Samples/AI-Gateway/blob/main/tools/tracing.ipynb) to inspect the full backend response.

---

## Further reading

- [APIM backend pools documentation](https://learn.microsoft.com/azure/api-management/backends?tabs=bicep)
- [Azure OpenAI models and availability](https://learn.microsoft.com/azure/ai-services/openai/concepts/models)
- [APIM retry policy reference](https://learn.microsoft.com/azure/api-management/retry-policy)
- [APIM managed identity authentication](https://learn.microsoft.com/azure/api-management/authentication-managed-identity-policy)
- [AI Gateway lab catalogue](https://github.com/Azure-Samples/AI-Gateway/tree/main/labs)
