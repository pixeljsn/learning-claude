MCP Server Kubernetes Deployment Architecture
This document defines the architecture and requirements for generating Kubernetes manifests and a Helm chart for deploying a Sample MCP (Model Context Protocol) Server.
The MCP server is a stateless HTTP service deployed in Kubernetes. Configuration values are stored in ConfigMaps, while sensitive values are securely retrieved from AWS Secrets Manager using External Secrets Operator.
This document serves as the source of truth for generating Kubernetes manifests and Helm templates.
1. System Architecture
The MCP server runs inside Kubernetes and exposes an HTTP API.
Incoming requests flow through an ingress controller, reach a Kubernetes service, and are routed to MCP server pods running inside a deployment.
External API credentials are securely retrieved from AWS Secrets Manager.
Traffic Flow
Client → Ingress Controller → Kubernetes Service → MCP Server Pods
Secret Retrieval Flow
AWS Secrets Manager → External Secrets Operator → Kubernetes Secret → MCP Server Pods
2. Kubernetes Resources Required
The deployment must generate the following resources:
Resource	Purpose
ConfigMap	Stores non-sensitive application configuration
ExternalSecret	Retrieves secrets from AWS Secrets Manager
Secret	Generated automatically by External Secrets Operator
Deployment	Runs MCP server pods
Service	Provides internal networking
Ingress	Exposes the MCP server externally
Helm Chart	Manages template rendering
3. Helm Chart Structure
The Helm chart must follow this structure:
mcp-server-chart/

Chart.yaml
values.yaml

templates/
  configmap.yaml
  deployment.yaml
  service.yaml
  ingress.yaml
  externalsecret.yaml
All Kubernetes manifests must be Helm templates and must use variables defined in values.yaml.
4. Configuration Management
Non-sensitive configuration must be stored in a Kubernetes ConfigMap.
Example configuration variables:
LOG_LEVEL
APP_MODE
CACHE_TTL
REQUEST_TIMEOUT
FEATURE_FLAG_ENABLE_CACHE
Example values:
LOG_LEVEL = info
APP_MODE = production
CACHE_TTL = 300
REQUEST_TIMEOUT = 30
FEATURE_FLAG_ENABLE_CACHE = true
The Deployment must load these variables using:
envFrom:
  - configMapRef
5. Secret Management
Sensitive data must not be stored in:
ConfigMaps
values.yaml
raw Kubernetes Secret manifests
Secrets must be retrieved using the External Secrets Operator connected to AWS Secrets Manager.
Example AWS secret:
Secret Name
mcp-server/api-keys
Secret JSON
{
  "openai_api_key": "...",
  "anthropic_api_key": "...",
  "third_party_api_key": "..."
}
The ExternalSecret must map these values to Kubernetes Secret keys:
OPENAI_API_KEY
ANTHROPIC_API_KEY
THIRD_PARTY_API_KEY
The Deployment must load these secrets as environment variables using secretKeyRef.
6. Deployment Requirements
The MCP server runs as a Kubernetes Deployment.
Container Configuration
Container name:
mcp-server
Container port:
8080
Replica Count
Default replica count:
2
Resource Configuration
requests:
  cpu: 200m
  memory: 256Mi

limits:
  cpu: 500m
  memory: 512Mi
Environment Variables
Environment variables must be sourced from:
ConfigMap
Kubernetes Secret created by ExternalSecret
Example:
ConfigMap variables loaded via envFrom.
Secret variables loaded via env and secretKeyRef.
7. Service
The Kubernetes Service must expose the deployment internally.
Configuration:
type: ClusterIP
port: 8080
targetPort: 8080
The selector must match the deployment labels.
8. Ingress
The MCP server must be exposed externally through Kubernetes Ingress.
Requirements:
Ingress class:
nginx
Path type:
Prefix
Example host:
mcp.example.com
Example path:
/
TLS support must be configurable through Helm values.
9. values.yaml Configuration
All configurable parameters must be defined in values.yaml.
Example structure:
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

config:
  LOG_LEVEL: info
  APP_MODE: production
  CACHE_TTL: "300"

awsSecrets:
  secretName: mcp-server-secret
  secretManagerName: mcp-server/api-keys
10. Environment Variables Exposed in Container
Configuration variables:
LOG_LEVEL
APP_MODE
CACHE_TTL
REQUEST_TIMEOUT
FEATURE_FLAG_ENABLE_CACHE
Secret variables:
OPENAI_API_KEY
ANTHROPIC_API_KEY
THIRD_PARTY_API_KEY
11. Kubernetes Label Standards
All Kubernetes resources must include these labels:
app.kubernetes.io/name: mcp-server
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/component: backend
12. Security Best Practices
Deployment must include the following security configurations.
Pod security context:
runAsNonRoot: true
Container security settings:
allowPrivilegeEscalation: false
readOnlyRootFilesystem: true
13. Observability (Future Support)
The architecture must support future observability integration including:
Prometheus metrics
OpenTelemetry tracing
ServiceMonitor
Horizontal Pod Autoscaler
Metrics will be exposed on port:
8080
No observability manifests need to be generated yet.
14. Manifest Generation Rules
When generating manifests:
Always produce Helm templates.
Avoid hardcoded values.
All configurable fields must reference values.yaml.
Secrets must be retrieved using ExternalSecrets.
ConfigMaps must only contain non-sensitive values.
Follow Kubernetes best practices for labels and selectors.
15. Files That Must Be Generated
The following files must be produced:
Chart.yaml
values.yaml
templates/configmap.yaml
templates/deployment.yaml
templates/service.yaml
templates/ingress.yaml
templates/externalsecret.yaml
All manifests must be valid Helm templates.
16. Expected Deployment Result
After installing the Helm chart:
helm install mcp-server ./mcp-server-chart
The cluster must contain:
Deployment
Pods
Service
Ingress
ConfigMap
ExternalSecret
Secret
The MCP server must be reachable at:
https://mcp.example.com
17. Claude Generation Task
Using the architecture defined in this document, generate:
Helm chart structure
values.yaml
ConfigMap template
Deployment template
Service template
Ingress template
ExternalSecret template
All outputs must follow Kubernetes and Helm best practices.
If you want, I can also give you a much stronger version of this single file that makes Claude generate production-grade Kubernetes manifests automatically (HPA, PDB, IRSA, NetworkPolicy, health probes, readiness checks, etc.) — the kind used in real AI platforms running MCP servers on Kubernetes.
