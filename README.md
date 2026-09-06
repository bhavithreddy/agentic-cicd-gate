<div align="center">

# 🤖 Agentic CI/CD: Kubernetes Deployment Gate

**An AI agent, exposed over MCP, decides whether your CI/CD pipeline is allowed to deploy.**

[![Kubernetes](https://img.shields.io/badge/Kubernetes-Kind-326CE5?logo=kubernetes&logoColor=white)](https://kind.sigs.k8s.io/)
[![ArgoCD](https://img.shields.io/badge/GitOps-ArgoCD-EF7B4D?logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)
[![OpenTelemetry](https://img.shields.io/badge/Observability-OpenTelemetry-425CC7?logo=opentelemetry&logoColor=white)](https://opentelemetry.io/)
[![Elastic](https://img.shields.io/badge/Elastic-Agent%20Builder%20%2B%20MCP-005571?logo=elasticsearch&logoColor=white)](https://www.elastic.co/)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![Cost](https://img.shields.io/badge/Infra%20Cost-%240-brightgreen)]()

</div>

---

## 💡 Why This Project Matters

Traditional CI/CD pipelines are **blind to runtime cluster state** — they check that
code builds and tests pass, then deploy regardless of whether the target cluster
can actually absorb it safely. This project closes that gap by giving the
pipeline a way to *ask* the cluster **"are you healthy enough for this?"**
before it commits to deploying — using an AI agent as the interface, not a
hardcoded script.

**What makes it more than a demo:**

- 🔒 **It's a real, enforced gate** — the `deploy` job has a hard dependency on
  the health-check job. Verified both ways: a forced pass *and* a forced fail
  correctly allowed / blocked deployment.
- 🧠 **Agent infrastructure used as production tooling, not chat.** The agent's
  reasoning is validated conversationally in Agent Builder, but the pipeline
  itself calls the underlying **Tool** directly over MCP — a deliberate choice
  for deterministic, parseable results instead of relying on LLM prose in a
  CI/CD context.
- 🔗 **A genuinely complete GitOps loop** — every hop (metrics → Elasticsearch →
  ES|QL → Agent Builder Tool → MCP → GitHub Actions → git → ArgoCD → cluster)
  was independently tested before being wired together.
- 🆕 **Built on a current, emerging pattern** — MCP-based agentic tooling inside
  CI/CD pipelines is at the leading edge of DevOps practice in 2026.

> **Resume-ready summary:** Built an agentic CI/CD deployment gate that blocks
> Kubernetes deployments based on live cluster health, using an AI agent
> (Elastic Agent Builder) exposed via Model Context Protocol (MCP) and
> consumed directly from a GitHub Actions pipeline — backed by a full
> OpenTelemetry → Elasticsearch → ES|QL observability pipeline and an
> end-to-end GitOps deploy flow via ArgoCD, all on zero-cost infrastructure.

---

## 🏗️ Architecture

```
 GitHub push
      │
      ▼
 ┌─────────────────────┐   GitHub-hosted runner
 │  build-and-push      │   • build Docker image, tag = commit SHA
 │                       │   • push to Docker Hub
 └─────────┬────────────┘
           │
           ▼
 ┌─────────────────────┐   self-hosted runner
 │  k8s-health-gate      │   • call Agent Builder tool via MCP
 │                       │   • ES|QL query → live OTel metrics
 │                       │   • parse memory_usage_pct vs threshold
 │                       │   • exit 1 → 🚫 pipeline stops here
 └─────────┬────────────┘
           │  ✅ only if healthy
           ▼
 ┌─────────────────────┐   self-hosted runner
 │  deploy               │   • bump image tag in k8s/deployment.yaml
 │                       │   • commit + push to git
 └─────────┬────────────┘
           │
           ▼
     ArgoCD (auto-sync) reconciles the Kind cluster
```

**Data path powering the gate:**

```
Kind cluster
   → OTel Collector (kubeletstats receiver, daemonset)
   → OTLP export
   → Elastic Cloud (Elasticsearch project)
   → ES|QL query
   → Agent Builder Tool
   → exposed via MCP
   → called by GitHub Actions
```

---

## 🧰 Stack

| Layer | Choice | Why |
|---|---|---|
| Kubernetes cluster | **Kind** (local) | Zero cost vs. EKS's continuous control-plane charge |
| GitOps / deploy | **ArgoCD** | Real automated sync from git, not a scripted `kubectl apply` |
| Metrics collection | **OpenTelemetry Operator + Collector** (daemonset, `kubeletstats`) | Standard OTel path into Elastic |
| Observability backend | **Elastic Cloud** (free trial) | Free Agent Builder execution quota |
| Health-check logic | **ES\|QL** query wrapped as an Agent Builder **Tool** | Reusable, independently testable |
| Agent | Agent Builder **Agent** (`kubernetes_analysis_agent`) | Demonstrates the agentic reasoning pattern |
| Exposure | Elastic's built-in **MCP server** | Any MCP client can call the tool — here, GitHub Actions |
| CI/CD | **GitHub Actions** (hybrid: cloud + self-hosted runner) | Self-hosted leg bridges to the local cluster + git push |
| Sample app | Minimal **FastAPI** service (`/`, `/health`) | Deliberately not the focus — the gate is |

---

## 📁 Repo Layout

```
agentic-cicd-gate/
├── app/
│   ├── main.py              # FastAPI app: / and /health
│   └── requirements.txt
├── Dockerfile
├── k8s/
│   └── deployment.yaml      # Deployment + NodePort Service (ArgoCD watches this path)
└── .github/
    └── workflows/
        └── deploy.yaml      # build-and-push → k8s-health-gate → deploy
```

---

## ⚙️ How the Health Gate Works

1. An OTel Collector daemonset scrapes kubelet stats (`k8s.node.cpu.usage`,
   `k8s.node.memory.usage/available/working_set`) every 20s and exports them
   via OTLP to Elastic.
2. An ES|QL query aggregates per-node averages and computes:
   ```
   memory_usage_pct = working_set / (working_set + available) * 100
   ```
3. That query is registered as an Agent Builder **Tool**
   (`k8s_node_health_check`) — independently testable in Kibana.
4. An **Agent** (`kubernetes_analysis_agent`) wraps the tool with a system
   prompt for conversational use — tested manually, both pass and fail
   thresholds verified against real data.
5. For the CI/CD gate itself, GitHub Actions calls the **tool directly** over
   MCP (`tools/call`) rather than the agent's chat, returning clean structured
   JSON so the pass/fail decision is deterministic (`jq` + `bc`, threshold =
   80% memory).
6. A failing check calls `exit 1`, halting the job — the `deploy` job (which
   depends on the gate) never runs.

---

## 🐛 Gotchas Hit During the Build

| Issue | Fix |
|---|---|
| MCP returned `406 Not Acceptable` | Requires explicit `Accept: application/json, text/event-stream` header |
| MCP tool response looked unparseable | It's **double-encoded** JSON — `result.content[0].text` is itself a JSON string; parse it twice |
| No CPU percentage field available | Only raw `k8s.node.cpu.usage` (fractional cores) exists — gate uses memory % as the hard threshold, CPU cores shown as informational context |
| GitHub-hosted runners couldn't reach the local cluster | Registered a **self-hosted runner** on the same machine as the Kind cluster |
| `jq` / `bc` missing on a fresh runner | Installed on the host before the workflow could parse/compare the MCP response |

---

## 🔑 Required GitHub Repo Secrets

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not your account password) |
| `ELASTIC_MCP_ENDPOINT` | `https://<project>.kb.<region>.<provider>.elastic.cloud/api/agent_builder/mcp` |
| `ELASTIC_MCP_API_KEY` | Elastic API key scoped for MCP access |

---

## 🚀 Local Setup

```bash
# 1. Local cluster
kind create cluster --config kind-config.yaml

# 2. ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. OTel Operator + Collector (see otel-rbac.yaml, otel-collector.yaml)
helm install opentelemetry-operator open-telemetry/opentelemetry-operator \
  -n opentelemetry-operator-system --create-namespace
kubectl apply -f otel-rbac.yaml
kubectl apply -f otel-collector.yaml

# 4. ArgoCD Application
kubectl apply -f argocd-app.yaml

# 5. Self-hosted GitHub Actions runner
cd actions-runner && ./run.sh
```

---

## ✅ Result

A push to `main` touching `app/**` or `Dockerfile` triggers:

**build → push to Docker Hub → live cluster health check via an AI agent's
tool over MCP → (if healthy) automatic GitOps deploy through ArgoCD.**

Verified end-to-end, including a deliberate low-threshold test that correctly
**blocked** a deployment.

---

## 🧹 Teardown

```bash
kind delete cluster --name agentic-cicd
```

- Delete the Elastic Cloud project: Kibana project dropdown → **Manage this
  project** → **Delete project**
- Remove the self-hosted runner: repo **Settings → Actions → Runners**
- Revoke the Docker Hub access token
