# Claude.md

## Role
You are a senior Platform Engineer and Kubernetes/Helm specialist.

## Primary Objective
Generate a **production-appropriate baseline Helm chart** for deploying a stateless **MCP (Model Context Protocol) Server** on Kubernetes, strictly following the requirements in this document.

The service is an HTTP backend exposed via Ingress. Non-sensitive config is in ConfigMap. Sensitive values are sourced from AWS Secrets Manager via External Secrets Operator.

---

## Required Output
You must generate exactly these files with valid Helm templating:

- `mcp-server-chart/Chart.yaml`
- `mcp-server-chart/values.yaml`
- `mcp-server-chart/templates/configmap.yaml`
- `mcp-server-chart/templates/deployment.yaml`
- `mcp-server-chart/templates/service.yaml`
- `mcp-server-chart/templates/ingress.yaml`
- `mcp-server-chart/templates/externalsecret.yaml`

Return each file as a separate fenced code block, prefixed by a clear header like:

`### mcp-server-chart/templates/deployment.yaml`

Do not omit any file.

---

## Architecture & Traffic Flows

### Request Flow
Client → Ingress Controller → Kubernetes Service → MCP Server Pods

### Secret Flow
AWS Secrets Manager → External Secrets Operator → Kubernetes Secret → MCP Server Pods

---

## Kubernetes Resources to Model
The chart must produce:

1. ConfigMap (non-sensitive app config)
2. ExternalSecret (pulls from AWS Secrets Manager)
3. Deployment (MCP server pods)
4. Service (internal ClusterIP)
5. Ingress (external exposure)

> Note: The Kubernetes Secret is created automatically by External Secrets Operator and should **not** be hand-authored as a raw Secret manifest.

---

## Helm Chart Structure Constraints
Use this exact structure:

```text
mcp-server-chart/
  Chart.yaml
  values.yaml
  templates/
    configmap.yaml
    deployment.yaml
    service.yaml
    ingress.yaml
    externalsecret.yaml
```

All manifests must be Helm templates and avoid hardcoded values wherever configuration is expected.

---

## Labeling Standard (Required on all resources)
Every generated resource must include these labels:

- `app.kubernetes.io/name: mcp-server`
- `app.kubernetes.io/instance: {{ .Release.Name }}`
- `app.kubernetes.io/component: backend`

Selectors must be consistent between Deployment and Service.

---

## values.yaml Requirements
All configurable settings must be declared in `values.yaml`.

At minimum include:

```yaml
replicaCount: 2

image:
  repository: myrepo/mcp-server
  tag: latest
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: nginx
  host: mcp.example.com
  path: /
  tls:
    enabled: false
    secretName: ""

config:
  LOG_LEVEL: info
  APP_MODE: production
  CACHE_TTL: "300"
  REQUEST_TIMEOUT: "30"
  FEATURE_FLAG_ENABLE_CACHE: "true"

awsSecrets:
  targetSecretName: mcp-server-secret
  secretManagerName: mcp-server/api-keys
  refreshInterval: 1h
```

You may add useful defaults, but keep the required keys intact.

---

## ConfigMap Requirements
ConfigMap must contain only non-sensitive variables:

- `LOG_LEVEL`
- `APP_MODE`
- `CACHE_TTL`
- `REQUEST_TIMEOUT`
- `FEATURE_FLAG_ENABLE_CACHE`

Deployment must consume ConfigMap via:

- `envFrom:`
  - `configMapRef`

---

## Secret Management Requirements
Do **not** place secret values in:

- ConfigMap
- `values.yaml` secret literals
- raw Secret YAML

Use `ExternalSecret` to map AWS secret JSON fields from `mcp-server/api-keys` into Kubernetes Secret keys:

- `openai_api_key` → `OPENAI_API_KEY`
- `anthropic_api_key` → `ANTHROPIC_API_KEY`
- `third_party_api_key` → `THIRD_PARTY_API_KEY`

Deployment must expose these via `env` + `secretKeyRef`.

---

## Deployment Requirements

- kind: Deployment
- container name: `mcp-server`
- container port: `8080`
- default replicas: `2`

Resources defaults:

```yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Required security settings:

- Pod security context: `runAsNonRoot: true`
- Container security context:
  - `allowPrivilegeEscalation: false`
  - `readOnlyRootFilesystem: true`

Environment variables in container must include:

From ConfigMap:
- `LOG_LEVEL`
- `APP_MODE`
- `CACHE_TTL`
- `REQUEST_TIMEOUT`
- `FEATURE_FLAG_ENABLE_CACHE`

From Secret:
- `OPENAI_API_KEY`
- `ANTHROPIC_API_KEY`
- `THIRD_PARTY_API_KEY`

---

## Service Requirements

- kind: Service
- type: `ClusterIP` (configurable via values)
- port: `8080`
- targetPort: `8080`
- selector must match pod labels

---

## Ingress Requirements

- Must be configurable and controlled by `ingress.enabled`
- Ingress class: default `nginx`
- host default: `mcp.example.com`
- path default: `/`
- pathType: `Prefix`
- TLS must be configurable via values (`ingress.tls.enabled`, `ingress.tls.secretName`)

---

## ExternalSecret Requirements
Generate `templates/externalsecret.yaml` using `external-secrets.io/v1beta1`.

It must:

- Create/update target Kubernetes Secret name from `awsSecrets.targetSecretName`
- Read remote secret from `awsSecrets.secretManagerName`
- Map the three required properties into the expected env var-style keys
- Include configurable refresh interval

Assume a pre-existing SecretStore/ClusterSecretStore exists; expose its reference in values (e.g., name/kind), and template it.

---

## Non-Goals / Future Extensions (Do Not Generate Yet)
Do not generate manifests for these in this task:

- HPA
- ServiceMonitor
- OpenTelemetry resources
- PDB
- NetworkPolicy

You may leave brief comments in output indicating where these can be added later.

---

## Quality Bar
Output must:

- Be valid YAML and valid Helm templates
- Follow Kubernetes best practices for labels/selectors
- Avoid hardcoded configurable values
- Keep secrets out of plain manifests and values literals
- Be installable with:

```bash
helm install mcp-server ./mcp-server-chart
```

Expected installed resource set:

- Deployment
- Pods
- Service
- Ingress
- ConfigMap
- ExternalSecret
- Kubernetes Secret (created by External Secrets Operator)

Expected endpoint (with DNS/TLS configured):

- `https://mcp.example.com`

---

## Response Format Rules (Strict)
1. Provide a short “Implementation Notes” section (5–10 bullets max).
2. Then provide all required files in order.
3. Do not include pseudo-code.
4. Do not skip Helm expressions.
5. Do not add extra files beyond the required list.
