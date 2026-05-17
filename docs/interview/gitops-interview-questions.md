# GitOps Interview Questions — Market-Ready (2026)

Practical, scenario-based questions asked in senior DevOps / Platform Engineering / SRE interviews.
Covers GitOps, ArgoCD, Helm, Kubernetes internals, real-time troubleshooting, and behavioral.
Answers are grounded in the Zen Pharma platform but generalizable to any production setup.

> **Complements** `interview-questions-part2.md` (pipeline flow, drift detection, ESO, crash-loop debugging).
> This file covers: GitOps principles, ApplicationSet, Helm design, K8s networking internals, real-time scenarios, DB migrations, cost, and new behavioral angles.

---

## Table of Contents

| Group | Topic | Questions |
|-------|-------|-----------|
| [J](#group-j--gitops-core-concepts) | GitOps Core Concepts | J1–J7 |
| [K](#group-k--helm-chart-design) | Helm Chart Design | K1–K6 |
| [L](#group-l--kubernetes-internals--networking) | Kubernetes Internals & Networking | L1–L6 |
| [M](#group-m--real-time-scenarios--curveballs) | Real-time Scenarios & Curveballs | M1–M8 |
| [N](#group-n--design--architecture) | Design & Architecture | N1–N5 |
| [P](#group-p--behavioral--situational) | Behavioral & Situational | P1–P6 |

---

# Group J — GitOps Core Concepts

---

## J1

### Question
> "What are the four principles of GitOps? Give me a real example of each from a project you've worked on."

### What the interviewer is really testing
Not whether you can recite the OpenGitOps spec — whether you've actually lived these principles and can explain what breaks when one is violated.

### Answer

The four principles, with real examples:

**1. Declarative** — desired state is described, not scripted.
> In Zen Pharma, every service is defined as a Helm values file (`envs/dev/values-api-gateway.yaml`). We never run `kubectl scale` or `kubectl set image` directly. The values file says what we want; ArgoCD figures out the steps.

**2. Versioned and immutable** — desired state is stored in Git with full history.
> Every image tag change creates a Git commit with author, timestamp, and message. When prod had an incident, we could `git log envs/prod/values-api-gateway.yaml` and see exactly who changed what and when — no CloudTrail archaeology needed.

**3. Pulled automatically** — an agent inside the cluster pulls state, not a pipeline pushing in.
> ArgoCD runs inside EKS. It pulls from GitHub every 3 minutes. Our CI pipeline never holds cluster credentials — it only updates the gitops repo. This is critical for security: a compromised CI runner can't touch the cluster directly.

**4. Continuously reconciled** — the agent detects and corrects drift.
> If someone does a `kubectl edit deployment api-gateway` directly on prod, ArgoCD with `selfHeal: true` reverts it within the next sync cycle. Git wins, always.

**What breaks when you skip a principle:**
- Skip declarative → pipeline scripts become the truth, impossible to audit
- Skip versioned → no rollback story
- Skip pull → CI holds credentials, blast radius if CI is compromised
- Skip reconciliation → cluster drifts silently, "works on my kubectl" problems

---

## J2

### Question
> "Push-based vs pull-based delivery — what's the actual security difference? When would you still use push?"

### Answer

**Push-based:** The CI pipeline (GitHub Actions, Jenkins) has cluster credentials (kubeconfig) and runs `kubectl apply` or `helm upgrade` directly.

Security problem: The credentials that can modify production live in your CI system. If CI is compromised (supply chain attack, leaked env var), an attacker can deploy anything to production. You also need to poke firewall holes from CI to your private cluster API endpoint.

**Pull-based (GitOps):** ArgoCD runs inside the cluster. It authenticates to GitHub with a read-only deploy key. The cluster API never needs to be reachable from the outside world.

Security gain: Nothing outside the cluster has write access to the cluster. A compromised CI runner can push a malicious image tag to the gitops repo — but that PR still needs approval before ArgoCD acts on it.

**When push-based is still fine:**
- Bootstrapping (ArgoCD itself must be installed somehow — we push that once)
- Non-Kubernetes targets: pushing to S3, Lambda, CloudFront — GitOps pull model doesn't apply there
- Emergency one-off scripts where the overhead of committing to a gitops repo isn't worth it

**The honest answer interviewers want to hear:** "Pull is safer for cluster state. Push is fine for non-cluster targets or bootstrapping. In practice, most orgs run a hybrid."

---

## J3

### Question
> "What is ArgoCD ApplicationSet and how is it different from just creating multiple ArgoCD Applications?"

### Answer

An ArgoCD **Application** manages one app in one cluster/namespace.

An **ApplicationSet** is a controller that generates multiple Applications from a template + a generator. You write one ApplicationSet manifest and it produces N Applications automatically.

**Real use case — our 9 services × 3 environments:**

Without ApplicationSet: 27 separate Application YAMLs, mostly copy-paste.

With ApplicationSet using a matrix generator:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: zen-pharma-apps
spec:
  generators:
    - matrix:
        generators:
          - list:
              elements:
                - service: api-gateway
                - service: auth-service
                - service: drug-catalog-service
                # ... 6 more
          - list:
              elements:
                - env: dev
                  autoSync: "true"
                - env: qa
                  autoSync: "true"
                - env: prod
                  autoSync: "false"
  template:
    metadata:
      name: "{{env}}-{{service}}"
    spec:
      source:
        repoURL: https://github.com/org/zen-gitops
        path: helm-charts
        helm:
          valueFiles:
            - "../../envs/{{env}}/values-{{service}}.yaml"
      destination:
        namespace: "{{env}}"
```

27 Applications generated from ~40 lines.

**When ApplicationSet adds friction:** If each service has wildly different sync policies, health checks, or destinations, the template becomes an if-statement maze — at that point, explicit Application YAMLs are clearer.

---

## J4

### Question
> "What's the difference between `ignoreDifferences` and `syncOptions: ServerSideApply` in ArgoCD? When would you use each?"

### Answer

Both solve the "OutOfSync loop" problem but for different reasons.

**`ignoreDifferences`** — tells ArgoCD to ignore specific fields when comparing desired vs actual state.

Use when: A controller or admission webhook is mutating your resource after ArgoCD applies it. For example, a service mesh injects sidecar annotations, or the HPA controller writes `currentReplicas` into the deployment spec.

```yaml
ignoreDifferences:
  - group: apps
    kind: Deployment
    jsonPointers:
      - /spec/replicas          # HPA manages this, don't fight it
      - /metadata/annotations/kubectl.kubernetes.io~1last-applied-configuration
```

**`ServerSideApply: true`** — switches ArgoCD from client-side apply (`kubectl apply`) to server-side apply. Instead of ArgoCD owning the full resource, field ownership is tracked per-manager.

Use when: Multiple controllers legitimately write to the same resource (e.g., the HPA owns `spec.replicas`, ArgoCD owns everything else). Server-side apply lets them coexist without conflicts.

**In practice:** `ignoreDifferences` is a quick fix. `ServerSideApply` is the cleaner long-term answer but requires understanding field ownership semantics.

---

## J5

### Question
> "ArgoCD shows an app as `Synced` and `Healthy` but the app is clearly broken — users are getting 500 errors. How is that possible?"

### Answer

This is a common trap. `Synced` means Git state matches cluster state. `Healthy` means ArgoCD's health checks passed. Neither means "the app is working correctly."

**Most common reasons it happens:**

1. **Health check passes but app is functionally broken** — `/actuator/health` returns 200 (Spring Boot is up) but the downstream PostgreSQL connection is timing out. ArgoCD sees healthy; users see 500s.

2. **Readiness probe is too lenient** — readiness check hits `/actuator/health` (liveness path) instead of `/actuator/health/readiness`. The shallow check passes even if the app hasn't finished DB connection pool setup.

3. **ArgoCD custom health check isn't configured** — by default ArgoCD only checks standard Kubernetes health (pod running, deployment available). It doesn't know your app's business logic is broken.

4. **Traffic is going to old pods** — a deployment rolled out, ArgoCD says healthy, but the Service endpoints haven't updated yet, or old terminating pods are still serving.

5. **Config issue in prod values** — a wrong env var was promoted to prod. The app starts (so ArgoCD is happy) but every request fails. Example: wrong DB schema name or a missing API key.

**What to check:** ArgoCD health is Kubernetes-level health. Always also monitor application-level SLOs (error rate, latency) via Prometheus/Grafana separately from ArgoCD.

---

## J6

### Question
> "You want ArgoCD to automatically sync dev and qa but require a manual approval for prod. How do you configure this without maintaining three separate Application YAMLs?"

### Answer

Use ApplicationSet with conditional sync policy based on environment:

```yaml
generators:
  - list:
      elements:
        - env: dev
          autoSync: true
        - env: qa
          autoSync: true
        - env: prod
          autoSync: false
template:
  spec:
    syncPolicy:
      automated:
        enabled: "{{autoSync}}"   # interpolated per element
        selfHeal: true
        prune: true
```

For prod, `automated: { enabled: false }` means ArgoCD watches for drift and shows OutOfSync, but never acts automatically. An engineer reviews in ArgoCD UI and clicks Sync — providing a human gate without leaving the GitOps workflow.

**Additional hardening on prod:**
- ArgoCD RBAC: only senior engineers in the `prod-sync` role can trigger sync on prod Applications
- Sync Windows: `syncPolicy.syncOptions: [SyncWindow]` — restrict syncs to maintenance windows (e.g., only Tue/Thu 10am–12pm)
- Require approval PR to `envs/prod/*` via GitHub CODEOWNERS

---

## J7

### Question
> "What happens to running pods if the ArgoCD server itself goes down? Does your application stop working?"

### Answer

**Nothing happens to running pods.** ArgoCD is the delivery mechanism, not a runtime dependency.

Once ArgoCD has applied the Kubernetes manifests, the cluster runs them independently. The Kubernetes control plane (API server, scheduler, kubelet) maintains the pod lifecycle. ArgoCD is only needed for future syncs.

**What you lose while ArgoCD is down:**
- No drift detection — manual changes to the cluster won't be reverted
- No new deployments — you can't sync new Git commits
- No rollbacks via ArgoCD UI
- ArgoCD metrics and alerting go dark

**Mitigation:**
- ArgoCD itself should be HA: multiple replicas, application-controller in statefulset
- ArgoCD state is stored in Kubernetes CRDs — no external database to lose
- Bootstrap it with its own ArgoCD Application (App of Apps) so it's self-healing
- In an emergency while ArgoCD is down: `kubectl apply` directly using the gitops repo files

**The answer interviewers want:** "My application has zero runtime dependency on ArgoCD. ArgoCD outage = deployment freeze, not service outage."

---

# Group K — Helm Chart Design

---

## K1

### Question
> "You have 9 microservices. How do you design a single shared Helm chart for all of them while handling the differences — some are Spring Boot, one is Node.js, some need ingress, some don't?"

### Answer

Design the chart to be service-agnostic with everything configurable via values. Our `helm-charts/` chart for Zen Pharma works for all 9 services:

**What's parameterized in values.yaml:**
```yaml
image:
  repository: ""      # each service provides its ECR URI
  tag: "latest"

service:
  port: 8080          # notification-service overrides to 3000

ingress:
  enabled: false      # opt-in per service
  path: /api          # api-gateway sets /api; pharma-ui sets /

configmap: {}         # service-specific env vars passed as map

envFrom: []           # secretRef names — services add their own secrets
```

**Key design decisions:**
1. `ingress.enabled: false` default — services opt in, not opt out. pharma-ui and api-gateway enable it; backend services stay ClusterIP-only.
2. `configmap: {}` as a free-form map — Spring Boot services put `SPRING_PROFILES_ACTIVE`, Node.js puts `NODE_ENV`. The template iterates the map without caring about keys.
3. `fullnameOverride` — each service overrides the release name so K8s resource names are predictable (`api-gateway`, not `helm-release-api-gateway`).
4. Probes are configurable — Spring Boot needs `initialDelaySeconds: 60–90`, pharma-ui needs only `10`.

**What you should NOT parameterize:**
Security defaults (runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities) — these are non-negotiable and hardcoded in the chart template. No service can override them via values.

---

## K2

### Question
> "A Helm upgrade failed halfway through and left the release in a `failed` state. ArgoCD keeps trying to sync and failing. Walk me through recovering it."

### Answer

**Step 1 — Confirm the state**
```bash
helm list -n dev                           # status: failed
helm history api-gateway -n dev            # see which revision failed
```

**Step 2 — Understand what partially applied**
```bash
kubectl get pods -n dev -l app=api-gateway  # are new pods running or stuck?
kubectl get events -n dev --sort-by=.lastTimestamp | tail -20
```

**Step 3 — Choose recovery path**

Option A — rollback to last good revision:
```bash
helm rollback api-gateway <previous-revision> -n dev
```
This works if the previous revision is intact. ArgoCD will then show OutOfSync (cluster has old revision, git has new) but at least the service is up.

Option B — if rollback also fails (corrupted release state):
```bash
helm uninstall api-gateway -n dev    # wipes the release record
# then trigger ArgoCD sync — it does a fresh helm install
```

Option C — in ArgoCD directly:
Use `Sync → Force` with `Replace` checked. This runs `helm uninstall` + `helm install` in one operation. Be aware: brief downtime during the uninstall window.

**Root cause investigation after recovery:**
- Was it a bad values file? (`helm template . -f values.yaml` locally)
- Was it a CRD version mismatch?
- Was it a resource quota exceeded mid-upgrade?
- Was it a pre/post-upgrade hook that failed?

**What interviewers want to hear:** You know the difference between `helm rollback`, `helm uninstall`, and ArgoCD Force Sync — and you understand the downtime implications of each.

---

## K3

### Question
> "How do you manage Helm chart versioning when you have a shared chart used by multiple teams? How do you prevent a chart update from breaking a team's deployment?"

### Answer

The key tension: shared chart benefits from centralisation but changes can break consumers.

**Pattern 1 — Chart version pinned in ArgoCD Application**
Each ArgoCD Application specifies `targetRevision: 1.4.2` for the chart. Teams upgrade on their own schedule by bumping that version.
Risk: teams fall far behind and face a large breaking upgrade.

**Pattern 2 — Chart lives in the same gitops repo (our approach)**
The chart and the values files evolve together. When you change the chart template, you update all values files in the same PR. ArgoCD picks up both together.
Risk: one PR can affect all 9 services if you're careless. Mitigate with helm lint, helm template, and PR review.

**Pattern 3 — Separate chart repo with semantic versioning**
Chart is published to an OCI registry (ECR, Artifact Hub). Breaking changes bump major version. Teams stay on `^1.x` until ready to upgrade.
Good for: large orgs with many teams. Overhead: chart release pipeline, version matrix.

**In practice for Zen Pharma:** Pattern 2 — chart in the same repo. When we change the chart, we run `helm template` against all 9 values files in CI to catch regressions before merge.

---

## K4

### Question
> "What's Helm's `--atomic` flag? Should you use it in a GitOps workflow?"

### Answer

`helm upgrade --atomic` means: if the upgrade fails (pods not ready within the timeout), automatically run `helm rollback` to restore the previous release. In traditional CI/CD it's attractive because it's self-healing.

**In a GitOps workflow (with ArgoCD), you generally should NOT use it:**

1. ArgoCD doesn't use `--atomic` by default. If you want it, you set `syncPolicy.syncOptions: [RespectIgnoreDifferences]` and configure the Application — but even then, ArgoCD has its own retry/rollback logic.

2. `--atomic` rollback is invisible to Git. Git still says the new version should be deployed. ArgoCD will keep trying to sync the new version, fighting the `--atomic` rollback. You get a conflict loop.

3. In GitOps, rollback should happen in Git (revert the values file commit), not at the Helm layer. That keeps the audit trail intact.

**When `--atomic` is OK:** In a pure CI push-based workflow with no ArgoCD. For teams transitioning from push to pull-based, this is a common source of confusion.

---

## K5

### Question
> "How do you pass environment-specific secrets to pods via Helm without storing secret values in the values files?"

### Answer

The values file only references the **name** of the secret — never the value:

```yaml
# envs/dev/values-api-gateway.yaml
envFrom:
  - secretRef:
      name: db-credentials    # K8s Secret must already exist in the namespace
  - secretRef:
      name: jwt-secret
```

The Helm chart template renders this into the pod spec:
```yaml
envFrom:
  - secretRef:
      name: db-credentials
```

The actual secret (`db-credentials`) is created by the External Secrets Operator, which reads from AWS Secrets Manager. ESO is also deployed via GitOps — its ExternalSecret CRD references the secret path, not the value.

**The three-layer separation:**
1. Git (gitops repo): secret names + ESO ExternalSecret manifests — safe to commit
2. AWS Secrets Manager: actual secret values — encrypted, access-controlled via IAM
3. Kubernetes: Secret objects — synced by ESO, mounted into pods via envFrom

**Interview trap:** Some candidates say "use `--set` to pass secrets." Don't. `helm upgrade --set db.password=secret` puts the value in Helm's release history (stored in a Kubernetes Secret as base64, readable by anyone with cluster access). It also doesn't work with GitOps since there's no pipeline running helm commands.

---

## K6

### Question
> "What's the difference between `helm template` and `helm install --dry-run`? When do you use each?"

### Answer

**`helm template`** — renders the Kubernetes YAML locally without contacting the cluster. No cluster connection needed. Output goes to stdout.

Use for:
- CI validation: render all templates, pipe to `kubectl apply --dry-run=server` for schema validation
- Debugging: see exactly what YAML ArgoCD will apply
- Diff: `helm template` before and after a change to see what changed

```bash
helm template api-gateway ./helm-charts -f envs/dev/values-api-gateway.yaml
```

**`helm install --dry-run`** — connects to the cluster, renders templates AND runs server-side validation (checks if API versions exist, validates against CRD schemas). Doesn't actually apply.

Use for:
- Pre-flight check before upgrading a live release
- CRD version compatibility check
- Admission webhook validation (webhook runs during dry-run)

**Limitation of `helm template` in ArgoCD context:** It can't validate Kubernetes API versions or admission webhooks. A chart that passes `helm template` can still fail on the cluster if it uses a deprecated API version. This is why ArgoCD's diff (which does server-side dry-run) is more accurate.

---

# Group L — Kubernetes Internals & Networking

---

## L1

### Question
> "A pod in the `dev` namespace needs to call a service in the `monitoring` namespace. What's the DNS name it uses? Why does this matter for your microservice configuration?"

### Answer

The full DNS name is: `<service-name>.<namespace>.svc.cluster.local`

So the api-gateway in `dev` calling Prometheus in `monitoring` would use:
`prometheus-server.monitoring.svc.cluster.local:9090`

**Why this matters in practice:**

Services within the same namespace can use the short name (`auth-service:8081`). This is what our api-gateway configmap uses:
```yaml
AUTH_SERVICE_URL: "http://auth-service:8081"
```
This only works because api-gateway and auth-service are in the same `dev` namespace. CoreDNS resolves `auth-service` to `auth-service.dev.svc.cluster.local`.

If you moved auth-service to a different namespace (e.g., `shared-services`), you'd need to update the URL to `http://auth-service.shared-services.svc.cluster.local:8081`.

**Common interview follow-up:** "What if you have the same service name in two namespaces?" — Each is fully isolated. `auth-service.dev.svc.cluster.local` and `auth-service.qa.svc.cluster.local` are different endpoints. This is why namespace-per-environment works without naming conflicts.

---

## L2

### Question
> "What actually happens at the network level between the time NGINX Ingress decides to forward a request to `api-gateway:8080` and the packet reaching the pod?"

### Answer

Four steps happen invisibly:

**1. CoreDNS resolution**
NGINX resolves `api-gateway.dev.svc.cluster.local` → ClusterIP address (e.g., `10.100.42.15`). This ClusterIP is a virtual IP — no pod actually has this IP.

**2. iptables DNAT (kube-proxy)**
The packet hits a node's iptables rules (written by kube-proxy). The DNAT rule intercepts traffic to `10.100.42.15:8080` and rewrites the destination to one of the actual pod IPs (e.g., `10.0.3.7:8080`) — chosen by random or round-robin across all ready endpoints.

**3. Routing to the pod**
The rewritten packet (dst: `10.0.3.7`) is routed through the VPC overlay network (AWS VPC CNI, or Flannel/Calico depending on setup). The packet reaches the node where that pod is running.

**4. Pod receives**
The pod sees: `src=NGINX-pod-IP, dst=my-pod-IP:8080`. It does NOT see the original browser IP — that's in the `X-Forwarded-For` HTTP header set by NGINX.

**Why this matters in interviews:**
- "Can a ClusterIP service fail?" → The IP itself doesn't fail; pods behind it can. If all pods are down, the DNAT has nowhere to send and the connection is refused.
- "Why can't I ping a ClusterIP?" → It's a virtual IP with no device behind it. It only works with TCP/UDP through iptables — ICMP doesn't match those rules.

---

## L3

### Question
> "What is the difference between liveness, readiness, and startup probes? What's the most common misconfiguration you've seen?"

### Answer

**Liveness probe** — "Is the process alive?" If it fails, kubelet kills the container and restarts it. Use for: detecting deadlocks, infinite loops, memory corruption.

**Readiness probe** — "Is the app ready to serve traffic?" If it fails, the pod is removed from the Service's endpoints. Traffic stops routing to it. The pod is NOT restarted. Use for: startup warmup, temporary overload, dependency unavailable.

**Startup probe** — "Has the app finished starting?" Runs first; while it's failing, liveness is paused. Prevents liveness from killing a slow-starting container during init.

**Most common misconfiguration (seen constantly in the wild):**

Using the same path for both liveness and readiness:
```yaml
livenessProbe:
  path: /actuator/health          # checks DB connection, dependency health
readinessProbe:
  path: /actuator/health          # same
```

Problem: `/actuator/health` in Spring Boot checks all health indicators including DB. If the DB blips for 30 seconds, the liveness probe fails → pod gets killed → restarts → DB blip causes another kill → you've made a transient DB issue into a CrashLoopBackOff.

**Correct approach:**
```yaml
livenessProbe:
  path: /actuator/health/liveness   # only checks if process is alive
readinessProbe:
  path: /actuator/health/readiness  # checks dependencies — removes from LB if DB is down
```
This is exactly what Zen Pharma uses. Liveness kills only when the process itself is broken. Readiness handles temporary dependency issues gracefully.

---

## L4

### Question
> "Your HPA (Horizontal Pod Autoscaler) is configured but the pods aren't scaling up even though CPU looks high. What do you check?"

### Answer

In order of likelihood:

**1. Metrics Server not installed or not working**
```bash
kubectl top pods -n dev           # if this fails, Metrics Server is the problem
kubectl get apiservice v1beta1.metrics.k8s.io
```

**2. HPA can't see metrics**
```bash
kubectl describe hpa api-gateway -n dev
# Look for: "unable to get metrics", "FailedGetScale"
```

**3. Resource requests not set on the pod**
HPA calculates utilization as `current CPU / requested CPU`. If `requests.cpu` is not set, the formula is undefined and HPA doesn't scale.

**4. HPA min/max bounds**
`maxReplicas: 3` and already at 3 → can't scale further. Check `kubectl get hpa -n dev`.

**5. Cooldown period**
HPA has a scale-up stabilization window (default 0s) and scale-down window (default 5min). It won't scale up again immediately after a recent scale event.

**6. Wrong metric target type**
`targetCPUUtilizationPercentage: 70` means 70% of the requested CPU. If pods are requested at 100m and using 80m, that's 80% — should trigger. But if the metric is measuring the node-level CPU (wrong), the numbers won't match.

**For Spring Boot specifically:** JVM's garbage collector can cause CPU spikes that are transient. HPA might scale up, GC finishes, HPA scales back down. This thrashing is solved by tuning the stabilization window or using custom metrics (request rate) instead of CPU.

---

## L5

### Question
> "What is a PodDisruptionBudget and when is it critical to have one?"

### Answer

A PodDisruptionBudget (PDB) defines the minimum number of pods that must remain available during **voluntary disruptions** — node drains, cluster upgrades, manual evictions. It does NOT protect against involuntary disruptions (node failure, OOMKill).

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-gateway-pdb
  namespace: dev
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: api-gateway
```

**When it's critical:**

1. **Node upgrades / Spot instance interruptions** — AWS may drain a node for an upgrade. Without a PDB, all pods on that node are evicted simultaneously. With `minAvailable: 1`, the drain waits for a replacement pod to be ready before evicting the next one.

2. **Rolling deployments on small replica counts** — with 2 replicas and no PDB, a rolling update can take both pods offline briefly. PDB prevents this.

3. **Cluster autoscaler scale-down** — the autoscaler won't evict a pod if it would violate the PDB.

**What we should add to Zen Pharma:** All production services should have a PDB. Our api-gateway and auth-service are critical paths — if both pods are evicted simultaneously during a node drain, users can't log in or make API calls.

**Common mistake:** Setting `minAvailable: 1` with only 1 replica. This fully blocks node drain forever — a PDB requires that at least 1 pod is available, but there's only 1, so none can be evicted. Always have `replicas >= minAvailable + 1` for the PDB to be meaningful.

---

## L6

### Question
> "How do you do a zero-downtime deployment for a service that has a database schema change — like adding a NOT NULL column to a table with 10 million rows?"

### Answer

This is the "expand-contract" pattern. Never do a migration that requires simultaneous app + DB changes. Break it into backward-compatible phases:

**Phase 1 — Expand (backward compatible)**
```sql
ALTER TABLE drugs ADD COLUMN expiry_date DATE NULL;  -- nullable first
```
Deploy this migration while the OLD app version runs. Both old and new app work fine — old ignores the column, new writes to it.

**Phase 2 — Deploy new app version**
The new Spring Boot drug-catalog-service writes `expiry_date` on every insert. Old pods (still running during rolling update) write NULL. Both are valid because column is nullable.

**Phase 3 — Backfill + constraint**
After 100% of traffic is on the new version:
```sql
UPDATE drugs SET expiry_date = '2099-01-01' WHERE expiry_date IS NULL;
ALTER TABLE drugs ALTER COLUMN expiry_date SET NOT NULL;
```

**Phase 4 — Contract (optional cleanup)**
Remove any old code paths that handled NULL.

**In GitOps:** The migration SQL runs as a Kubernetes Job (or Flyway/Liquibase init container) that deploys separately from the app. The Job is tracked in Git — it's part of the release. You can see in the gitops history exactly when a migration ran.

**What interviewers are probing:** Whether you know that "deploy code + migrate DB simultaneously" is how production tables get locked and downtime happens.

---

# Group M — Real-time Scenarios & Curveballs

---

## M1

### Question
> "GitHub goes down. You have an urgent hotfix that must reach production in 30 minutes. What do you do?"

### Answer

**Immediate actions:**

1. **Check if ArgoCD already has the current Git state cached** — ArgoCD caches the last successful Git fetch. If the fix is already in the cached state, you can trigger a manual sync without GitHub being up.

2. **Apply directly to the cluster** — this is the emergency "break glass" procedure:
```bash
# Get the current prod values
kubectl get configmap api-gateway -n prod -o yaml > /tmp/api-gateway-backup.yaml
# Apply the fix directly
kubectl set image deployment/api-gateway pharma-service=873135413040.dkr.ecr.us-east-1.amazonaws.com/api-gateway:hotfix-sha
```

3. **Document everything** — note the time, the exact command, the reason. When GitHub is back, immediately create a PR with the same change so Git reflects reality.

4. **Git mirror** — if your org has a GitLab/Bitbucket mirror of the repo, point ArgoCD at the mirror temporarily.

**After the incident:**
- GitOps repo is now behind the cluster (drift). Update the values file with the hotfix image tag immediately when GitHub is back.
- Consider whether ArgoCD selfHeal would revert your emergency fix before you can update Git.  If selfHeal is on, ArgoCD WILL revert your kubectl change. Disable selfHeal temporarily, fix Git, re-enable.

**Lesson for the interviewer:** "GitOps doesn't remove your ability to kubectl apply — it just makes it visible as drift. The process is: apply emergency fix → Git catches up → drift resolves."

---

## M2

### Question
> "A developer says 'it works in dev but breaks in prod'. How do you systematically investigate configuration drift between environments?"

### Answer

This is a difference-finding problem. Work through each layer:

**1. Image tag**
```bash
kubectl get deployment api-gateway -n dev -o jsonpath='{.spec.template.spec.containers[0].image}'
kubectl get deployment api-gateway -n prod -o jsonpath='{.spec.template.spec.containers[0].image}'
```
Same image? If not — prod has older code.

**2. Environment variables**
```bash
kubectl exec -n dev deployment/api-gateway -- env | sort > /tmp/dev-env.txt
kubectl exec -n prod deployment/api-gateway -- env | sort > /tmp/prod-env.txt
diff /tmp/dev-env.txt /tmp/prod-env.txt
```

**3. Resource limits** — a service that works on dev (500m CPU limit) may OOMKill or throttle heavily on prod if limits are tighter.
```bash
kubectl get deployment api-gateway -n dev -o yaml | grep -A5 resources
kubectl get deployment api-gateway -n prod -o yaml | grep -A5 resources
```

**4. Secrets** — dev has a test DB password, prod has a rotated one that's wrong, or the ESO sync failed.
```bash
kubectl get externalsecret -n prod     # check sync status
kubectl describe secret db-credentials -n prod
```

**5. Network policies** — prod may have NetworkPolicies that dev doesn't. A downstream call that's allowed in dev may be blocked in prod.

**6. Replicas and HPA** — prod may have 3 replicas and one has a stale in-memory cache. Test by hitting each pod directly:
```bash
kubectl get pods -n prod -l app=api-gateway -o wide
kubectl exec -n prod <pod-name> -- curl localhost:8080/actuator/health
```

**The real answer interviewers want:** Systematic, layer-by-layer. Not "check the logs" as step 1.

---

## M3

### Question
> "Your team wants to add a 10th microservice to the platform. Walk me through everything you need to do end-to-end."

### Answer

In order:

**Application side:**
1. New GitHub repo for the service code (or add to monorepo)
2. Dockerfile, multi-stage build
3. GitHub Actions CI workflow: build, test, push to ECR on merge to main

**AWS side:**
4. Create ECR repository: `aws ecr create-repository --repository-name compliance-service`
5. Create RDS schema: `CREATE SCHEMA IF NOT EXISTS compliance;` + grant permissions
6. Create AWS Secrets Manager entries if the service needs secrets
7. IAM role for the service account (IRSA) if it needs AWS access

**Gitops repo:**
8. Create values files for each environment:
   - `envs/dev/values-compliance-service.yaml`
   - `envs/qa/values-compliance-service.yaml`
   - `envs/prod/values-compliance-service.yaml`
9. Create ArgoCD Application (or add to ApplicationSet if using that pattern)
10. Add to NGINX Ingress if it needs an external path

**Kubernetes:**
11. If the service is called by api-gateway: add `COMPLIANCE_SERVICE_URL` to api-gateway configmap
12. RBAC — does it need cluster-level permissions? Add to RBAC manifests.
13. If it needs to call other services: check NetworkPolicy allows it

**Operational:**
14. Add health check endpoints to monitoring (Prometheus ServiceMonitor)
15. Add to on-call runbook
16. Update architecture diagram

**The answer interviewers respect:** You went beyond "just apply a YAML" — you covered the full surface: AWS, Git, ArgoCD, monitoring, operational readiness.

---

## M4

### Question
> "You need to reduce your EKS cluster costs by 40%. You can't turn off any production services. What do you do?"

### Answer

In order of effort vs impact:

**Quick wins (week 1):**

1. **Right-size resource requests** — most Spring Boot services over-request. Run `kubectl top pods -n prod` and compare actual usage to requested. If api-gateway uses 80m CPU but requests 100m, reduce requests. Smaller requests = better bin packing = fewer nodes.

2. **Scale down non-prod outside business hours** — dev and qa don't need to run 24/7. A CronJob or KEDA scale-to-zero setup shuts down dev namespace at 8pm, restarts at 7am. Saves ~30% on non-prod node costs.

3. **Remove unused resources** — `kubectl get all -n dev` — any orphaned services, ConfigMaps, PVCs from old experiments?

**Medium-term (month 1):**

4. **Spot instances for non-critical workloads** — dev/qa on Spot node groups (up to 70% cheaper). Use `tolerations` and `nodeSelector` to keep prod on On-Demand. Configure PodDisruptionBudgets so Spot interruptions don't cause outages.

5. **Graviton (ARM) node groups** — AWS Graviton instances are 20% cheaper and often faster for Java workloads. Multi-arch Docker images needed.

6. **RDS right-sizing** — check if the dev/qa RDS instances are over-provisioned. A `db.t3.medium` is often sufficient for dev.

**Architecture changes (quarter 1):**

7. **Consolidate dev/qa to one cluster** — if they're on separate clusters, share infra. Use namespaces for isolation.

8. **KEDA for event-driven scaling** — notification-service doesn't need constant replicas if it only fires on events. Scale to zero when idle.

**What to present to stakeholders:** Show a before/after cost estimate. CloudWatch Container Insights gives per-namespace cost visibility. Always lead with "here's what we save and here's the risk."

---

## M5

### Question
> "You're mid-rollout on a production deployment and users start reporting elevated error rates. The new pods are 50% deployed. What's your immediate action?"

### Answer

**Immediate — stop the rollout:**
```bash
kubectl rollout pause deployment/api-gateway -n prod
```
This freezes the rolling update. New pods stop being created. Existing traffic splits between old (50%) and new (50%) pods.

**Assess the situation:**
```bash
kubectl get pods -n prod -l app=api-gateway    # which pods are old vs new
kubectl logs -n prod -l app=api-gateway --tail=50 | grep ERROR
```

**If the new pods are clearly bad — roll back:**
```bash
kubectl rollout undo deployment/api-gateway -n prod
```
This restores the previous ReplicaSet. Old pods come back up, new pods terminate. In GitOps: also revert the image tag in `envs/prod/values-api-gateway.yaml` immediately after, otherwise ArgoCD will try to re-deploy the bad version.

**If root cause is fixable fast (config issue, wrong env var):**
Fix it in Git, push, and let ArgoCD continue — but only after confirming the fix actually resolves the errors in dev first.

**If unsure:**
Keep the rollout paused. Don't undo yet. Investigate logs, check Grafana error rate per pod (to confirm new pods are the problem, not both). Make a deliberate decision, not a panic rollback.

**What interviewers are testing:** Your instinct to pause before fixing, not undo immediately. A rollback that introduces its own issues (e.g., old version also has a bug) is worse than a controlled pause.

---

## M6

### Question
> "Your NGINX Ingress Controller pod is healthy but all ingress routes are returning 404. No app pods are showing errors. What happened?"

### Answer

404 from NGINX itself (not from the app) means NGINX doesn't have a matching location block.

**Likely causes in order:**

**1. Ingress class mismatch**
```bash
kubectl get ingress -n dev api-gateway -o yaml | grep ingressClassName
# Must match the controller: ingressClassName: nginx
```
If someone removed or changed `ingressClassName`, the NGINX controller ignores that Ingress. The NGINX controller only manages Ingresses that claim it.

**2. Ingress not picked up after creation**
```bash
kubectl describe ingress api-gateway -n dev
# Check: "Events" — does the controller acknowledge it?
kubectl logs -n kube-system -l app.kubernetes.io/name=ingress-nginx | grep api-gateway
```

**3. The Ingress resource has a host that doesn't match the request**
```yaml
rules:
  - host: pharma.example.com    # requests to wrong host → 404
```
If you hit the IP directly or use a different hostname, NGINX has no matching rule.

**4. Path type mismatch**
`pathType: Exact` vs `pathType: Prefix` — with Exact, `/api` only matches that exact path. `/api/drugs` returns 404.

**5. NGINX controller restarted and hasn't re-synced Ingress resources**
Check controller logs for re-sync messages. Usually resolves in under a minute.

**6. Namespace annotation**
Some NGINX controllers require `kubernetes.io/ingress.class: nginx` annotation instead of `ingressClassName`. If the cluster upgraded NGINX controller version, the behavior may have changed.

```bash
# Quick diagnostic:
kubectl exec -n kube-system <nginx-pod> -- nginx -T | grep location
```
This dumps the live nginx.conf. If your service's location block is missing — NGINX never picked up the Ingress.

---

## M7

### Question
> "A Node in your cluster shows `NotReady`. It has pods running the auth-service and api-gateway. What's your sequence of actions?"

### Answer

**Step 1 — Don't panic. Check if pods have already rescheduled.**
```bash
kubectl get pods -n prod -l app=api-gateway -o wide   # are new pods running on other nodes?
kubectl get nodes                                       # which node is NotReady?
```
Kubernetes will reschedule pods from NotReady nodes after the pod eviction timeout (default: 5 minutes). If pods are already running elsewhere, users may not have noticed.

**Step 2 — Diagnose the node**
```bash
kubectl describe node <node-name>          # check Conditions, Events
kubectl get events -A | grep <node-name>   # cluster-wide events
```
Common causes: disk pressure, memory pressure, network partition, kubelet crash, EC2 instance hardware failure.

**Step 3 — Check if it's coming back**
```bash
kubectl get node <node-name> -w            # watch status
```
If it comes back Ready within 2–3 minutes: transient network issue. Monitor but no action needed.

**Step 4 — If node is genuinely broken: cordon + drain**
```bash
kubectl cordon <node-name>      # no new pods scheduled here
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```
`drain` evicts all pods (respecting PodDisruptionBudgets — critical for api-gateway). Evicted pods reschedule on healthy nodes.

**Step 5 — Terminate the EC2 instance**
If it's an ASG-managed node, terminating it triggers a replacement. In EKS managed node groups, the ASG handles this automatically.

**What NOT to do:** Force-delete pods (`kubectl delete pod --force --grace-period=0`) before draining — this bypasses PDBs and can cause split-brain for stateful apps.

---

## M8

### Question
> "How do you handle a situation where the RDS PostgreSQL is at 95% CPU and queries are timing out? Your team is paging you at 2am."

### Answer

**Immediate (first 5 minutes):**

1. **Identify the culprit query**
```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND (now() - pg_stat_activity.query_start) > interval '5 seconds'
ORDER BY duration DESC;
```

2. **Kill long-running queries** — be careful, only kill safe ones:
```sql
SELECT pg_terminate_backend(pid) WHERE pid = <problematic-pid>;
```

3. **Check for lock contention**
```sql
SELECT * FROM pg_locks WHERE granted = false;
```

**Short-term mitigation:**
- Scale up the RDS instance (RDS supports vertical scaling with a brief restart)
- Enable RDS Performance Insights if not already on — instant visibility
- Add read replica and redirect read-heavy services to it

**Root cause investigation:**
- New deployment? Check if a new app version added a missing index query
- Scheduled job? CRON that runs a full table scan at 2am
- Traffic spike? Check ALB access logs for unusual request volume
- Missing index? `EXPLAIN ANALYZE` the slow query

**GitOps angle interviewers sometimes ask:** "How do you make RDS changes GitOps-managed?" — Infrastructure as code (Terraform) in a separate `zen-infra` repo. RDS parameter group changes, read replica configs are all in code. The gitops repo manages application state; the infra repo manages infrastructure.

---

# Group N — Design & Architecture

---

## N1

### Question
> "How would you structure a GitOps repository for a platform with 50 microservices across dev, staging, qa, and prod environments?"

### Answer

Two viable patterns:

**Pattern A — Monorepo (what Zen Pharma uses, scales to ~50 services)**
```
zen-gitops/
├── helm-charts/          # one shared chart for all services
├── envs/
│   ├── dev/
│   │   ├── values-service-a.yaml
│   │   ├── values-service-b.yaml
│   │   └── ...
│   ├── qa/
│   ├── staging/
│   └── prod/
├── argocd/
│   ├── dev-apps.yaml        # ApplicationSet for dev
│   ├── qa-apps.yaml
│   └── prod-apps.yaml
└── k8s/
    └── namespaces.yaml
```
Pros: one PR for a cross-service change, unified history, simple ArgoCD setup.
Cons: any team can see all service configs (including other teams' env vars), one broken PR can block everyone.

**Pattern B — Poly-repo (for 50+ services with multiple teams)**
Each team owns their gitops repo. A central platform team maintains the shared Helm chart in a chart repo.
```
team-catalog/gitops:  envs/{dev,prod}/values-*.yaml + ArgoCD Apps
team-auth/gitops:     envs/{dev,prod}/values-*.yaml + ArgoCD Apps
platform/helm-charts: published to OCI registry
```
Pros: team autonomy, blast radius isolated.
Cons: chart upgrades require PRs to each repo, harder to see cross-team dependencies.

**What to say in interviews:** "For 50 services with one platform team, I'd use monorepo. For 50 services with 10 product teams, poly-repo with a central chart registry and ArgoCD ApplicationSet per team."

---

## N2

### Question
> "How do you implement image promotion across environments automatically but with a human gate before prod?"

### Answer

**The full promotion pipeline:**

```
CI builds image (sha-abc123) → ECR
         │
         ▼
CI bot opens PR to gitops repo:
  Update envs/dev/values-api-gateway.yaml: tag: sha-abc123
         │
PR merged (auto, or dev lead approves)
         ▼
ArgoCD auto-syncs dev
         │
Dev tests pass (automated integration tests / smoke tests)
         │
         ▼
Promotion bot opens PR:
  Update envs/qa/values-api-gateway.yaml: tag: sha-abc123
         │
QA engineer approves PR
         │
ArgoCD auto-syncs QA
         │
QA sign-off
         │
         ▼
Promotion bot opens PR:
  Update envs/prod/values-api-gateway.yaml: tag: sha-abc123
         │
CODEOWNERS: requires 2 senior engineer approvals
         │
PR merged → ArgoCD shows OutOfSync (manual sync policy on prod)
         │
Release manager clicks Sync in ArgoCD UI
```

**Key principle:** The same image SHA flows through all environments. You're not rebuilding — you're promoting the exact artifact that passed all tests.

**Tooling for the promotion bot:** A GitHub Action triggered on successful ArgoCD sync of dev/qa, using `yq` or `sed` to update the image tag in the target environment's values file and open a PR automatically.

---

## N3

### Question
> "Your team is debating whether to use Kustomize or Helm for managing Kubernetes manifests. What's your recommendation and why?"

### Answer

Both are valid. The choice depends on the use case:

**Helm:**
- Best when: you're packaging a chart for distribution (other teams/orgs use it), you need templating logic (loops, conditionals), you want Helm's release management (history, rollback, atomic upgrades).
- Weakness: Go templating is powerful but hard to read/debug. YAML + template directives together is messy.
- Zen Pharma uses Helm because: one chart for 9 services, consistent values interface, ArgoCD native support.

**Kustomize:**
- Best when: you want plain YAML + patches (no templating language), you're managing internal apps only, you want to extend base manifests without forking them.
- Strength: output is pure YAML, easy to read what will be applied. No Helm release state to manage.
- Weakness: complex environments need many patches; logic (if-then) is hard to express.

**In practice, many teams use both:**
- Kustomize for base manifests + environment overlays
- Helm for third-party charts (NGINX, cert-manager, Prometheus)

ArgoCD supports both natively. You can point an Application at a Kustomize directory or a Helm chart.

**Interview answer:** "I'd use Helm if we need a reusable chart across services or orgs. Kustomize if the team is YAML-native and wants to avoid Go templating. For a platform team managing their own apps, either works — consistency within the team matters more than which tool."

---

## N4

### Question
> "How would you add observability to this platform? What's the minimum viable setup and what does a mature setup look like?"

### Answer

**Minimum viable (week 1):**
1. Prometheus + Grafana via `kube-prometheus-stack` Helm chart
2. All Spring Boot services already expose `/actuator/prometheus` (see `MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: health,info,metrics,prometheus`)
3. Add a `ServiceMonitor` CRD per service — Prometheus auto-discovers targets
4. Out-of-the-box dashboards: CPU, memory, JVM heap, GC time, HTTP request rate

**Mature setup (month 1–3):**

**Metrics:**
- Custom business metrics in each service (orders processed, drugs looked up)
- Prometheus AlertManager: alert on error rate > 1%, p99 latency > 500ms, pod restarts
- SLO dashboards (availability %, error budget burn rate)

**Logging:**
- Fluent Bit DaemonSet ships pod logs to CloudWatch Logs / OpenSearch
- Structured logging (JSON) from all services so logs are queryable
- Log retention policy (dev: 7 days, prod: 90 days for compliance)

**Tracing:**
- OpenTelemetry SDK in each Spring Boot service (auto-instrumentation via Java agent)
- Traces to AWS X-Ray or Tempo
- Trace ID in logs for correlation

**Alerting:**
- PagerDuty/OpsGenie integration with AlertManager
- Runbook links in alert annotations
- Alert routing by severity and namespace (prod alerts page on-call, dev alerts go to Slack)

**What's specific to Zen Pharma:** The pharma domain has compliance requirements. All security events (failed auth, unauthorized access attempts) must be logged and retained for audit. This is a separate log stream from operational logs.

---

## N5

### Question
> "How would you implement canary deployments for the api-gateway to reduce risk when deploying changes?"

### Answer

**Option 1 — Argo Rollouts (native, most control)**

Replace the standard Deployment with a Rollout resource:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 10       # 10% of traffic to new version
        - pause: {duration: 5m}
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100      # full rollout if no issues
      analysis:
        templates:
          - templateName: error-rate   # auto-rollback if error rate > 1%
```
NGINX Ingress splits traffic using weight annotations. Prometheus provides the analysis metrics.

**Option 2 — NGINX Ingress traffic split (simpler, less automation)**

Create two Services: `api-gateway-stable` and `api-gateway-canary`. Use NGINX annotation:
```yaml
nginx.ingress.kubernetes.io/canary: "true"
nginx.ingress.kubernetes.io/canary-weight: "10"
```
Manually adjust weight and monitor. Promote or roll back manually. Less automation but works without Argo Rollouts.

**Option 3 — Feature flags (no Kubernetes changes)**

Deploy the new code to all pods but gate the feature with a feature flag (LaunchDarkly, Unleash). Release to 5% of users via flag. This decouples deployment from release — the safest model but requires instrumentation in the app code.

**What interviewers want to hear:** You know multiple options and can articulate when each is appropriate. Option 1 is most complete. Option 3 is most production-safe for high-risk features. Option 2 is the pragmatic starting point.

---

# Group P — Behavioral & Situational

---

## P1

### Question
> "Your manager asks you to bypass the PR approval process and deploy directly to prod because of a business deadline. How do you handle it?"

### Answer

This is a values question as much as a process question.

**My response in this situation:**

First, understand the actual urgency: "What breaks if we wait 30 minutes for a proper review?" Usually the answer is "the deadline is a business commitment, not a technical emergency." In that case, I'd find one other engineer for a quick review — 15 minutes is enough for a targeted change.

If it's a genuine emergency (production is down, customers are impacted right now), then I'd:
1. Apply the fix directly (break-glass procedure)
2. Document everything I did in real-time
3. Immediately after: create the PR, get the retrospective review, and update the gitops repo so it matches production

What I would not do: bypass the process habitually, or bypass without documentation. The second time you bypass, you're not in an emergency — you're building a culture where bypass is normal.

**What I'd say to my manager:** "I can get this deployed in 15 minutes with a quick review rather than 5 minutes without one. The extra 10 minutes protects us from deploying something that makes the situation worse. Can we get [colleague] on a call?"

**The answer interviewers respect:** Shows you're not blindly rule-following (you'll break glass in real emergencies) but you also push back on process erosion.

---

## P2

### Question
> "You've been asked to migrate a legacy application to Kubernetes. The development team is resistant — they say 'our app works fine on EC2.' How do you approach this?"

### Answer

Resistance to Kubernetes is almost always rational — the team is worried about:
- Learning curve (new ops model, new debugging paradigm)
- Risk (what if we break production?)
- Effort (migration takes time away from features)

**My approach:**

1. **Listen first** — don't lead with "Kubernetes is better." Ask what specific pain they feel now: deployments, scaling, reliability? If they have no pain, maybe the migration isn't urgent.

2. **Don't pitch Kubernetes — pitch the outcomes:**
   - "Right now deployments take 45 minutes and require SSH access. On EKS, it's 3 minutes with full audit history."
   - "You're paying for an always-on EC2. With K8s, dev scales to zero overnight."
   Concrete benefits they care about beat abstract architecture improvements.

3. **Start small, show it working** — migrate one non-critical service first. Let the team see it working. Success builds confidence better than any slide deck.

4. **Don't abandon them after migration** — the biggest killer of Kubernetes adoption is "the platform team migrated us and then disappeared." Pair-program the first Helm chart with a dev from their team. Write runbooks for common operations.

5. **Acknowledge the genuine downsides** — Kubernetes adds complexity. CrashLoopBackOff debugging is harder than `tail -f /var/log/app.log`. If they have a 5-line bash app with no scaling needs, maybe EC2 is genuinely the right answer.

**What interviewers are testing:** Empathy, pragmatism, ability to get organizational buy-in. Technical correctness is table stakes — the real skill is bringing people along.

---

## P3

### Question
> "Tell me about a time you made a mistake that caused a production issue. What happened and what did you change afterward?"

### Answer

*(Frame this around a realistic GitOps scenario.)*

**The situation:** We enabled `selfHeal: true` on all ArgoCD Applications including production, to prevent manual drift. What we didn't account for: our certificate rotation process involved temporarily patching a Secret manually in the prod namespace while the new cert was being issued. ArgoCD detected the drift and reverted the Secret to the old cert within 3 minutes — before the new cert was ready. The auth-service couldn't validate JWTs, and users got 401s for about 8 minutes until we manually disabled selfHeal.

**What I did wrong:** Treated production the same as dev for sync policies. Didn't map out which operational processes touched cluster resources directly outside of Git.

**What I changed:**
1. Production ArgoCD Applications now have manual sync + selfHeal: false
2. We created a runbook for cert rotation that includes "pause ArgoCD sync before patching"
3. We moved cert rotation itself into a GitOps workflow — ESO now manages the cert Secret from ACM, so no manual patching needed
4. Added a pre-mortm item: "audit which ops processes write directly to the cluster; migrate to GitOps or document them"

**What interviewers are looking for:** Ownership (no blame-shifting), specific root cause analysis, systemic fix (not just "I'll be more careful"), and reflection on what changed in the process.

---

## P4

### Question
> "How do you communicate a major infrastructure change — like migrating from a monolith to microservices or from EC2 to EKS — to non-technical stakeholders?"

### Answer

Non-technical stakeholders care about three things: risk, timeline, and cost. Not architecture.

**Framework I use:**

1. **Lead with the problem you're solving, not the technology:**
   "Our current deployment process takes 2 hours and requires a full application restart, causing 5-minute downtime every release. This limits us to deploying twice a month."
   
   Not: "We're migrating to Kubernetes because microservices are more scalable."

2. **Quantify the benefit:**
   "After migration: deployments take 5 minutes, zero downtime, and we can release 10× per day."

3. **Be honest about the risk:**
   "The migration will take 8 weeks. During weeks 4–6, we'll run both old and new systems in parallel, which increases infra cost by ~$3,000/month temporarily."

4. **Give them a decision, not a proposal:**
   "We recommend proceeding. The 8-week timeline fits before the Q3 product launch. If you want to wait until Q4, we can — but the current system won't handle the 3× traffic growth expected from the campaign."

5. **One-pager, not a 30-slide deck:**
   Problem → Proposed solution → Risk → Timeline → Cost → Decision needed.

**What interviewers are testing:** Whether you can translate technical work into business language. Senior engineers own this skill. "I just implement what I'm told" is a junior answer.

---

## P5

### Question
> "Your team is on-call and you receive a PagerDuty alert at 3am: `api-gateway error rate > 5%`. Walk me through your thought process and actions in the first 10 minutes."

### Answer

**Minute 0–2: Situate**
- Acknowledge the alert so the team knows it's being handled
- Open Grafana: is this a spike or sustained? What started when?
- Check ArgoCD: was there a recent deployment? (`kubectl rollout history deployment/api-gateway -n prod`)
- Check the error type: 500 (server error), 503 (no pods available), 502 (NGINX can't reach pods)?

**Minute 2–5: Scope**
- How many users affected? Check if it's all endpoints or one (e.g., `/api/drugs` only)
- Are other services impacted or just api-gateway?
- Is the error rate climbing or stable?

**Minute 5–8: Mitigate**
- If a deployment happened in last 30 minutes → roll back: `kubectl rollout undo deployment/api-gateway -n prod`
- If pods are crash-looping → check logs: `kubectl logs -n prod -l app=api-gateway --tail=100`
- If it's a downstream service (auth-service returning 500) → isolate that service

**Minute 8–10: Communicate**
- Post in the incident Slack channel (or create one): "api-gateway error rate elevated since 2:53am, investigating, initial suspicion is [X]"
- Loop in whoever might be needed (DB admin if it's a DB query issue, network team if it's connectivity)

**What I don't do:** Start randomly restarting pods ("percussive maintenance"). Have a hypothesis before taking action.

**After resolution:** 5-line incident summary written immediately while it's fresh. Full post-mortem within 48 hours. Timeline, root cause, remediation items, who owns each item.

---

## P6

### Question
> "A senior engineer on your team disagrees with your GitOps approach — they say ArgoCD adds unnecessary complexity and they can deploy with a simple `kubectl apply` in a CI pipeline. How do you handle the disagreement?"

### Answer

This is a legitimate technical debate, not a clear right-or-wrong situation.

**First: engage with their argument, don't dismiss it.**

"Tell me more about what complexity you're finding in ArgoCD. Is it the setup, the debug experience, the concepts?" The answer tells you whether their concern is valid experience or unfamiliarity.

Their points may be right for their context:
- Small teams with 2–3 services may genuinely not need ArgoCD
- `kubectl apply` in CI is simpler to reason about
- ArgoCD requires its own operational overhead (upgrades, HA, backups)

**Where I'd push back with evidence:**

"The push model requires the CI pipeline to have cluster credentials. In our threat model, that's a significant risk. Who has access to those credentials? Can you show me the full audit trail of who deployed what to prod last month? With GitOps, it's `git log`."

"When we had a bad deploy at 2am, the rollback was `git revert` — one command, immediate, full audit trail. What's the rollback story with your push pipeline at 3am?"

**Resolution approach:**

"Let's timebox an experiment. Run both approaches for 60 days on different services. Track: deployment frequency, mean time to recovery, number of prod incidents caused by deployment. Decide based on data."

**What interviewers are testing:** Intellectual honesty (you're willing to be wrong), data-driven decision making (you propose an experiment), and team dynamics (you don't steamroll — you build consensus with evidence).

---

*End of gitops-interview-questions.md*
*See also: `DevOps_Interview_Full_2026.md` (architecture walkthrough) and `interview-questions-part2.md` (ArgoCD, ESO, incident debugging commands).*
