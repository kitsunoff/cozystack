# CozyIDP Market Analysis

## Executive Summary

This document analyzes existing solutions that abstract developers from infrastructure complexity, providing a foundation for CozyIDP design decisions.

## Categories of Solutions

### Category 1: PaaS (Heroku-style)

Managed platforms that provide simple deployment workflows with minimal infrastructure exposure.

| Solution | Model | Config Format | Self-hosted | Notes |
|----------|-------|---------------|-------------|-------|
| **Heroku** | Managed | Procfile | No | Original PaaS, expensive at scale |
| **Railway** | Managed | Procfile/Dockerfile | No | Modern UX, usage-based pricing |
| **Render** | Managed | UI + YAML | No | Simple, but expensive at scale |
| **Fly.io** | Managed | fly.toml | No | Edge deployment, lightweight VMs |
| **Qovery** | BYOC | qovery.yml | Partial (engine OSS) | Runs in your AWS/GCP/Azure |
| **Northflank** | BYOC | UI + API | No | Enterprise focus |

**Key characteristics:**
- Git push → deploy workflow
- Managed databases and services
- Simple pricing (per dyno/instance or usage-based)
- Limited customization

---

### Category 2: Platform Orchestrators / IDP Frameworks

Tools for building internal developer platforms with self-service capabilities.

| Solution | Approach | Config Format | Self-hosted | Notes |
|----------|----------|---------------|-------------|-------|
| **Humanitec** | Platform Orchestrator | Score | No | Dynamic resource provisioning |
| **Backstage** | Developer Portal | Templates | Yes (CNCF) | Catalog + Scaffolding, no deploy |
| **Port** | Developer Portal | Blueprints | No | Backstage alternative, SaaS |
| **Kratix** | Platform Framework | Promises | Yes (OSS) | Kubernetes-native, composable |

**Key characteristics:**
- Focus on developer self-service
- Separation of platform team and developer concerns
- Extensible through plugins/promises
- Often paired with other tools for full workflow

---

### Category 3: Application Models (Kubernetes-native)

Declarative application specifications designed for cloud-native environments.

| Solution | Model | Config Format | Self-hosted | Notes |
|----------|-------|---------------|-------------|-------|
| **KubeVela** | OAM | Application CRD | Yes (CNCF) | Components + Traits + Policies |
| **Score** | Workload Spec | score.yaml | Yes (OSS) | Platform-agnostic, multiple runtimes |
| **Waypoint** | Workflow | waypoint.hcl | Yes (HashiCorp) | Build → Deploy → Release |
| **Acorn** | App Packaging | Acornfile | Yes (OSS) | Simplified K8s deployment |

**Key characteristics:**
- Kubernetes-native or Kubernetes-compatible
- Declarative application definitions
- Separation of concerns (dev vs. platform)
- Extensible resource types

---

## Configuration Format Comparison

### Heroku Procfile

The original simple format.

```yaml
web: gunicorn app:app
worker: celery -A tasks worker
```

| Pros | Cons |
|------|------|
| Extremely simple | No dependency declarations |
| Well-known standard | No resource configuration |
| | No environment management |

---

### Score Specification

Platform-agnostic workload specification by Humanitec.

```yaml
apiVersion: score.dev/v1b1
metadata:
  name: my-app
containers:
  main:
    image: myapp:latest
    variables:
      DATABASE_URL: "postgresql://${resources.db.host}:${resources.db.port}/${resources.db.database}"
resources:
  db:
    type: postgres
  cache:
    type: redis
```

| Pros | Cons |
|------|------|
| Declarative dependencies | No workflow/environments built-in |
| Placeholder substitution | Requires Score implementation per platform |
| Platform-agnostic | Limited ecosystem maturity |
| Multiple runtime targets | |

**Implementations:**
- `score-compose` - Docker Compose
- `score-k8s` - Kubernetes manifests
- `score-humanitec` - Humanitec Platform Orchestrator

---

### KubeVela OAM (Open Application Model)

Full-featured application model with components, traits, policies, and workflows.

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: my-app
spec:
  components:
    - name: web
      type: webservice
      properties:
        image: myapp:latest
        port: 8080
      traits:
        - type: scaler
          properties:
            replicas: 3
        - type: gateway
          properties:
            domain: myapp.example.com
  policies:
    - name: topology
      type: topology
      properties:
        clusters: ["staging", "production"]
  workflow:
    - name: deploy-staging
      type: deploy
      properties:
        policies: ["topology"]
```

| Pros | Cons |
|------|------|
| Full control over deployment | Higher complexity for developers |
| Built-in scaling, gateway, multi-cluster | Steeper learning curve |
| Workflow support (approvals, canary) | OAM concepts to learn |
| CNCF project, active development | |
| Extensible via CUE | |

---

### Qovery Configuration

Heroku-like simplicity with enterprise features.

```yaml
application:
  name: my-app
  project: acme
  publicly_accessible: true
databases:
  - type: postgresql
    version: "14"
    name: main-db
```

| Pros | Cons |
|------|------|
| Managed services built-in | Vendor-specific format |
| Simple format | Limited to Qovery platform |
| Environment management | |

---

### HashiCorp Waypoint

Workflow-centric approach with explicit build/deploy/release phases.

```hcl
project = "my-app"

app "web" {
  build {
    use "pack" {}  # Cloud Native Buildpacks
  }
  deploy {
    use "kubernetes" {
      replicas = 3
    }
  }
  release {
    use "kubernetes" {
      ingress {
        host = "myapp.example.com"
      }
    }
  }
}
```

| Pros | Cons |
|------|------|
| Explicit build → deploy → release | HCL, not YAML |
| Plugin-based extensibility | HashiCorp ecosystem dependency |
| Platform-agnostic deployment | Project maintenance uncertain |

---

## Key Insights

### 1. Score as a Foundation

Score is a compelling candidate for CozyIDP because:

- **Platform-agnostic**: No vendor lock-in, can target multiple platforms
- **Simple format**: Developers only describe what they need
- **Existing ecosystem**: Multiple implementations exist
- **Extensible**: Custom resource types supported
- **Active development**: Backed by Humanitec, growing community

**Potential workflow:**
```
Developer writes score.yaml
        ↓
score-cozystack generates Cozystack resources
        ↓
Cozystack creates HelmRelease + managed services
```

---

### 2. KubeVela for Advanced Use Cases

KubeVela OAM provides:

- Full control for platform engineers
- Multi-cluster deployment
- Workflow orchestration
- CNCF backing

Could serve as an "advanced mode" for users who need more control.

---

### 3. Flux Image Automation for GitOps

Cozystack already includes Flux with image-reflector-controller and image-automation-controller. This enables:

- No webhook endpoints needed
- GitOps-native image updates
- Audit trail in Git
- Multi-environment support via ImagePolicy

---

### 4. Hybrid Approach Recommendation

```
┌─────────────────────────────────────────────────────┐
│                    Developer                         │
├─────────────────────────────────────────────────────┤
│  Simple: score.yaml    │  Advanced: OAM Application │
├─────────────────────────────────────────────────────┤
│              CozyIDP Controller                      │
│  - Parses score.yaml OR Application                 │
│  - Creates managed services (PostgreSQL, Redis)     │
│  - Configures Flux Image Automation                 │
│  - Sets up environments (staging, prod, preview)    │
├─────────────────────────────────────────────────────┤
│                    Cozystack                         │
│  ApplicationDefinition → HelmRelease → Services     │
└─────────────────────────────────────────────────────┘
```

---

## Comparison Matrix

| Feature | Heroku | Railway | Score | KubeVela | Qovery |
|---------|--------|---------|-------|----------|--------|
| Self-hosted | No | No | Yes | Yes | Partial |
| Git push deploy | Yes | Yes | Via impl | Via workflow | Yes |
| Managed services | Yes | Yes | Via resources | Via components | Yes |
| Multi-environment | Yes | Yes | No (external) | Yes (policies) | Yes |
| Preview environments | Yes | Yes | No | Via workflow | Yes |
| Custom resources | No | No | Yes | Yes (CUE) | No |
| Kubernetes-native | No | No | Optional | Yes | Yes |
| CNCF project | No | No | No | Yes | No |
| Open source | No | No | Yes | Yes | Engine only |

---

## Recommendations for CozyIDP

### Option A: Score-based

Implement `score-cozystack` as a Score implementation:

- Developers use familiar `score.yaml` format
- CozyIDP translates to Cozystack ApplicationDefinition
- Add environment management as Score extension

### Option B: Custom Format (Score-inspired)

Create CozyIDP-specific format with:

- Score-like simplicity for resources
- Built-in environment management
- Cozystack-specific optimizations

### Option C: KubeVela Integration

Use KubeVela OAM directly:

- Leverage existing CNCF project
- Full workflow capabilities
- Higher learning curve for developers

### Option D: Hybrid (Recommended)

- Simple format (Score-compatible) for 80% use cases
- Escape hatch to full Cozystack/OAM for advanced needs
- CLI + UI for self-service

---

## Sources

- [Score Specification](https://docs.score.dev/docs/)
- [Score GitHub](https://github.com/score-spec/spec)
- [KubeVela Core Concepts](https://kubevela.io/docs/getting-started/core-concept/)
- [KubeVela OAM Model](https://kubevela.io/docs/platform-engineers/oam/oam-model)
- [Humanitec + Score Integration](https://humanitec.com/blog/deploy-a-workload-with-score-and-humanitec)
- [Qovery Engine (GitHub)](https://github.com/Qovery/engine)
- [HashiCorp Waypoint](https://www.hashicorp.com/en/blog/announcing-waypoint)
- [Kratix](https://kratix.io)
- [Backstage](https://backstage.io)
- [Railway vs Render vs Fly.io Comparison](https://ritza.co/articles/gen-articles/render-vs-heroku-vs-vercel-vs-railway-vs-fly-io-vs-aws/)
- [Top IDP Tools 2025](https://wso2.com/library/blogs/top-ten-internal-developer-platforms-compared-2025/)
- [Platform Engineering Tools](https://platformengineering.org/tools)
- [IDP Comparison (MetalBear)](https://metalbear.com/blog/comparison-of-internal-developer-platforms/)
