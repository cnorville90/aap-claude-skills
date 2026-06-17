---
name: aap-enable-dashboard-ocp
description: "Enable the Ansible Automation Platform automation dashboard post-installation on an OpenShift operator deployment. Patches the AnsibleAutomationPlatform CR to set FEATURE_DASHBOARD_COLLECTION_ENABLED, verifies the flag propagates to the metrics service ConfigMap, confirms pods restart, and queries the database to validate backfill status. TRIGGER when: user wants to enable the automation dashboard after AAP is already installed on OCP/operator, or asks about post-installation dashboard enablement on OpenShift. SKIP: if the user is on a containerized (RPM/VM) installation — use aap-enable-dashboard-containerized instead. SKIP: if dashboard is already enabled."
---

# AAP Enable Automation Dashboard (Post-Installation, Operator/OCP)

This skill enables the automation dashboard on an existing Red Hat Ansible Automation Platform 2.7 operator deployment on OpenShift. It patches the AnsibleAutomationPlatform CR, verifies flag propagation, confirms pod restarts, and validates backfill status — all with zero platform downtime.

> **Technology Preview:** The automation dashboard is a Tech Preview feature in AAP 2.7. Enabling it post-installation triggers up to 90 days of historical data backfill from the Controller database. Fresh installs with no job history complete backfill immediately with `job_count: 0`.

## Preflight Check — Verify OCP Login

Before anything else, confirm the user is logged into the correct OCP cluster:

```bash
oc whoami && oc cluster-info 2>&1 | head -3
```

If this fails, stop and tell the user:

```
❌ Not logged into an OCP cluster. Please log in first:

  oc login --token=<token> --server=<api-url>

Then re-run /aap-enable-dashboard-ocp.
```

Do not proceed until `oc whoami` succeeds.

## Preflight Check — Verify Operator/OCP Deployment

Confirm this is an operator-managed AAP deployment (not containerized) by checking for the AnsibleAutomationPlatform CRD and the AAP operator:

```bash
# Check for the AAP CRD
oc get crd ansibleautomationplatforms.aap.ansible.com 2>&1

# Check for the AAP operator pod
oc get pods -A -l app.kubernetes.io/managed-by=aap-operator 2>&1 | head -5
```

If the CRD is not found or no operator-managed pods exist, stop and tell the user:

```
❌ This does not appear to be an OpenShift operator deployment of AAP.

  - For containerized (RPM/VM) installations, use /aap-enable-dashboard-containerized instead.
  - If you believe this is an OCP deployment, verify you are logged into the correct cluster
    and that the AAP operator is installed.
```

Do not proceed until both checks pass.

## Step 1 — Collect CR Name and Namespace

Ask the user:
- **AnsibleAutomationPlatform CR name** (commonly `aap`)
- **Namespace** where AAP is deployed (commonly `aap`)

Then confirm the CR exists:

```bash
oc get AnsibleAutomationPlatform <cr-name> -n <namespace> -o jsonpath='{.metadata.name}'
```

If the CR is not found, stop and tell the user to verify the CR name and namespace with:
```bash
oc get AnsibleAutomationPlatform -A
```

## Step 2 — Patch the CR to Enable Dashboard Collection

Post-installation enablement uses the same CR edit as during-installation — there is no separate update playbook or process.

Patch the CR using `oc patch`:

```bash
oc patch AnsibleAutomationPlatform <cr-name> -n <namespace> --type=merge -p '{
  "spec": {
    "feature_flags": {
      "FEATURE_DASHBOARD_COLLECTION_ENABLED": true
    },
    "metrics": {
      "disabled": false,
      "name": "<cr-name>-metrics"
    }
  }
}'
```

Expected output: `ansibleautomationplatform.aap.ansible.com/<cr-name> patched`

The AAP operator and automationmetricsservice operator begin reconciliation automatically. There is no separate install command to re-run.

## Step 3 — Verify Feature Flag Reached Metrics Service ConfigMap

After reconciliation (typically 1–2 minutes), confirm the feature flag appears in the metrics service ConfigMap. Poll until it appears (up to 3 minutes):

```bash
until oc get cm <cr-name>-metrics-env-properties -n <namespace> -o yaml 2>/dev/null | grep -q DASHBOARD; do sleep 5; done && oc get cm <cr-name>-metrics-env-properties -n <namespace> -o yaml | grep -i DASHBOARD
```

Expected output:
```
METRICS_SERVICE_FEATURE_DASHBOARD_COLLECTION_ENABLED: '@bool True'
```

If this does not appear within 3 minutes, check operator reconciliation status:
```bash
oc get AnsibleAutomationPlatform <cr-name> -n <namespace> -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

## Step 4 — Verify Metrics Service Pods Restarted

After the ConfigMap is updated, the operator restarts the metrics service pods. Wait for all three to be Ready:

```bash
oc wait pod -n <namespace> -l app.kubernetes.io/name=<cr-name>-metrics --for=condition=Ready --timeout=180s
```

Expected: Three pods running (`metrics-web`, `metrics-tasks`, `metrics-scheduler`), age matching time since reconciliation.

Confirm with:
```bash
oc get pods -n <namespace> -l app.kubernetes.io/name=<cr-name>-metrics
```

## Step 5 — Monitor Historical Data Backfill

After pods restart, metrics service creates and schedules the `initial_dashboard_collection` task automatically.

Monitor backfill progress via tasks pod logs:

```bash
TASKS_POD=$(oc get pods -n <namespace> \
  -l app.kubernetes.io/name=<cr-name>-metrics \
  --field-selector=status.phase=Running \
  -o jsonpath='{.items[?(@.metadata.name contains "tasks")].metadata.name}')

oc logs -n <namespace> $TASKS_POD | grep -i initial_dashboard
```

Or stream in real time:
```bash
oc logs -n <namespace> -l app.kubernetes.io/name=<cr-name>-metrics \
  -c metrics-tasks -f | grep -i dashboard
```

**Note for fresh installs:** If AAP has no historical automation jobs, the backfill completes immediately with `job_count: 0`. This is expected and successful — not an error.

## Step 6 — Query Database to Validate Backfill Status

Get the postgres pod and query the metrics database:

```bash
DB_POD=$(oc get pods -n <namespace> \
  -l app.kubernetes.io/component=database,app.kubernetes.io/part-of=<cr-name> \
  -o jsonpath='{.items[0].metadata.name}')

oc exec -n <namespace> $DB_POD -- \
  psql -U metricsservice -d metricsservice \
  -c "SELECT COUNT(*), MIN(finished), MAX(finished) FROM dashboard_job_data;"
```

Interpret results:
- `COUNT > 0` — data has been collected; `MAX(finished)` should be close to current date
- `COUNT = 0` with no min/max — fresh install with no jobs yet, or backfill still in progress (check Step 5 logs)

## Step 7 — Report Results

Print a summary in this format:

```
Automation Dashboard Enabled

  Feature flag:  ✅ FEATURE_DASHBOARD_COLLECTION_ENABLED: '@bool True'
  Metrics pods:  ✅ metrics-web, metrics-tasks, metrics-scheduler — Running
  Backfill:      ✅ COUNT=<n> jobs collected  (or: ✅ Fresh install — no historical data, COUNT=0)

Dashboard UI available at: <AAP URL from .status.URL>

Note: Dashboard data collection runs every 6 hours. Data will appear in the
dashboard UI after the first collection cycle completes.
```

The AAP URL can be retrieved with:
```bash
oc get AnsibleAutomationPlatform <cr-name> -n <namespace> -o jsonpath='{.status.URL}'
```
