---
name: aap-deploy-intelligent-assistant-ocp
description: "Deploy the Ansible Lightspeed intelligent assistant (AI chatbot) on an existing AAP 2.7 operator deployment on OpenShift Container Platform. Guides through collecting LLM connection details, creating the chatbot configuration secret, patching the AnsibleAutomationPlatform CR, and verifying the chat bubble appears in the AAP UI. Supports all four LLM provider types: OpenAI-compatible, Red Hat OpenShift AI (rhoai_vllm), Red Hat Enterprise Linux AI (rhelai_vllm), and Microsoft Azure OpenAI. TRIGGER when: user wants to deploy the intelligent assistant, Ansible Lightspeed chatbot, or AI assistant on AAP OCP. SKIP: if the user is on a containerized (RPM/VM) installation — use the containerized deployment path instead. SKIP: if the intelligent assistant is already deployed."
---

# AAP Deploy Intelligent Assistant (OCP Operator)

This skill deploys the Ansible Lightspeed intelligent assistant on an existing Red Hat Ansible Automation Platform 2.7 operator deployment on OpenShift Container Platform. The assistant appears as a chat bubble in the AAP UI and lets users ask questions about their AAP environment.

---

## Preflight Check — Verify OCP Login

Confirm the user is logged into the correct OCP cluster with admin privileges:

```bash
oc whoami && oc cluster-info 2>&1 | head -3
```

If this fails, stop and tell the user:

```
❌ Not logged into an OCP cluster. Please log in first:

  oc login --token=<token> --server=<api-url>

Then re-run /aap-deploy-intelligent-assistant-ocp.
```

Do not proceed until `oc whoami` succeeds.

---

## Preflight Check — Verify AAP is Running

Confirm AAP is deployed and healthy:

```bash
oc get pods -n aap | grep -E "gateway|controller" | grep Running
```

If no pods are running, ask the user for the namespace (it may not be `aap`), then retry. If AAP is not running, stop — the intelligent assistant requires a running AAP instance.

Also confirm the Ansible Lightspeed operator is installed:

```bash
oc get deployment -n aap | grep lightspeed
```

If the `ansible-lightspeed-operator-controller-manager` deployment is not present, stop and tell the user the Ansible Lightspeed operator must be installed from OperatorHub before proceeding.

---

## Step 1 — Collect LLM Connection Details

Ask the user for the following (do not proceed until all three are provided):

1. **LLM provider type** — one of:
   - `openai` — OpenAI-compatible API (most self-hosted and cloud endpoints, including LiteLLM)
   - `rhoai_vllm` — Red Hat OpenShift AI serving vLLM
   - `rhelai_vllm` — Red Hat Enterprise Linux AI serving vLLM
   - `azure_openai` — Microsoft Azure OpenAI

2. **LLM inference URL** — the base API URL (e.g. `https://your-llm-host/v1`).
   - For Azure OpenAI use the base endpoint without a version path (e.g. `https://your-resource.openai.azure.com`)

3. **Model name** — the model name as configured on the LLM endpoint (e.g. `granite-3.3-8b-instruct`, `gpt-4o-mini`, `Granite-Vision-3.2`)

4. **API token / API key** — the authentication token. Tell the user they can type it directly or you can show them the commands with a placeholder to fill in themselves.

For **Azure OpenAI only**, also ask for the **API version** (e.g. `2025-01-01-preview`) — needed for `chatbot_model_config_extras`.

---

## Step 2 — Determine AAP Namespace and CR Name

Run:

```bash
oc get ansibleautomationplatform --all-namespaces
```

Record the **namespace** and **CR name**. These are typically both `aap` on standard deployments, but confirm with the user if multiple results appear.

---

## Step 3 — Create the Chatbot Configuration Secret

Build the secret YAML based on the provider type collected in Step 1. Write it to a file and apply it:

### For openai / rhoai_vllm / rhelai_vllm:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: chatbot-configuration-secret
  namespace: <NAMESPACE>
stringData:
  chatbot_llm_provider_type: <PROVIDER_TYPE>
  chatbot_url: <LLM_URL>
  chatbot_model: <MODEL_NAME>
  chatbot_token: <API_TOKEN>
```

### For azure_openai (includes api_version extra):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: chatbot-configuration-secret
  namespace: <NAMESPACE>
stringData:
  chatbot_llm_provider_type: azure_openai
  chatbot_url: <LLM_URL>
  chatbot_model: <MODEL_NAME>
  chatbot_token: <API_TOKEN>
  chatbot_model_config_extras: '{"api_version": "<API_VERSION>"}'
```

Write the file to the working directory and apply:

```bash
oc apply -f chatbot-configuration-secret.yaml
```

Confirm: `secret/chatbot-configuration-secret created`

---

## Step 4 — Patch the AnsibleAutomationPlatform CR

Patch the AAP CR to enable Lightspeed and reference the secret:

```bash
oc patch ansibleautomationplatform <CR_NAME> -n <NAMESPACE> --type=merge \
  -p '{"spec":{"lightspeed":{"disabled":false,"chatbot_config_secret_name":"chatbot-configuration-secret"}}}'
```

Confirm: `ansibleautomationplatform.aap.ansible.com/<CR_NAME> patched`

---

## Step 5 — Verify the Pods Come Up

The operator will now reconcile and create the Lightspeed pods. This typically takes 2–5 minutes. Check:

```bash
oc get pods -n <NAMESPACE> | grep lightspeed
```

You are looking for these pods in `Running` status:
- `<CR_NAME>-lightspeed-chatbot-api-<hash>`

The `ansible-lightspeed-operator-controller-manager` pod was already running — that is the operator itself, not the chatbot service.

Also verify the AAP CR reconciliation succeeded:

```bash
oc get ansibleautomationplatform <CR_NAME> -n <NAMESPACE> \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

Look for `"type": "Successful"` with `"status": "True"`. If you see `"type": "Failure"` with `"status": "True"`, check the operator logs:

```bash
oc logs -n <NAMESPACE> deployment/ansible-lightspeed-operator-controller-manager --tail=100 \
  | grep -i 'fail\|error\|lightspeed'
```

---

## Step 6 — Verify the Chat Bubble in the AAP UI

Once the `lightspeed-chatbot-api` pod is `Running`, tell the user to:

1. Log into the AAP UI
2. Refresh the browser
3. Look for the **chat bubble icon** in the top-right corner of the toolbar
4. Click it — the **Ansible Lightspeed Intelligent Assistant** panel should open with a welcome message

> If the bubble does not appear immediately after the pod is Running, wait 1–2 minutes and refresh again. The gateway may take a moment to register the new service.

---

## Step 7 — Report Results

Print a summary:

```
Ansible Lightspeed Intelligent Assistant Deployed ✅

  Namespace:        <NAMESPACE>
  AAP CR:           <CR_NAME>
  Secret:           chatbot-configuration-secret
  LLM Provider:     <PROVIDER_TYPE>
  LLM URL:          <LLM_URL>
  Model:            <MODEL_NAME>

Access the assistant:
  → Log into the AAP UI
  → Click the chat bubble icon (top-right toolbar)

The assistant can answer questions about your AAP environment,
help with playbook authoring, and explain automation concepts.
```

---

## Troubleshooting

### Lightspeed chatbot-api pod not appearing after 5+ minutes

Check the AAP CR status for reconciliation failures:

```bash
oc get ansibleautomationplatform <CR_NAME> -n <NAMESPACE> -o yaml | grep -A 30 'status:'
```

If status shows `Failure`, check the full operator log:

```bash
oc logs -n <NAMESPACE> deployment/ansible-lightspeed-operator-controller-manager --tail=200 \
  | grep -E 'FAILED|fatal:|lightspeed' | head -30
```

### Chat bubble not appearing after pod is Running

Verify the secret was created correctly and has all required keys:

```bash
oc get secret chatbot-configuration-secret -n <NAMESPACE> -o jsonpath='{.data}' \
  | python3 -m json.tool
```

All keys should be present (`chatbot_url`, `chatbot_model`, `chatbot_token`, `chatbot_llm_provider_type`). If any are missing, delete the secret, recreate it with the correct keys, then create a new secret with a different name and re-patch the CR with the new secret name.

> **Important:** Do not update an existing chatbot configuration secret in place — the reconciliation logic does not detect secret content changes. Always create a new secret with a new name and patch the CR to reference it.

### Changing the LLM model after deployment

Do not edit the existing `chatbot-configuration-secret`. Instead:

1. Create a new secret with a different name (e.g. `chatbot-configuration-secret-v2`)
2. Patch the CR with the new secret name:
   ```bash
   oc patch ansibleautomationplatform <CR_NAME> -n <NAMESPACE> --type=merge \
     -p '{"spec":{"lightspeed":{"chatbot_config_secret_name":"chatbot-configuration-secret-v2"}}}'
   ```
3. The operator detects the change and redeploys the chatbot pod automatically.

### 401 / authentication errors from LLM

Verify the token in the secret matches what the LLM endpoint expects. Test the endpoint directly:

```bash
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer <TOKEN>" \
  <LLM_URL>/models
```

A `200` response confirms the token and URL are correct.
