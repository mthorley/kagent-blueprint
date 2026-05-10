# Agent K8s Demo

Basic setup to support declaritive and langchain BYO agents onto kagent.dev for vulnerability triage. Very simple and contrived, meant to show 

## Agents

| Agent | Type | What it does | Tools |
| --- | --- | --- | --- |
| `nvd-agent` | Declarative | Fetches and summarises a CVE from the NVD REST API | `mcp-fetch` |
| `kev-agent` | Declarative | Checks if a CVE is in the CISA KEV catalog | `mcp-fetch` |
| `vulnerability-triage-agent` | Declarative (A2A) | Calls `nvd-agent` and `kev-agent` (parallel or sequential), synthesises a triage urgency | sub-agents |
| `vulnerability-triage-lg-agent` | BYO (LangGraph) | Same orchestration as above, implemented as a Python LangGraph state machine + A2A server | sub-agents |
| `k8s-agent` | Declarative (shipped with kagent) | General Kubernetes troubleshooting | `kagent-tool-server` |

The two triage orchestrators understand a `mode=parallel` / `mode=sequential` parameter (or natural-language equivalents). Sequential mode short-circuits: if NVD reports the CVE doesn't exist, KEV is skipped.

## Layout

```
.
├── setup/                              # how to (re-)render the kagent helm stack
│   └── README.md
├── cluster/                            # everything kubectl applies into the cluster
│   ├── infrastructure/common/kagent/   # kagent platform: CRDs, controller, UI, kagent-tools, kmcp, mcp-fetch
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── kagent-crds-stack.yaml      # rendered helm output (kagent-crds)
│   │   ├── kagent-stack.yaml           # rendered helm output (kagent, only k8s-agent enabled)
│   │   ├── mcp-fetch.yaml              # MCPServer + RemoteMCPServer for generic HTTP fetch
│   │   └── secret.yaml                 # Anthropic API key (Secret/kagent-anthropic) or other to be supplied
│   └── applications/common/agents/     # the four CVE-related agents
│       ├── nvd-agent.yaml              # Declarative — looks up a CVE in NVD via mcp-fetch
│       ├── kev-agent.yaml              # Declarative — checks CISA's Known Exploited Vulnerabilities catalog
│       ├── vulnerability-triage-agent.yaml      # Declarative — A2A orchestrator over nvd + kev
│       └── vulnerability-triage-lg-agent.yaml   # BYO — same orchestration, implemented as LangGraph
├── langchain-agents/
│   └── vulnerability-triage-lg-agent/  # source for the BYO LangGraph agent (built into a local image)
│       ├── main.py
│       ├── requirements.txt
│       ├── Dockerfile
│       └── README.md
├── .github/agents/                     # VSCode/Copilot subagents that mirror the cluster prompts
│   └── nvd.agent.md
└── .gitignore                          # excludes secret.yaml
```

## Apply

```sh
# Anthropic key (one-time, kept out of git)
kubectl create namespace kagent
kubectl -n kagent create secret generic kagent-anthropic \
  --from-literal=ANTHROPIC_API_KEY=sk-ant-...

# Build the BYO LangGraph agent into minikube's docker daemon
eval $(minikube docker-env)
docker build -t vulnerability-triage-lg-agent:0.1.0 \
  langchain-agents/vulnerability-triage-lg-agent/

# Apply everything (CRDs need server-side apply due to size)
kubectl apply -k cluster/infrastructure/common/kagent --server-side
kubectl apply -f cluster/applications/common/agents/
```

To re-render the kagent helm stack, see [setup/README.md](setup/README.md).

## Notes

- **CRD apply** uses `--server-side` because kagent's `agents.kagent.dev` CRD has a schema larger than the 256 KB `last-applied-configuration` annotation limit.
- **kagent helm extras stripped** — only the `k8s-agent` chart subagent is kept enabled; cilium/istio/argo/kgateway/helm/observability/promql are all disabled.
- **mcp-fetch** is two CRDs for one server: an `MCPServer` (kmcp manages the pod + stdio→HTTP gateway) and a `RemoteMCPServer` (how agents reference it). kmcp doesn't auto-create the latter, so it's defined explicitly.
