# agentgateway on EKS + MCP Workshop

In this hands-on workshop (about 60 minutes), you'll deploy Solo Enterprise agentgateway to an EKS cluster and put real MCP traffic through it: routing, dynamic discovery, JWT auth, and tool-level RBAC.

**What you'll do:**

| Module | What you'll build | ~Time |
|---|---|---|
| 1 | Install enterprise agentgateway on EKS | 15 min |
| 2 | Deploy a sample MCP server and route to it | 10 min |
| 3 | Connect with MCP Inspector, switch to dynamic backends | 10 min |
| 4 | Secure MCP: JWT auth + tool-level RBAC | 15 min |
| 5 | Explore your MCP traffic in the UI | 5 min |

---

## Prerequisites

Set these up before you start:

1. **EKS cluster** running a Kubernetes version in the [compatibility matrix](https://docs.solo.io/agentgateway/kubernetes/latest/release-notes/versions/#compatibility-matrix) (≥ 1.31), with 2-3 nodes at 2-4 vCPU / 8-16 GiB each. For example:
   ```bash
   eksctl create cluster --name agw-workshop --region us-east-1 --version 1.33 \
     --nodes 2 --node-type m5.xlarge --managed --node-ami-family AmazonLinux2023
   ```
2. **Solo trial license key** exported:
   ```bash
   export SOLO_TRIAL_LICENSE_KEY=<your-key>
   ```
3. CLIs installed: `kubectl` (pointed at the cluster), `helm`, `node`/`npx` (for MCP Inspector).
4. **Kubernetes Gateway API CRDs** installed:
   ```bash
   kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.0/standard-install.yaml
   ```

---

## Background

agentgateway is a Gateway API-native data plane purpose-built for agentic traffic: LLM, **MCP**, and A2A. It sits between everything that makes agentic requests (CLIs, agents, apps) and everything they call (LLMs, MCP servers, APIs):

![agentgateway between clients (CLIs, agents, apps) and destinations (LLMs, MCP servers, APIs)](images/enterprise/diagram-hub.png)

MCP is becoming the USB port for agent tools, and that creates a new, unmanaged traffic class in your cluster. Who can call which tool? Where are the metrics? Who approved that server?

In this workshop you'll install enterprise agentgateway on EKS, put an MCP server behind it, and then progressively lock it down. Everything happens through Kubernetes CRDs, without restarting the proxy.

---

## Module 1: Install enterprise agentgateway

### 1.1 CRDs

```bash
export ENTERPRISE_AGW_VERSION=v2026.11.0

kubectl create namespace agentgateway-system

helm upgrade -i --create-namespace --namespace agentgateway-system \
    --version $ENTERPRISE_AGW_VERSION enterprise-agentgateway-crds \
    oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway-crds
```

### 1.2 The product: controller, proxy, and UI

One chart installs the whole product. The `ui` block enables the built-in UI, which ships with its own OTel collector and ClickHouse trace store:

```bash
helm upgrade -i -n agentgateway-system enterprise-agentgateway \
  oci://us-docker.pkg.dev/solo-public/enterprise-agentgateway/charts/enterprise-agentgateway \
  --create-namespace \
  --version $ENTERPRISE_AGW_VERSION \
  --set-string licensing.licenseKey=$SOLO_TRIAL_LICENSE_KEY \
  -f - <<EOF
ui:
  enabled: true
  service:
    type: ClusterIP
  features:
    costManagement: true
    costManagementWrites: true
  traces:
    verbose: true
  clickhouse:
    enabled: true
EOF
```

The rollout takes a couple of minutes. You're getting the controller, ext-auth service, rate limiter, WAF server, the UI, and ClickHouse:

```bash
kubectl rollout status deploy/enterprise-agentgateway-ui -n agentgateway-system --timeout=300s
kubectl get pods -n agentgateway-system
```

### 1.3 Deploy the gateway proxy (EKS: NLB)

`EnterpriseAgentgatewayParameters` carries the infra settings. The NLB annotation is what you want on EKS:

```bash
kubectl apply -f- <<'EOF'
---
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayParameters
metadata:
  name: agentgateway-config
  namespace: agentgateway-system
spec:
  logging:
    level: info
  service:
    metadata:
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    spec:
      type: LoadBalancer
  deployment:
    spec:
      replicas: 2
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: enterprise-agentgateway
  infrastructure:
    parametersRef:
      group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayParameters
      name: agentgateway-config
  listeners:
    - name: http
      port: 8080
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
EOF
```

### 1.4 Turn on tracing

Point the proxy's traces at the product's telemetry collector:

```bash
kubectl apply -f- <<'EOF'
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: tracing
  namespace: agentgateway-system
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: agentgateway-proxy
  frontend:
    tracing:
      backendRef:
        name: enterprise-agentgateway-ui-telemetry-collector
        namespace: agentgateway-system
        port: 4317
      protocol: GRPC
      randomSampling: "true"
EOF
```

### 1.5 Open the UI

```bash
kubectl port-forward -n agentgateway-system svc/enterprise-agentgateway-ui 4000:80
```

Open **http://localhost:4000** and click around: Dashboard, Tracing, Cost Management. It's empty for now. The next modules change that.

> **NLB note:** the NLB from step 1.3 takes 2-3 minutes to provision on EKS. Continue with Module 2 while it comes up.

---

## Module 2: Deploy a sample MCP server and route to it

This workshop needs an MCP server to put behind the gateway, so you'll deploy a **sample**: `@modelcontextprotocol/server-everything`, the official MCP reference server. Its toy tools (echo, get-sum, get-env, …) exist purely for testing. It stands in for whatever MCP servers your teams actually run; everything in this module works the same with your own.

![MCP routing through agentgateway: client to Gateway, HTTPRoute, and Backend, then to the MCP server](images/enterprise/diagram-mcp-routing.png)

### 2.1 Deploy the sample MCP server

Two details matter here. The `appProtocol: agentgateway.dev/mcp` on the Service port tells agentgateway to speak MCP to this backend, and the `app: mcp-server-everything` label is what dynamic discovery will select on in Module 3.

```bash
kubectl create namespace mcp --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server-everything
  namespace: mcp
  labels:
    app: mcp-server-everything
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-server-everything
  template:
    metadata:
      labels:
        app: mcp-server-everything
    spec:
      containers:
        - name: mcp-everything
          image: node:20-alpine
          command:
            - sh
            - -c
            - |
              export NODE_OPTIONS="--max-old-space-size=10240 --max-semi-space-size=64"
              npx -y @modelcontextprotocol/server-everything streamableHttp
          ports:
            - name: mcp-http
              containerPort: 3001
          env:
            - name: PORT
              value: "3001"
          readinessProbe:
            tcpSocket:
              port: 3001
            initialDelaySeconds: 15
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-server-everything
  namespace: mcp
  labels:
    app: mcp-server-everything
spec:
  selector:
    app: mcp-server-everything
  ports:
    - name: mcp-http
      port: 8080
      targetPort: 3001
      appProtocol: agentgateway.dev/mcp
EOF

kubectl rollout status deployment/mcp-server-everything -n mcp
```

### 2.2 Static backend + HTTPRoute

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: mcp-target
      static:
        host: mcp-server-everything.mcp.svc.cluster.local
        port: 8080
        protocol: StreamableHTTP
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp
  namespace: agentgateway-system
spec:
  parentRefs:
  - name: agentgateway-proxy
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /mcp
      backendRefs:
      - name: mcp-backend
        group: enterpriseagentgateway.solo.io
        kind: EnterpriseAgentgatewayBackend
      timeouts:
        request: "0s"
EOF
```

### 2.3 Get the gateway address

On EKS the NLB gives you a hostname, not an IP:

```bash
export GATEWAY_IP=$(kubectl get svc -n agentgateway-system --selector=gateway.networking.k8s.io/gateway-name=agentgateway-proxy -o jsonpath='{.items[*].status.loadBalancer.ingress[0].ip}{.items[*].status.loadBalancer.ingress[0].hostname}')
echo $GATEWAY_IP
```

Notice the split: the route is plain Gateway API (`HTTPRoute`), while the MCP smarts live in the `EnterpriseAgentgatewayBackend`. In practice that means platform teams own the gateway and app teams own the servers.

The new config shows up in the UI right away. **Inventory → Gateways** shows `agentgateway-proxy` Active with the `mcp` route and `mcp-backend` destination attached:

![UI Gateways page showing agentgateway-proxy with the mcp route and mcp-backend destination](images/enterprise/ui-gateways.png)

---

## Module 3: MCP Inspector + dynamic backends

### 3.1 Connect with MCP Inspector

```bash
npx @modelcontextprotocol/inspector@0.21.1
```

In the Inspector:
- **Transport Type:** Streamable HTTP
- **URL:** `http://<GATEWAY_IP>:8080/mcp`
- Click **Connect**, then **Tools → List Tools**

Run **Echo Tool** with `Hello from agentgateway!` and **Get Sum Tool**. Full MCP protocol, through the gateway.

Back in the UI Dashboard, the traffic is already visible. The Tool Requests panel counts `echo` and `get-sum` calls per tool:

![UI Dashboard showing request counts and the Tool Requests panel with echo and get-sum bars](images/enterprise/ui-dashboard.png)

### 3.2 Migrate to a dynamic backend (zero route changes)

Swap the hard-coded host and port for a label selector, and agentgateway discovers matching Services itself:

![Static target compared with dynamic label-selector discovery; the Backend name and HTTPRoute stay the same](images/enterprise/diagram-dynamic-backend.png)

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayBackend
metadata:
  name: mcp-backend
  namespace: agentgateway-system
spec:
  mcp:
    targets:
    - name: mcp-target
      selector:
        namespaces:
          matchLabels:
            kubernetes.io/metadata.name: mcp
        services:
          matchLabels:
            app: mcp-server-everything
EOF
```

Reconnect in the Inspector: same URL, same route, still works. The gateway config is now a stable contract. App teams can redeploy, rename, or scale their MCP servers without touching it.

### 3.3 Scale and watch session stickiness (optional)

```bash
kubectl scale deployment mcp-server-everything -n mcp --replicas=2
kubectl logs -n mcp -l app=mcp-server-everything --prefix --follow
```

Run **echo** a few times in the Inspector and watch the logs: every call lands on the same pod. agentgateway pins MCP sessions to a backend pod via a self-describing `mcp-session-id` token (the default, `sessionRouting: Stateful`). Disconnect and reconnect, and the new session may land on the other replica.

---

## Module 4: Secure MCP with JWT auth + tool-level RBAC

Two policy layers. JWT validation attaches to the Gateway and gates everything; CEL authorization attaches to the Backend, which makes it MCP-aware, so it can see tool names:

![Policy flow: JWT authentication at the Gateway, then CEL authorization at the Backend, with deny and allow outcomes](images/enterprise/diagram-policy-layers.png)

### 4.1 Require a JWT at the gateway

```bash
kubectl apply -f - <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: jwt
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway-proxy
  traffic:
    jwtAuthentication:
      mode: Strict
      providers:
        - issuer: solo.io
          jwks:
            inline: |
                {
                  "keys": [
                    {
                      "kty": "RSA",
                      "kid": "solo-public-key-001",
                      "n": "vlmc5pb-jYaOq75Y4r91AC2iuS9B0sm6sxzRm3oOG7nIt2F1hHd4AKll2jd6BZg437qvsLdREnbnVrr8kU0drmJNPHL-xbsTz_cQa95GuKb6AI6osAaUAEL3dPjuoqkGNRe1sAJyOi48qtcbV0kPWcwFmCV0-OiqliCms12jrd1PSI_LYiNc3GcutpxY6BiHkbxxNeIuWDxE-i_Obq8EhhGkwha1KVUvLHV-EwD4M_AY8BegGsX-sjoChXOxyueu_ReqWV227I-FTKwMnjwWW0BQkeI6g1w1WqADmtKZ2sLamwGUJgWt4ZgIyhQ-iQfeN1WN2iupTWa5JAsw--CQJw",
                      "e": "AQAB",
                      "use": "sig",
                      "alg": "RS256"
                    }
                  ]
                }
EOF
```

Give the policy 10-30 seconds to reach both proxy replicas, then click **Reconnect** in the Inspector and watch it fail:

```
MCP error -32001: ... authentication failed
```

### 4.2 Provide a valid JWT

In the Inspector's **Authentication → Custom Headers**:
- **Header Name:** `Authorization`
- **Header Value:** `Bearer <token below>` (the `Bearer ` prefix is required)
- **Toggle the header ON** (easy to miss: headers are disabled by default)

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6InVzZXItaWQiLCJ0ZWFtIjoidGVhbS1pZCIsImV4cCI6MjA3OTU1NjEwNCwibGxtcyI6eyJvcGVuYWkiOlsiZ3B0LTRvIl19fQ.e49g9XE6yrttR9gQAPpT_qcWVKe-bO6A7yJarMDCMCh8PhYs67br00wT6v0Wt8QXMMN09dd8UUEjTunhXqdkF5oeRMXiyVjpTPY4CJeoF1LfKhgebVkJeX8kLhqBYbMXp3cxr2GAmc3gkNfS2XnL2j-bowtVzwNqVI5D8L0heCpYO96xsci37pFP8jz6r5pRNZ597AT5bnYaeu7dHO0a5VGJqiClSyX9lwgVCXaK03zD1EthwPoq34a7MwtGy2mFS_pD1MTnPK86QfW10LCHxtahzGHSQ4jfiL-zp13s8MyDgTkbtanCk_dxURIyynwX54QJC_o5X7ooDc3dxbd8Cw
```

Reconnect, and the tools are back. Paste the token into jwt.io and look at the claims: `org=solo.io`, `sub=user-id`, `team=team-id`. These are what you'll authorize on next. (This token and JWKS are long-lived workshop fixtures signed with a throwaway key. Fine for this lab, never for production.)

### 4.3 CEL-based RBAC on the MCP backend

Attach a policy to the Backend, not the Gateway. That scope makes it MCP-aware. First, deny on purpose:

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: mcp-rbac
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayBackend
      name: mcp-backend
  backend:
    mcp:
      authorization:
        action: Allow
        policy:
          matchExpressions:
            - 'jwt.org == "admin"'
EOF
```

Reconnect, then **List Tools**: the list comes back empty. Unauthorized tools are filtered out of the catalog, not just blocked on call. A denied agent never even learns the tools exist.

### 4.4 Tool-level allowlist

Now allow the `solo.io` org, but only the `echo` tool:

```bash
kubectl apply -f- <<EOF
apiVersion: enterpriseagentgateway.solo.io/v1alpha1
kind: EnterpriseAgentgatewayPolicy
metadata:
  name: mcp-rbac
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: enterpriseagentgateway.solo.io
      kind: EnterpriseAgentgatewayBackend
      name: mcp-backend
  backend:
    mcp:
      authorization:
        action: Allow
        policy:
          matchExpressions:
            - 'jwt.org == "solo.io" && mcp.tool.name == "echo"'
EOF
```

Reconnect, and only **Echo Tool** is listed. Calling `get-sum` directly returns:

```
{"jsonrpc":"2.0","id":2,"error":{"code":-32602,"message":"unknown resource"}}
```

You just set per-tool authorization from JWT claims in a single CRD, without touching the app.

The UI's **Inventory → Policies** page shows all three policies you applied (`jwt`, `mcp-rbac`, `tracing`) as Active:

![UI Policies page listing the jwt, mcp-rbac, and tracing policies as Active](images/enterprise/ui-policies.png)

---

## Module 5: Explore your MCP traffic in the UI

### See what you just did

Observability is part of the product. The proxy streams spans to the built-in telemetry collector, ClickHouse stores them, and the UI renders them:

![Telemetry inside the product: proxy spans flow to the built-in collector, ClickHouse, and the UI](images/enterprise/diagram-observability.png)

With the port-forward from Module 1 still running, open **http://localhost:4000 → Tracing**. Every MCP call you made is there: `initialize`, `tools/list`, and `tools/call` spans with the MCP payload inline.

![UI Tracing page listing MCP spans: initialize, tools/list, and tools/call with payloads](images/enterprise/ui-tracing-list.png)

Click a `tools/call` row to open the trace detail: execution flow, trace tree, and span attributes including `gen_ai.tool: echo`, `mcp.session.id`, the route, and, because of the JWT policy, the caller identity (`jwt.sub: user-id`). That last one is the observability payoff of Module 4: every tool call you make is now attributable to a user.

![UI trace detail for a tools/call span showing execution flow and MCP span attributes including the JWT subject](images/enterprise/ui-trace-detail.png)

Access logs tell the same story:

```bash
kubectl logs -n agentgateway-system -l app.kubernetes.io/name=agentgateway-proxy --prefix --tail 20
```

### Where to go next

- **[Remote MCP](https://github.com/solo-io/fe-enterprise-agentgateway-workshop/blob/main/labs/mcp/remote-mcp.md)**: proxy an external MCP server (Solo docs at `search.solo.io/mcp`) and wire it into Claude Code via `claude mcp add`
- **[MCP Tool Federation](https://github.com/solo-io/fe-enterprise-agentgateway-workshop/blob/main/labs/mcp/mcp-tool-federation.md)**: four MCP servers behind one endpoint with ~34 tools, persona-based filtering, and `failureMode: FailOpen`
- **[OpenAPI → MCP](https://github.com/solo-io/fe-enterprise-agentgateway-workshop/blob/main/labs/mcp/openapi-to-mcp-external-api.md)**: turn any REST API into MCP tools, no code required
- **[MCP eager OAuth](https://github.com/solo-io/fe-enterprise-agentgateway-workshop/blob/main/labs/mcp/mcp-eager-auth-okta.md)** (Auth0/Okta/Entra): IdP-backed MCP auth
- **[MCP guardrails](https://github.com/solo-io/fe-enterprise-agentgateway-workshop/blob/main/labs/mcp/mcp-guardrails.md)** and **[MCP tool rate limiting](https://github.com/solo-io/fe-enterprise-agentgateway-workshop/blob/main/labs/mcp/mcp-tool-rate-limiting.md)**

---

## Cleanup

```bash
kubectl delete enterpriseagentgatewaypolicy -n agentgateway-system jwt mcp-rbac tracing --ignore-not-found
kubectl delete httproute -n agentgateway-system mcp
kubectl delete enterpriseagentgatewaybackend -n agentgateway-system mcp-backend
kubectl delete namespace mcp
kubectl delete gateway agentgateway-proxy -n agentgateway-system
kubectl delete enterpriseagentgatewayparameters agentgateway-config -n agentgateway-system
helm uninstall enterprise-agentgateway -n agentgateway-system
helm uninstall enterprise-agentgateway-crds -n agentgateway-system
kubectl delete namespace agentgateway-system
```

