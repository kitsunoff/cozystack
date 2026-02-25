# CozyIDP Architecture Notes

## Core Principle: Simple as Fuck

**CozyIDP must not become another Backstage.**

No heavy portals. No React apps with plugin ecosystems. No 500MB node_modules. No "platform team required to maintain". No cognitive overload.

### Design Philosophy

```
Backstage:     "Here's a framework, build your own portal"
                → 6 months later, 3 engineers maintaining it
                → Nobody understands how it works
                → Upgrade = pain

CozyIDP:       "Here's your app running"
                → 5 minutes to deploy
                → Zero maintenance
                → Just works
```

### Simplicity Rules

1. **If it needs a manual, it's too complex**
2. **If it needs a dedicated team, it's too complex**
3. **If developer asks "how do I...", we failed**
4. **Convention over configuration, always**
5. **Zero YAML is better than minimal YAML**

### What "Simple" Means

| Aspect | Complex (avoid) | Simple (goal) |
|--------|-----------------|---------------|
| **Install** | Helm + values + secrets + config | `kubectl apply` one manifest |
| **First app** | Read docs, write YAML, configure CI | `cozy init && git push` |
| **Add database** | Create CRD, configure operator, inject secrets | `cozy add postgres` |
| **View logs** | Find pod name, kubectl logs, grep | `cozy logs` |
| **Check status** | Dashboard → namespace → deployment → pods | `cozy status` |

### Anti-Patterns to Avoid

- ❌ Plugin systems
- ❌ Template languages (except Go templates for secrets)
- ❌ Custom DSLs
- ❌ "Extensibility frameworks"
- ❌ Multiple ways to do the same thing
- ❌ Configuration options that nobody uses
- ❌ Abstractions on top of abstractions

### Success Criteria

**Junior developer with zero Kubernetes knowledge should:**
1. Deploy first app in < 10 minutes
2. Add PostgreSQL in < 1 minute
3. Never see a Kubernetes manifest
4. Never ask "what namespace is my app in"
5. Never debug YAML indentation

---

## Architecture: Kubernetes-native, Zero Bloat

Just Kubernetes CRDs + CLI + existing Cozystack Dashboard.

---

## Why NOT Backstage

| Aspect | Backstage | What we want |
|--------|-----------|--------------|
| **Stack** | React + Node + PostgreSQL + plugins | Kubernetes-native |
| **Installation** | `npx @backstage/create-app` → 500MB node_modules | `kubectl apply` |
| **Maintenance** | Needs JS developer, plugin updates | GitOps, Helm upgrade |
| **Plugins** | Each plugin = separate dependency, versions, bugs | Built-in or not needed |
| **Updates** | Pain: yarn upgrade, breaking changes | Helm upgrade |
| **Target** | Spotify with 1000+ services | Normal companies |

**Verdict:** Backstage is overkill. Too heavy for what we need.

---

## What Developer Actually Needs

```
1. Create app          →  CLI or simple UI
2. Add services        →  "I want postgres"
3. Deploy              →  git push
4. Check status        →  UI or CLI
5. View logs           →  CLI
6. Environment vars    →  Auto-injected from services
```

That's it. No React portal with plugins needed.

---

## Proposed Architecture

```
┌──────────────────────────────────────────┐
│              Developer                    │
│                                          │
│   $ cozy init                            │
│   $ cozy services add postgres           │
│   $ git push                             │
│   $ cozy status                          │
│   $ cozy logs                            │
│                                          │
├──────────────────────────────────────────┤
│         CozyIDP Controller               │
│         (single Go binary)               │
│                                          │
│   - Watches Git repos                    │
│   - Auto-discovers apps                  │
│   - Creates Cozystack resources          │
│   - Manages Flux Image Automation        │
├──────────────────────────────────────────┤
│         Cozystack Dashboard              │
│         (already exists)                 │
│                                          │
│   + Add: apps list view                  │
│   + Add: create app wizard               │
│   + Add: environment switcher            │
└──────────────────────────────────────────┘
```

---

## Components

### 1. CozyIDP Controller

Single Go binary running in Kubernetes.

**Responsibilities:**
- Watch GitRepository sources
- Discover apps (Containerfile, package.json, go.mod)
- Parse cozy.yaml (when present)
- Create Cozystack Application resources
- Configure Flux ImagePolicy + ImageUpdateAutomation
- Manage environments (staging, production, preview)

**Installation:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cozyidp-controller
  namespace: cozy-system
spec:
  template:
    spec:
      containers:
        - name: controller
          image: ghcr.io/cozystack/cozyidp:latest
```

### 2. CRDs (minimal set)

```yaml
# App - represents a deployable application
apiVersion: cozyidp.io/v1
kind: App
metadata:
  name: my-api
  namespace: tenant-acme
spec:
  source:
    git:
      url: https://github.com/acme/my-api
      branch: main
      path: ./apps/api
  services:
    - postgres
    - redis
  environments:
    production:
      replicas: 3
    staging:
      replicas: 1
```

```yaml
# Platform - represents a monorepo with multiple apps
apiVersion: cozyidp.io/v1
kind: Platform
metadata:
  name: acme-platform
spec:
  source:
    git:
      url: https://github.com/acme/platform
  discovery:
    apps: "./apps/*"
    workers: "./workers/*"
  services:
    - postgres
    - redis
```

### 3. CLI (`cozy`)

```bash
# Initialize new app
$ cozy init
Detected: Go (go.mod found)
? Add PostgreSQL? Yes
? Add Redis? No
Created cozy.yaml

# Add service
$ cozy services add redis
Added redis to cozy.yaml

# Deploy status
$ cozy status
my-api (production)
├── web: 3/3 replicas ready
├── postgres: healthy
└── redis: healthy

# Logs
$ cozy logs
$ cozy logs --service web --tail 100

# Environment variables
$ cozy env
DATABASE_URL=postgresql://...
REDIS_URL=redis://...

# Open dashboard
$ cozy dashboard
```

### 4. Cozystack Dashboard Extensions

Extend existing dashboard with:

- **Apps List:** Show all discovered and deployed apps
- **App Details:** Status, logs, environment variables
- **Create App Wizard:** Step-by-step app creation
- **Environment Switcher:** Toggle between staging/production/preview

---

## Discovery Mechanism

### Convention over Configuration

CozyIDP discovers apps automatically based on directory structure:

```
my-platform/
├── apps/           # HTTP services
│   ├── api/
│   │   └── Containerfile  → App "api"
│   └── web/
│       └── Containerfile  → App "web"
├── workers/        # Background workers
│   └── mailer/
│       └── Containerfile  → Worker "mailer"
├── jobs/           # Cron jobs
│   └── cleanup/
│       ├── Containerfile  → Job "cleanup"
│       └── schedule       → Cron expression
└── cozy.yaml       # Optional: only for overrides
```

### Auto-detection Rules

| What | How detected |
|------|--------------|
| App name | Directory name |
| App type | Parent directory (apps/, workers/, jobs/) |
| Port | `EXPOSE` in Containerfile, or default 8080 |
| Services needed | Dependencies in package.json/go.mod/requirements.txt |
| Cron schedule | `schedule` file or directory name pattern |

### Override Only When Needed

```yaml
# apps/api/cozy.yaml - only specify what differs from defaults
replicas: 5
resources:
  memory: 2Gi
```

No need to repeat name, port, type - derived from convention.

---

## GitOps Flow with Flux

Cozystack already includes Flux with image-reflector-controller and image-automation-controller.

### Flow

```
Developer pushes code
        ↓
External CI (GitLab/GitHub) builds image
        ↓
Image pushed to registry
        ↓
Flux ImageReflector detects new tag
        ↓
Flux ImageAutomation commits new tag to Git
        ↓
Flux HelmController deploys updated app
```

### No Webhooks Needed

- Flux polls registry (configurable interval)
- Updates are committed to Git (audit trail)
- No external endpoints to secure

---

## Deployment Model

### Simple Case

```yaml
# cozy.yaml (minimal)
services:
  - postgres
```

CozyIDP:
1. Discovers Containerfile
2. Creates ImageRepository + ImagePolicy
3. Creates Cozystack PostgreSQL Application
4. Creates HelmRelease for the app
5. Injects DATABASE_URL environment variable

### Monorepo Case

```yaml
# cozy.yaml (root)
discovery:
  apps: "./apps/*"
  workers: "./workers/*"

services:
  - postgres
  - redis

environments:
  production:
    branch: main
  staging:
    branch: develop
```

CozyIDP:
1. Scans directories matching patterns
2. Creates resources for each discovered app/worker
3. Shared services across all apps
4. Per-environment configuration

---

## What We DON'T Build

- React portal (use Cozystack Dashboard)
- Plugin system (keep it simple)
- Built-in CI (use external: GitLab, GitHub Actions)
- Documentation hosting (out of scope)
- Service mesh (Cozystack/Kubernetes handles this)

---

## Technology Choices

| Component | Choice | Why |
|-----------|--------|-----|
| Controller | Go | Kubernetes ecosystem, single binary |
| CRDs | Kubebuilder | Standard K8s tooling |
| CLI | Go (Cobra) | Single binary, cross-platform |
| Image automation | Flux | Already in Cozystack |
| Dashboard | Cozystack Dashboard | Already exists, extend it |
| Config format | YAML | Kubernetes convention |

---

## Multi-Cluster Tenant Architecture

### Cozystack Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INFRA CLUSTER                                   │
│                      (Cozystack Control Plane)                       │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │   Operators  │  │   Cozystack  │  │     Flux     │              │
│  │  (CNPG,      │  │  Controller  │  │  Controllers │              │
│  │   Redis Op)  │  │              │  │              │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  ┌──────────────────────────────────────────────────┐              │
│  │              Managed Services                     │              │
│  │  PostgreSQL, Redis, MongoDB, Kafka, etc.         │              │
│  │  (live here, accessible from tenant clusters)    │              │
│  └──────────────────────────────────────────────────┘              │
│                                                                      │
│  ┌──────────────────────────────────────────────────┐              │
│  │              Tenant K8s Clusters                  │              │
│  │  (Kamaji control planes, secrets with kubeconfig)│              │
│  └──────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
           │                    │                    │
           │ credentials        │ credentials        │ credentials
           ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ TENANT CLUSTER  │  │ TENANT CLUSTER  │  │ TENANT CLUSTER  │
│   (team-alpha)  │  │   (team-beta)   │  │   (team-gamma)  │
│                 │  │                 │  │                 │
│  Applications   │  │  Applications   │  │  Applications   │
│  (user code)    │  │  (user code)    │  │  (user code)    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### Key Insight: Apps Deploy to Tenant Clusters

- **Managed services** (PostgreSQL, Redis) run in **infra cluster**
- **Applications** (user code) run in **tenant clusters**
- **Secrets** must be synced from infra → tenant clusters

---

## Secret Delivery: No CrossPlane, No Vault (for MVP)

### Why CrossPlane is NOT Needed

CrossPlane is for provisioning cloud resources (AWS RDS, GCP CloudSQL, etc.).

**But Cozystack already has:**
- CloudNative-PG operator for PostgreSQL
- Redis Operator for Redis
- Helm charts for all managed services
- Secrets created automatically by operators

CrossPlane would be redundant abstraction.

### Why Vault is NOT Needed (for MVP)

Vault adds:
- Another component to maintain
- Complexity for secret rotation
- Learning curve

**For MVP:** Simple SecretSync is enough. Vault can be added later for:
- Secret rotation
- Audit logging
- Fine-grained access control

---

## SecretSync Controller

### The Problem

```
INFRA CLUSTER                          TENANT CLUSTER
┌─────────────────────────┐           ┌─────────────────────────┐
│                         │           │                         │
│  PostgreSQL (CNPG)      │           │      App Deployment     │
│  namespace: tenant-X    │           │                         │
│                         │           │      Needs:             │
│  Secret:                │     ?     │      DATABASE_URL       │
│  my-db-credentials      │ ────────▶ │                         │
│  - username: app        │           │                         │
│  - password: xyz123     │           │                         │
└─────────────────────────┘           └─────────────────────────┘
```

### The Solution: SecretSync CRD

```yaml
apiVersion: cozyidp.io/v1
kind: SecretSync
metadata:
  name: my-db-to-app
  namespace: tenant-alpha
spec:
  source:
    secretName: my-db-credentials
    # namespace = current (tenant-alpha in infra cluster)

  target:
    cluster: tenant-alpha-k8s  # tenant kubernetes cluster
    namespace: production
    secretName: database-credentials

  # Template: transform fields into connection strings
  template:
    DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@my-db-postgresql.tenant-alpha.svc:5432/app"
    PGHOST: "my-db-postgresql.tenant-alpha.svc"
    PGPORT: "5432"
    PGUSER: "{{ .username }}"
    PGPASSWORD: "{{ .password }}"
```

### How SecretSync Controller Works

```
1. Watch SecretSync CRD in infra cluster
2. Read source Secret from infra cluster
3. Apply template (build DATABASE_URL, etc.)
4. Get kubeconfig for tenant cluster from Kamaji secret
   (Secret: {cluster-name}-admin-kubeconfig, key: super-admin.svc)
5. Create/Update Secret in tenant cluster
6. Watch for source Secret changes, re-sync automatically
```

### Getting Tenant Cluster Kubeconfig

Cozystack uses Kamaji for tenant clusters. Kubeconfig is stored in:

```
Secret: {cluster-name}-admin-kubeconfig
Namespace: tenant-{name}
Key: super-admin.svc
```

To get kubeconfig:
```bash
kubectl get secret -n tenant-alpha kubernetes-alpha-admin-kubeconfig \
  -o go-template='{{ index .data "super-admin.svc" | base64decode }}'
```

---

## Full Flow: Developer Creates App with PostgreSQL

### Step 1: Developer Creates App

```yaml
apiVersion: cozyidp.io/v1
kind: App
metadata:
  name: my-api
  namespace: tenant-alpha  # tenant namespace in infra cluster
spec:
  source:
    git:
      url: https://github.com/acme/my-api

  target:
    cluster: tenant-alpha-k8s  # where to deploy the app
    namespace: production

  services:
    - postgres  # request PostgreSQL
```

### Step 2: CozyIDP Controller Actions

```
1. Create PostgreSQL Application in infra cluster
   ┌────────────────────────────────────────┐
   │ apiVersion: apps.cozystack.io/v1alpha1 │
   │ kind: PostgreSQL                       │
   │ metadata:                              │
   │   name: my-api-postgres                │
   │   namespace: tenant-alpha              │
   │ spec:                                  │
   │   size: small                          │
   │   users:                               │
   │     app: {}                            │
   │   databases:                           │
   │     app:                               │
   │       roles:                           │
   │         admin: [app]                   │
   └────────────────────────────────────────┘

2. Wait for PostgreSQL to be ready
   - CNPG creates Cluster
   - Secret my-api-postgres-credentials created

3. Create SecretSync
   ┌────────────────────────────────────────┐
   │ apiVersion: cozyidp.io/v1              │
   │ kind: SecretSync                       │
   │ metadata:                              │
   │   name: my-api-postgres-sync           │
   │   namespace: tenant-alpha              │
   │ spec:                                  │
   │   source:                              │
   │     secretName: my-api-postgres-creds  │
   │   target:                              │
   │     cluster: tenant-alpha-k8s          │
   │     namespace: production              │
   │     secretName: my-api-db              │
   │   template:                            │
   │     DATABASE_URL: "postgresql://..."   │
   └────────────────────────────────────────┘

4. Wait for Secret synced to tenant cluster

5. Deploy App to tenant cluster
   - Create HelmRelease or Deployment
   - Reference synced secret for env vars
```

### Step 3: Result in Tenant Cluster

```yaml
# Secret (synced by SecretSync)
apiVersion: v1
kind: Secret
metadata:
  name: my-api-db
  namespace: production
stringData:
  DATABASE_URL: "postgresql://app:xyz123@my-api-postgres.tenant-alpha.svc:5432/app"

---
# Deployment (created by CozyIDP)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
  namespace: production
spec:
  template:
    spec:
      containers:
        - name: app
          image: registry.example.com/acme/my-api:latest
          envFrom:
            - secretRef:
                name: my-api-db
```

---

## What We DON'T Need

| Component | Needed? | Why |
|-----------|---------|-----|
| CrossPlane | ❌ | Cozystack already manages resources via operators |
| Vault | ❌ (MVP) | SecretSync is enough, add Vault later for rotation |
| External Secrets Operator | ❌ (MVP) | SecretSync is simpler for our use case |
| Service Binding Spec | ❌ | Overkill, env vars are enough |
| Backstage | ❌ | Too heavy, Cozystack Dashboard exists |

---

## Updated Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      INFRA CLUSTER                                   │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                    CozyIDP Controller                          │ │
│  │                                                                │ │
│  │  Watches:                     Creates:                         │ │
│  │  - App CRD                    - Cozystack Applications         │ │
│  │  - Platform CRD               - SecretSync resources           │ │
│  │  - SecretSync CRD             - HelmRelease in tenant cluster  │ │
│  │                               - Flux ImagePolicy               │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────┐    │
│  │  PostgreSQL  │  │    Redis     │  │  Tenant K8s Cluster    │    │
│  │  (CNPG)      │  │  (Operator)  │  │  (Kamaji)              │    │
│  │              │  │              │  │                        │    │
│  │  Secret:     │  │  Secret:     │  │  Secret:               │    │
│  │  *-creds     │  │  *-creds     │  │  *-admin-kubeconfig    │    │
│  └──────┬───────┘  └──────┬───────┘  └────────────┬───────────┘    │
│         │                 │                       │                 │
│         └────────┬────────┘                       │                 │
│                  │                                │                 │
│         ┌────────▼────────┐                       │                 │
│         │   SecretSync    │◀──────────────────────┘                 │
│         │   Controller    │      uses kubeconfig                    │
│         └────────┬────────┘                                         │
└──────────────────┼──────────────────────────────────────────────────┘
                   │
                   │ push secrets
                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      TENANT CLUSTER                                  │
│                                                                      │
│  ┌──────────────────┐     ┌──────────────────┐                      │
│  │      Secret      │────▶│   App Deployment │                      │
│  │  (synced)        │     │                  │                      │
│  │                  │     │  env:            │                      │
│  │  DATABASE_URL    │     │    DATABASE_URL  │                      │
│  │  REDIS_URL       │     │    REDIS_URL     │                      │
│  └──────────────────┘     └──────────────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Open Questions

1. **Network connectivity:** How do apps in tenant cluster reach PostgreSQL in infra cluster?
   - Option A: Expose via LoadBalancer/NodePort
   - Option B: Cross-cluster networking (Cilium ClusterMesh?)
   - Option C: VPN/tunnel between clusters

2. **Secret rotation:** When PostgreSQL password changes, how to update tenant secrets?
   - SecretSync should watch source and re-sync automatically

3. **Preview environments:** How to handle PR-based previews?
   - Create temporary namespace in tenant cluster
   - Create temporary services or use shared services?

4. **Resource quotas:** How to enforce limits per tenant?
   - ResourceQuota in tenant namespace (infra cluster)
   - LimitRange in tenant cluster

5. **Multi-tenant cluster:** Can multiple tenants share one K8s cluster?
   - Namespace isolation with NetworkPolicy
   - Or dedicated cluster per tenant (current model)

---

## Next Steps

1. Define CRD schemas (App, Platform, SecretSync)
2. Implement SecretSync controller
3. Design CLI commands and UX
4. Prototype full flow: App → PostgreSQL → SecretSync → Deploy
5. Extend Cozystack Dashboard for app management
6. Document network connectivity options
