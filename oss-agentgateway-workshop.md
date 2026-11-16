# Open source agentgateway on EKS + MCP Workshop

In this hands-on workshop (about 60 minutes), you'll deploy open source agentgateway to an EKS cluster in Kubernetes mode and put real MCP traffic through it: routing, dynamic discovery, JWT auth, and tool-level RBAC.

**What you'll do:**

| Module | What you'll build | ~Time |
|---|---|---|
| 1 | Install agentgateway on EKS (Kubernetes mode) | 10 min |
| 2 | Deploy a sample MCP server and route to it | 10 min |
| 3 | Connect with MCP Inspector, switch to dynamic backends | 10 min |
| 4 | Secure MCP: JWT auth + tool-level RBAC | 15 min |
| 5 | Explore the built-in UI and metrics | 10 min |

---

## Prerequisites

Set these up before you start:

1. **EKS cluster** running Kubernetes ≥ 1.31, with 2-3 nodes at 2-4 vCPU / 8-16 GiB each. For example:
   ```bash
   eksctl create cluster --name agw-oss-workshop --region us-east-1 --version 1.33 \
     --nodes 2 --node-type m5.large --managed --node-ami-family AmazonLinux2023
   ```
2. CLIs installed: `kubectl` (pointed at the cluster), `helm`, `node`/`npx` (for MCP Inspector).
3. **Kubernetes Gateway API CRDs** installed:
   ```bash
   export GWAPI_VERSION=1.6.3
   kubectl apply --server-side --force-conflicts -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v$GWAPI_VERSION/standard-install.yaml
   ```

No license key. Everything in this workshop is open source.

---

## Background

agentgateway is a Gateway API-native data plane purpose-built for agentic traffic: LLM, **MCP**, and A2A. It sits between everything that makes agentic requests (CLIs, agents, apps) and everything they call (LLMs, MCP servers, APIs):

![agentgateway between clients (CLIs, agents, apps) and destinations (LLMs, MCP servers, APIs)](images/oss/diagram-hub.png)

MCP is becoming the USB port for agent tools, and that creates a new, unmanaged traffic class in your cluster. Who can call which tool? Where are the metrics? Who approved that server?

In Kubernetes mode, agentgateway ships its own control plane: you configure everything with CRDs (`AgentgatewayBackend`, `AgentgatewayPolicy`, `AgentgatewayParameters`) plus the standard Gateway API resources, and every proxy carries a built-in read-only UI. In this workshop you'll install it on EKS, put an MCP server behind it, and progressively lock it down without restarting the proxy.

---

## Module 1: Install agentgateway

### 1.1 CRDs

```bash
export AGW_VERSION=v1.7.0

helm upgrade -i agentgateway-crds oci://cr.agentgateway.dev/charts/agentgateway-crds \
  --create-namespace --namespace agentgateway-system \
  --version $AGW_VERSION
```

### 1.2 Control plane

```bash
helm upgrade -i agentgateway oci://cr.agentgateway.dev/charts/agentgateway \
  --namespace agentgateway-system \
  --version $AGW_VERSION \
  --wait

kubectl get pods -n agentgateway-system
```

### 1.3 Deploy the gateway proxy (EKS: NLB)

The controller creates the proxy Deployment and a LoadBalancer Service for every `Gateway` of class `agentgateway`. The `infrastructure.annotations` land on the generated Service, which is how you ask EKS for an NLB:

```bash
kubectl apply -f- <<'EOF'
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway-proxy
  namespace: agentgateway-system
spec:
  gatewayClassName: agentgateway
  infrastructure:
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
EOF

kubectl rollout status deployment/agentgateway-proxy -n agentgateway-system --timeout=120s
```

### 1.4 Open the built-in UI

Every proxy serves a read-only UI on port 15000:

```bash
kubectl port-forward deployment/agentgateway-proxy -n agentgateway-system 15000
```

Open **http://localhost:15000/ui/** and click around: Gateway Overview, Listeners, Routes, Policies, and a CEL Playground. In Kubernetes mode the UI is read-only; the CRDs you apply are the source of truth, and the UI shows you what the proxy is actually running. It's nearly empty for now. The next modules change that.

> **NLB note:** the NLB takes 2-3 minutes to provision on EKS. Continue with Module 2 while it comes up.

---

## Module 2: Deploy a sample MCP server and route to it

This workshop needs an MCP server to put behind the gateway, so you'll deploy a **sample**: `@modelcontextprotocol/server-everything`, the official MCP reference server. Its toy tools (echo, get-sum, get-env, …) exist purely for testing. It stands in for whatever MCP servers your teams actually run; everything in this module works the same with your own.

![MCP routing through agentgateway: client to Gateway, HTTPRoute, and Backend, then to the MCP server](images/oss/diagram-mcp-routing.png)

### 2.1 Deploy the sample MCP server

Two details matter here. The `appProtocol: agentgateway.dev/mcp` on the Service port tells agentgateway to speak MCP (Streamable HTTP) to this backend, and the `app: mcp-server-everything` label is what dynamic discovery will select on in Module 3.

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
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
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
        group: agentgateway.dev
        kind: AgentgatewayBackend
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

Notice the split: the route is plain Gateway API (`HTTPRoute`), while the MCP smarts live in the `AgentgatewayBackend`. In practice that means platform teams own the gateway and app teams own the servers.

The built-in UI picks the config up right away. The Routes page shows the `mcp` route, and the Gateway Overview counts the new backend:

![Built-in UI Routes page showing the mcp route attached to agentgateway-proxy](images/oss/ui-routes.png)

---

## Module 3: MCP Inspector + dynamic backends

### 3.1 Connect with MCP Inspector

```bash
npx @modelcontextprotocol/inspector@0.21.2
```

In the Inspector:
- **Transport Type:** Streamable HTTP
- **URL:** `http://<GATEWAY_IP>/mcp`
- Click **Connect**, then **Tools → List Tools**

Run **Echo Tool** with `Hello from agentgateway!` and **Get Sum Tool**. Full MCP protocol, through the gateway.

### 3.2 Migrate to a dynamic backend (zero route changes)

Swap the hard-coded host and port for a label selector, and agentgateway discovers matching Services itself:

![Static target compared with dynamic label-selector discovery; the Backend name and HTTPRoute stay the same](images/oss/diagram-dynamic-backend.png)

```bash
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayBackend
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

Run **echo** a few times in the Inspector and watch the logs: every call lands on the same pod. agentgateway pins MCP sessions to a backend pod via a self-describing `mcp-session-id` token. Disconnect and reconnect, and the new session may land on the other replica.

---

## Module 4: Secure MCP with JWT auth + tool-level RBAC

Two policy layers. JWT validation attaches to the Gateway and gates everything; CEL authorization attaches to the Backend, which makes it MCP-aware, so it can see tool names:

![Policy flow: JWT authentication at the Gateway, then CEL authorization at the Backend, with deny and allow outcomes](images/oss/diagram-policy-layers.png)

### 4.1 Require a JWT at the gateway

```bash
kubectl apply -f - <<EOF
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
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

Give the policy a few seconds to program the proxy, then click **Reconnect** in the Inspector and watch it fail:

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
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-rbac
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
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
apiVersion: agentgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: mcp-rbac
  namespace: agentgateway-system
spec:
  targetRefs:
    - group: agentgateway.dev
      kind: AgentgatewayBackend
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

The error is the same one a nonexistent tool gets, so a caller can't probe for hidden tools. You just set per-tool authorization from JWT claims in a single CRD, without touching the app.

---

## Module 5: Explore the built-in UI and metrics

### The UI

The UI isn't a separate install. It's built into every proxy and reads the proxy's live configuration:

![The built-in UI is part of every agentgateway proxy and renders its live config](images/oss/diagram-builtin-ui.png)

With the port-forward from Module 1 still running, open **http://localhost:15000/ui/**:

- **Gateway Overview** summarizes the listeners, routes, and policies the proxy serves
- **Routes** lists the `mcp` route you applied
- **Policies** shows `jwt` and `mcp-rbac`, their targets, and how they're inherited
- **CEL Playground** (under Tools) lets you test expressions like the ones in Module 4 against sample input

![Built-in UI Gateway Overview showing listeners, routes, and policies](images/oss/ui-overview.png)

![Built-in UI Policies page showing the jwt and mcp-rbac policies](images/oss/ui-policies.png)

### Metrics and logs

The proxy exposes Prometheus metrics on port 15020, with MCP-specific counters per method and tool:

```bash
kubectl port-forward deployment/agentgateway-proxy -n agentgateway-system 15020 &
curl -s http://localhost:15020/metrics | grep agentgateway_mcp_requests_total
```

```
agentgateway_mcp_requests_total{method="tools/call",resource_type="tool",server="mcp-server-everything-mcp-http",resource="echo",...} 1
```

Access logs carry the same MCP fields (`mcp.method.name`, `mcp.target`, `mcp.session.id`) plus the JWT subject once the policy is on:

```bash
kubectl logs -n agentgateway-system deployment/agentgateway-proxy --tail 20
```

### Where to go next

- **[Virtual MCP](https://agentgateway.dev/docs/kubernetes/latest/documentation/mcp/virtual.md)**: federate multiple MCP servers behind one endpoint
- **[MCP auth with an IdP](https://agentgateway.dev/docs/kubernetes/latest/documentation/mcp/auth/about.md)**: OAuth protected resource metadata and real identity providers
- **[LLM quickstart](https://agentgateway.dev/docs/kubernetes/latest/documentation/quickstart/llm.md)**: route OpenAI and other providers through the same gateway
- **[Stateful MCP sessions](https://agentgateway.dev/docs/kubernetes/latest/documentation/mcp/session.md)**: how session routing works under the hood
- Need cost management, budgets, virtual keys, tracing dashboards, or multi-cluster? That's [Solo Enterprise agentgateway](https://www.solo.io/products/agentgateway), which builds on these same CRDs.

---

## Cleanup

```bash
kubectl delete agentgatewaypolicy -n agentgateway-system jwt mcp-rbac --ignore-not-found
kubectl delete httproute -n agentgateway-system mcp
kubectl delete agentgatewaybackend -n agentgateway-system mcp-backend
kubectl delete namespace mcp
kubectl delete gateway agentgateway-proxy -n agentgateway-system
helm uninstall agentgateway -n agentgateway-system
helm uninstall agentgateway-crds -n agentgateway-system
kubectl delete namespace agentgateway-system
```
