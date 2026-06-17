---
name: aap-install-portal-ocp
description: "Install the Ansible automation portal (Self-Service Portal) on an existing AAP 2.7 operator deployment on OpenShift Container Platform using a Helm chart. Guides through OAuth application setup in AAP, OCP secret creation, Helm chart deployment, post-install OAuth redirect URI update, and initial RBAC configuration. TRIGGER when: user wants to install the Ansible automation portal or self-service portal on OCP. SKIP: if the user is on a containerized (RPM/VM) installation — the portal only supports OCP. SKIP: if portal is already installed."
---

# AAP Install Ansible Automation Portal (OCP)

This skill installs the Ansible automation portal (self-service portal) on an existing Red Hat Ansible Automation Platform 2.7 operator deployment on OpenShift Container Platform. The portal is built on Red Hat Developer Hub and provides a simplified web interface for business users to launch AAP job templates without needing Ansible expertise.

> **OCP only:** The portal only supports installation via Helm chart on OpenShift Container Platform.

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

Then re-run /aap-install-portal-ocp.
```

Do not proceed until `oc whoami` succeeds.

---

## Preflight Check — Verify AAP is Running

Confirm AAP is deployed and healthy in the `aap` namespace (or ask the user for the namespace):

```bash
oc get pods -n aap | grep -E "gateway|controller" | grep Running
```

If no pods are running, stop and tell the user AAP must be installed and running before the portal can be installed.

---

## Step 1 — Collect Environment Info

Run these commands to gather values needed throughout the install:

```bash
# Get AAP gateway URL
oc get route -n aap -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.host}{"\n"}{end}'

# Get OCP cluster router base domain
oc get ingresses.config cluster -o jsonpath='{.spec.domain}'

# Check Helm chart availability and version
helm repo add openshift-helm-charts https://charts.openshift.io/ 2>/dev/null || true
helm repo update openshift-helm-charts
helm search repo openshift-helm-charts/redhat-rhaap-portal

# Get default imageTagInfo from chart
helm show values openshift-helm-charts/redhat-rhaap-portal --version <chart-version> | grep imageTagInfo
```

Record:
- **AAP gateway URL** (e.g. `https://aap-aap.apps.cluster-xxx.example.com`)
- **clusterRouterBase** (e.g. `apps.cluster-xxx.example.com`)
- **Helm chart version** (e.g. `2.2.1`)
- **imageTagInfo** (e.g. `2.2`)

---

## Step 2 — Configure AAP (Browser Steps)

Tell the user to complete these steps in the AAP UI at the gateway URL collected in Step 1.

### 2a — Verify or create an organization

1. Navigate to **Access Management > Organizations**
2. Note the organization name to use (default is `Default`)
3. If none exists, click **Create organization**, enter a name, click **Finish**

### 2b — Create an OAuth Application

1. Navigate to **Access Management > OAuth Applications**
2. Click **Create OAuth Application** and fill in:
   - **Name:** `ansible-automation-portal` (or any name)
   - **Organization:** choose your organization
   - **Authorization grant type:** `Authorization code`
   - **Client type:** `Confidential`
   - **Redirect URIs:** `https://example.com` (placeholder — updated after deploy)
3. Click **Create OAuth application**
4. **Copy and save the `clientId` and `clientSecret`** from the popup

### 2c — Enable OAuth token creation for external users

1. Navigate to **Settings > Platform gateway**
2. Click **Edit platform gateway settings**
3. Set **Allow external users to create OAuth2 tokens** to **Enabled**
4. Click **Save platform gateway settings**

### 2d — Generate an API token

1. Navigate to **Access Management > API Tokens**
2. Click **Create API Token**
3. Add a description, select your OAuth application, set **Scope** to `Write`
4. Click **Create Token** and **save the token value**

Once the user has `clientId`, `clientSecret`, and `API token`, proceed to Step 3.

---

## Step 3 — Create OCP Project and Secrets

```bash
# Create the portal namespace
oc new-project rhaap-portal

# Create the AAP auth secret
oc create secret generic secrets-rhaap-portal \
  --from-literal=aap-host-url="<AAP_GATEWAY_URL>" \
  --from-literal=oauth-client-id="<CLIENT_ID>" \
  --from-literal=oauth-client-secret="<CLIENT_SECRET>" \
  --from-literal=aap-token="<API_TOKEN>" \
  -n rhaap-portal

# Create the OCI registry pull secret for registry.redhat.io
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > /tmp/auth.json
oc create secret generic rhaap-portal-dynamic-plugins-registry-auth \
  --from-file=auth.json=/tmp/auth.json \
  -n rhaap-portal
rm /tmp/auth.json
```

---

## Step 4 — Deploy the Helm Chart

Create a values file then deploy:

```bash
cat > /tmp/portal-values.yaml << EOF
redhat-developer-hub:
  global:
    clusterRouterBase: <CLUSTER_ROUTER_BASE>
    pluginMode: oci
    imageTagInfo: "<IMAGE_TAG_INFO>"
  upstream:
    backstage:
      appConfig:
        catalog:
          providers:
            rhaap:
              '{{- include "catalog.providers.env" . }}':
                orgs: "<AAP_ORGANIZATION_NAME>"
EOF

helm install rhaap-portal openshift-helm-charts/redhat-rhaap-portal \
  --version <CHART_VERSION> \
  --namespace rhaap-portal \
  -f /tmp/portal-values.yaml
```

> **Important:** The `orgs` key and the template key `'{{- include "catalog.providers.env" . }}'` must be used exactly as shown. Do not add a top-level `schedule` key under the provider — the chart already includes schedules in its defaults and adding one causes a duplicate YAML key error.

Watch the pods come up:

```bash
oc get pods -n rhaap-portal -w
```

The init container (`install-dynamic-plugins`) pulls OCI plugin artifacts from `registry.redhat.io` — this takes 2–3 minutes. Once it completes, the `backstage-backend` container starts. Full readiness (`2/2 Running`) typically takes 5–8 minutes total.

Get the portal route URL:

```bash
oc get route -n rhaap-portal
```

---

## Step 5 — Update OAuth Redirect URI in AAP

Once the pod is `2/2 Running`, update the OAuth application with the real redirect URI.

Determine the redirect URI:
```
<PORTAL_URL>/api/auth/rhaap/handler/frame
```

For example, if the portal URL is `https://rhaap-portal-rhaap-portal.apps.cluster-xxx.example.com`, the redirect URI is:
```
https://rhaap-portal-rhaap-portal.apps.cluster-xxx.example.com/api/auth/rhaap/handler/frame
```

Tell the user to:
1. Go to AAP → **Access Management > OAuth Applications**
2. Click the OAuth application created in Step 2b
3. Replace the `https://example.com` placeholder in **Redirect URIs** with the real redirect URI
4. Click **Save**

---

## Step 6 — Verify Startup

Check backend startup completed without errors:

```bash
POD=$(oc get pods -n rhaap-portal --no-headers | grep -v postgresql | awk '{print $1}' | head -1)
oc logs -n rhaap-portal $POD -c backstage-backend | grep -E "initialized|startup" | grep -v incomingRequest | tail -5
```

Expected: `Plugin initialization complete` with all plugins listed.

If you see `No schedule provided via config for AapResourceEntityProvider`, the catalog provider config was not rendered correctly. Run `helm upgrade` with the corrected values file.

If you see `Map keys must be unique`, there is a duplicate YAML key in the rendered ConfigMap — check that you are using the template key `'{{- include "catalog.providers.env" . }}'` and not adding a separate `schedule:` block at the provider level.

---

## Step 7 — Sign In

Tell the user to navigate to the portal URL and:
1. Click **Sign In**
2. Log in with their **AAP admin credentials** (username: `admin`)

> Regular (non-admin) users will see a 404 or empty page until RBAC is configured in Step 8.

---

## Step 8 — Set Up Initial RBAC

After signing in as admin:

1. Navigate to **Administration > RBAC**
2. Click **Create**, enter a role name (e.g. `portal-users`), click **Next**
3. In **Users and Groups**, select the AAP teams and users to grant access, click **Next**
4. Under **Add permission policies**:
   - Select **Catalog plugin** → enable `catalog.entity.read`
   - Select **Scaffolder plugin** → enable all six permissions:
     - `scaffolder.template.parameter.read`
     - `scaffolder.template.step.read`
     - `scaffolder.action.execute`
     - `scaffolder.task.cancel`
     - `scaffolder.task.create`
     - `scaffolder.task.read`
5. Click **Next** → **Create**

---

## Step 9 — Report Results

Print a summary:

```
Ansible Automation Portal Installed ✅

  Namespace:     rhaap-portal
  Portal URL:    <PORTAL_URL>
  AAP org:       <ORGANIZATION_NAME>
  Helm chart:    redhat-rhaap-portal <CHART_VERSION>
  Plugin mode:   OCI (registry.redhat.io)

Post-install checklist:
  ✅ OCI plugins loaded
  ✅ Backend initialized (all plugins)
  ✅ OAuth redirect URI updated in AAP
  ✅ Initial RBAC role created

Sign in at: <PORTAL_URL>
Use AAP admin credentials for first login. Non-admin users need an RBAC role assigned.

To sync job templates: templates sync automatically every 60 minutes.
Force a sync: Administration > RBAC (wait one sync cycle after login).
```

---

## Troubleshooting

### Init container CrashLoopBackOff — registry auth error
```
unable to retrieve auth token: invalid username/password: unauthorized
```
The `rhaap-portal-dynamic-plugins-registry-auth` secret must have a key named `auth.json` (not `.dockerconfigjson`). Recreate it:
```bash
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > /tmp/auth.json
oc delete secret rhaap-portal-dynamic-plugins-registry-auth -n rhaap-portal
oc create secret generic rhaap-portal-dynamic-plugins-registry-auth \
  --from-file=auth.json=/tmp/auth.json -n rhaap-portal
rm /tmp/auth.json
oc rollout restart deployment/rhaap-portal -n rhaap-portal
```

### Backend crash — No schedule provided
```
No schedule provided via config for AapResourceEntityProvider:default
```
The catalog provider config key must match the chart's environment. Because `_production: true` in the chart defaults, the rendered key is `'production'`. Use `'{{- include "catalog.providers.env" . }}'` as the key in your values file (not `default` or `production` literally).

### Backend crash — Map keys must be unique (YAML parse error)
A `schedule:` block was added at the provider level, creating a duplicate key alongside the chart's built-in schedules. Remove any `schedule:` key from directly under the provider — the chart's defaults already include all required schedules under `sync.orgsUsersTeams` and `sync.jobTemplates`.

### 404 on portal URL
The portal serves a single-page app — initial load requires authentication. Navigate to the URL and click **Sign In**. Log in as an AAP admin user (`admin`). Regular users need an RBAC role (Step 8) before they can access the portal.
