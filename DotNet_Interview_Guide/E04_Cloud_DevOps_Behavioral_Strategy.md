# E04 — Cloud, DevOps, Behavioral & Interview Strategy

Audience: .NET developer (C#, .NET, EF Core, MSSQL). Ladder tags: Easy → Follow-up → Medium → Hard → Tricky.

## Contents
1. Docker  2. Kubernetes  3. Azure Essentials  4. CI/CD  5. Git  6. IaC  7. Deployment Strategies  8. Monitoring & Alerting  9. Azure Functions  10. Cost/Performance  11. Serverless vs Containers vs VMs  12. Twelve-Factor  — then Behavioral & Career, Interview Strategy.

---

## Topic 1: Docker

**Study notes**
- An **image** is an immutable, layered, read-only filesystem plus metadata; a **container** is a running instance (image + thin writable layer + isolated namespaces/cgroups).
- Each Dockerfile instruction that changes the filesystem (`RUN`, `COPY`, `ADD`) creates a layer. Layers are cached; order instructions from least to most frequently changing (restore before copying source).
- **Multi-stage builds**: build with the SDK image (`mcr.microsoft.com/dotnet/sdk`), run on the smaller runtime image (`mcr.microsoft.com/dotnet/aspnet`). Final image has no compilers/source.
- Networking drivers: `bridge` (default, single host), `host`, `none`, `overlay` (multi-host/Swarm). On user-defined bridge networks, containers resolve each other by name (built-in DNS); on the default bridge they do not.
- Volumes persist data outside the container lifecycle; bind mounts map host paths.
- .NET 8+ official images run as non-root capability: `USER $APP_UID` and default port **8080** (not 80) for ASP.NET images.
- Compose: declarative multi-container local environments (`compose.yaml`), `depends_on` with `condition: service_healthy`.

### 1.1 [Easy] What is the difference between an image and a container?
**Answer**: An image is a read-only, layered template; a container is a running (or stopped) instance of an image with a writable layer and its own process/network/filesystem namespaces.
**Explanation**: Many containers can start from one image and share its read-only layers (copy-on-write). Deleting a container discards its writable layer unless data is in a volume.
**Code/config**
```bash
docker build -t myapi:1.0 .
docker run -d --name api -p 8080:8080 myapi:1.0
docker ps; docker logs api; docker exec -it api sh
```

### 1.2 [Follow-up] Write a multi-stage Dockerfile for an ASP.NET Core API. Why multi-stage?
**Answer**: Use the SDK image to restore/build/publish, then copy only the publish output into the aspnet runtime image. It yields a smaller, safer image and better layer caching.
**Explanation**: Copy `.csproj` files first and run `dotnet restore` so the restore layer is cached until dependencies change. The runtime image excludes the SDK, shrinking size and attack surface.
**Code/config**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/MyApi/MyApi.csproj", "src/MyApi/"]
RUN dotnet restore "src/MyApi/MyApi.csproj"
COPY . .
RUN dotnet publish "src/MyApi/MyApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
USER $APP_UID
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### 1.3 [Medium] Why does changing one source file rebuild everything, and how do you fix slow builds?
**Answer**: Because `COPY . .` precedes `dotnet restore`, any file change invalidates the restore layer. Copy only project/props files, restore, then copy the rest. Add a `.dockerignore` (bin, obj, .git, node_modules).
**Explanation**: Cache invalidation cascades: once a layer changes, all subsequent layers rebuild. BuildKit cache mounts (`RUN --mount=type=cache,target=/root/.nuget/packages`) can also speed restores in CI.
**Code/config**
```dockerfile
# .dockerignore
**/bin
**/obj
.git
*.md
```

### 1.4 [Medium] How do containers talk to each other and to SQL Server in Compose?
**Answer**: Compose creates a default network; services reach each other by service name (e.g., `Server=db,1433`). Publish ports only for what the host needs.
**Explanation**: `depends_on` controls start order only; use a healthcheck + `condition: service_healthy`, and EF migration/retry logic (`EnableRetryOnFailure`) for readiness. Never put the SA password in the compose file in plain text for shared repos — use env files/secrets.
**Code/config**
```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      ConnectionStrings__Default: "Server=db,1433;Database=App;User Id=sa;Password=${SA_PASSWORD};TrustServerCertificate=True"
    depends_on:
      db:
        condition: service_healthy
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: ${SA_PASSWORD}
    volumes: [sqldata:/var/opt/mssql]
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools18/bin/sqlcmd -C -S localhost -U sa -P \"$$MSSQL_SA_PASSWORD\" -Q 'SELECT 1' || exit 1"]
      interval: 10s
      retries: 10
volumes:
  sqldata:
```
Note: `__` in env var names maps to `:` in .NET configuration.

### 1.5 [Tricky] Why does your container work locally but ignore Ctrl+C / lose requests on stop, and why can it fail with "address already in use / cannot bind port 80"?
**Answer**: (a) `docker stop` sends SIGTERM; ASP.NET Core's host handles it for graceful shutdown (default 30 s in .NET 8 via `HostOptions.ShutdownTimeout`, 5 s in earlier versions) — using shell-form `ENTRYPOINT dotnet app.dll` wraps in `/bin/sh` which may not forward signals; use exec form. (b) .NET 8 images listen on 8080 and run non-root, which cannot bind ports <1024; set `ASPNETCORE_HTTP_PORTS`.
**Explanation**: Exec-form `["dotnet","MyApi.dll"]` makes dotnet PID 1 so it receives SIGTERM. Also `localhost` inside a container is the container itself, not the host or other containers.
**Code/config**
```dockerfile
ENV ASPNETCORE_HTTP_PORTS=8080
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**Interview traps**
- Saying containers are "lightweight VMs": they share the host kernel.
- Baking secrets into image layers (they persist in history even if deleted later).
- Using `latest` tags in production; pin versions/digests.
- `localhost` inside containers; `depends_on` ≠ readiness.
- Shell-form ENTRYPOINT breaking signal handling.

**Cheat sheet**
- Order: copy csproj → restore → copy src → publish.
- SDK image to build, aspnet image to run; non-root; port 8080.
- `.dockerignore`, exec-form ENTRYPOINT, healthchecks, volumes for state.
- Secrets via env/secret stores at runtime, never in layers.

---

## Topic 2: Kubernetes Basics for .NET Developers

**Study notes**
- **Pod**: smallest deployable unit (one or more containers sharing network/storage). Ephemeral.
- **Deployment**: declarative desired state; manages a ReplicaSet; supports rolling updates and rollback (`kubectl rollout undo`).
- **Service**: stable virtual IP/DNS for pods selected by labels. Types: `ClusterIP` (default), `NodePort`, `LoadBalancer`. **Ingress** (or Gateway API) provides HTTP routing.
- **Probes**: `startupProbe` (slow starts), `readinessProbe` (receive traffic?), `livenessProbe` (restart?). ASP.NET Core: `AddHealthChecks()` + `MapHealthChecks`.
- **Config**: ConfigMap (non-secret), Secret (base64-encoded, not encrypted by default — enable encryption at rest / use CSI Secrets Store with Key Vault).
- **HPA**: HorizontalPodAutoscaler scales replicas on CPU/memory/custom metrics; requires resource `requests` and metrics-server.
- Resource `requests` (scheduling) vs `limits` (enforced; CPU throttled, memory over limit → OOMKilled).

### 2.1 [Easy] Pod vs Deployment vs Service?
**Answer**: Pod runs containers; Deployment keeps N replicas of a pod template and handles rollouts; Service gives a stable endpoint load-balancing across matching pods.
**Explanation**: Pods get new IPs when recreated, so you never address them directly.
**Code/config**
```bash
kubectl get pods,deploy,svc
kubectl rollout status deploy/api
kubectl rollout undo deploy/api
```

### 2.2 [Follow-up] Write a Deployment + Service for a .NET API with probes.
**Answer**: See YAML; readiness checks dependencies, liveness is lightweight.
**Explanation**: Don't make liveness check the database — a DB outage would restart all pods (cascading failure). Readiness may check dependencies so traffic is withheld.
**Code/config**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api }
spec:
  replicas: 3
  selector: { matchLabels: { app: api } }
  strategy:
    rollingUpdate: { maxSurge: 1, maxUnavailable: 0 }
  template:
    metadata: { labels: { app: api } }
    spec:
      containers:
      - name: api
        image: myacr.azurecr.io/myapi:1.0.3
        ports: [{ containerPort: 8080 }]
        envFrom:
        - configMapRef: { name: api-config }
        env:
        - name: ConnectionStrings__Default
          valueFrom: { secretKeyRef: { name: api-secrets, key: sql } }
        resources:
          requests: { cpu: 250m, memory: 256Mi }
          limits: { memory: 512Mi }
        readinessProbe: { httpGet: { path: /health/ready, port: 8080 }, periodSeconds: 5 }
        livenessProbe:  { httpGet: { path: /health/live,  port: 8080 }, periodSeconds: 10 }
        startupProbe:   { httpGet: { path: /health/live,  port: 8080 }, failureThreshold: 30, periodSeconds: 2 }
---
apiVersion: v1
kind: Service
metadata: { name: api }
spec:
  selector: { app: api }
  ports: [{ port: 80, targetPort: 8080 }]
```
```csharp
builder.Services.AddHealthChecks().AddDbContextCheck<AppDbContext>("db", tags: new[] { "ready" });
app.MapHealthChecks("/health/live", new() { Predicate = _ => false });
app.MapHealthChecks("/health/ready", new() { Predicate = r => r.Tags.Contains("ready") });
```
(`AddDbContextCheck` is from `Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore`.)

### 2.3 [Medium] How do you configure and secure settings/secrets for a .NET app?
**Answer**: ConfigMap → env vars/mounted files; Secrets via K8s Secrets (with encryption at rest) or Azure Key Vault CSI driver / Workload Identity. .NET config reads env vars with `__` as section separator.
**Explanation**: Secrets are only base64-encoded by default and visible to anyone with RBAC read access. Prefer workload identity + Key Vault so no secret lives in the cluster.
**Code/config**
```bash
kubectl create configmap api-config --from-literal=Logging__LogLevel__Default=Information
kubectl create secret generic api-secrets --from-literal=sql='Server=...'
```

### 2.4 [Hard] How does HPA work and what breaks it?
**Answer**: HPA periodically (default 15 s) computes `desired = ceil(current * currentMetric / targetMetric)` and adjusts replicas within min/max. It needs CPU `requests` defined and metrics-server.
**Explanation**: Percent utilization is relative to *requests*. Without requests, CPU-based HPA can't work. Scaling doesn't help if the bottleneck is the SQL database or connection pool. KEDA scales on queue depth (Service Bus) — better for workers.
**Code/config**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: api }
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

### 2.5 [Tricky] Pods are "Running" but users get errors during a rollout / pod restarts with exit code 137. Why?
**Answer**: Exit 137 = SIGKILL, commonly OOMKilled (memory limit exceeded) or a terminationGracePeriod expiry. Errors during rollout come from missing readiness probes, or in-flight requests cut off because the app doesn't drain before termination.
**Explanation**: .NET GC is container-aware (respects cgroup limits), but a tight limit plus large heaps still OOMs. Add readiness probe, `preStop` sleep (few seconds) to let endpoints deregister, and ensure graceful host shutdown. `kubectl describe pod` shows `Last State: OOMKilled`.
**Code/config**
```yaml
lifecycle:
  preStop: { exec: { command: ["sleep","5"] } }
terminationGracePeriodSeconds: 45
```

**Interview traps**
- Liveness probe depending on external services.
- No resource requests → HPA/scheduler problems.
- "Secrets are encrypted" — not by default.
- Running DB migrations from every replica at startup (race); use a Job/init step.
- Storing state in pod filesystem.

**Cheat sheet**
- Deployment → ReplicaSet → Pods; Service selects by label.
- startup/readiness/liveness roles; liveness stays cheap.
- requests = scheduling/HPA basis; limits = enforcement.
- `kubectl describe`, `logs --previous`, `rollout undo`, `get events`.

---

## Topic 3: Azure Essentials

**Study notes**
- **App Service**: PaaS web hosting (Windows/Linux, code or container); deployment slots (Standard tier+) for swap-based zero-downtime; autoscale rules; "Always On" for non-idle.
- **Azure Functions**: event-driven serverless (see Topic 9).
- **Container Apps**: serverless containers on Kubernetes (KEDA, Dapr built in), scale to zero, no cluster management. **AKS**: managed Kubernetes for full control.
- **Service Bus**: enterprise messaging — queues (point-to-point), topics/subscriptions (pub/sub), sessions (ordering), dead-letter queue, duplicate detection, at-least-once delivery (PeekLock). **Storage Queues** simpler/cheaper; **Event Hubs** high-throughput streaming; **Event Grid** reactive event routing.
- **Key Vault**: secrets, keys, certificates; access via RBAC or access policies; soft delete + purge protection.
- **Azure SQL**: managed SQL Server (single DB, elastic pool, Managed Instance); DTU vs vCore models; built-in backups/PITR, geo-replication, failover groups; serverless tier auto-pauses.
- **Blob Storage**: object storage; tiers Hot/Cool/Cold/Archive; SAS tokens, user-delegation SAS.
- **Application Insights**: APM on Azure Monitor/Log Analytics (workspace-based).
- **Managed Identity**: Azure-managed credential for a resource (system-assigned: tied to resource lifetime; user-assigned: reusable). No secrets in code.
- **Front Door** (global, L7, CDN+WAF, anycast) vs **Application Gateway** (regional L7 LB + WAF) vs **Load Balancer** (L4) vs **Traffic Manager** (DNS-based).

### 3.1 [Easy] App Service vs Functions vs Container Apps vs AKS — when each?
**Answer**: App Service for classic web apps/APIs with minimal ops; Functions for event-driven short tasks; Container Apps for microservices/containers without managing K8s; AKS when you need full K8s control (custom operators, networking, many teams).
**Explanation**: Choose the least operational burden that meets requirements. AKS brings upgrades, node pools, networking, security responsibility.
**Code/config**
```bash
az webapp create -g rg -p plan -n myapi --runtime "DOTNETCORE:8.0"
az containerapp create -n api -g rg --environment env --image myacr.azurecr.io/myapi:1 --target-port 8080 --ingress external
```

### 3.2 [Follow-up] How do you connect an app to Azure SQL and Key Vault without secrets?
**Answer**: Enable Managed Identity, grant it access (Key Vault RBAC role "Key Vault Secrets User"; SQL contained user for the identity), and use `DefaultAzureCredential`.
**Explanation**: `DefaultAzureCredential` chains env, workload identity, managed identity, Visual Studio/CLI credentials so same code runs locally and in Azure. For SQL, use Microsoft Entra auth in the connection string.
**Code/config**
```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var client = new SecretClient(new Uri("https://myvault.vault.azure.net/"), new DefaultAzureCredential());
KeyVaultSecret s = await client.GetSecretAsync("Sql-Password");

// or load as configuration
builder.Configuration.AddAzureKeyVault(new Uri("https://myvault.vault.azure.net/"), new DefaultAzureCredential());
```
```sql
CREATE USER [myapi-identity] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [myapi-identity];
ALTER ROLE db_datawriter ADD MEMBER [myapi-identity];
```
```
Server=tcp:myserver.database.windows.net;Database=App;Authentication=Active Directory Default;
```

### 3.3 [Medium] Service Bus: how do you handle poison messages and duplicates?
**Answer**: PeekLock mode; if processing fails, abandon → redelivery until `MaxDeliveryCount` then the message moves to the dead-letter queue. Delivery is at-least-once, so handlers must be idempotent (or use duplicate detection / an inbox table).
**Explanation**: Complete the message only after a successful DB commit. For DB + message atomicity use the Outbox pattern.
**Code/config**
```csharp
await using var client = new ServiceBusClient("ns.servicebus.windows.net", new DefaultAzureCredential());
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions { MaxConcurrentCalls = 8, AutoCompleteMessages = false });
processor.ProcessMessageAsync += async args =>
{
    await HandleAsync(args.Message.Body.ToString());
    await args.CompleteMessageAsync(args.Message);
};
processor.ProcessErrorAsync += args => { Console.Error.WriteLine(args.Exception); return Task.CompletedTask; };
await processor.StartProcessingAsync();
```

### 3.4 [Hard] Design global high availability for a .NET API + Azure SQL.
**Answer**: Front Door (WAF, health probes, failover) → App Services/Container Apps in two regions (or zone-redundant in one) → Azure SQL failover group (read-write listener endpoint) → Blob with GRS/RA-GRS; secrets in per-region Key Vault; telemetry to Log Analytics.
**Explanation**: State the RTO/RPO. Zone redundancy handles datacenter failure; multi-region handles region failure. Failover groups give an auto-failover listener so connection strings don't change. Tradeoffs: cost, async replication data loss (RPO > 0), config drift.
**Code/config**
```bash
az sql failover-group create -n fog -s primary-srv -g rg --partner-server secondary-srv --add-db App --failover-policy Automatic --grace-period 1
```

### 3.5 [Tricky] Your App Service slot swap caused errors / DB connections fail after swap. Why?
**Answer**: Slot-specific ("sticky") settings weren't marked, so production pointed at staging config (or vice versa); also cold instances weren't warmed up and schema changes weren't backward-compatible.
**Explanation**: Mark connection strings/app settings as slot settings; use swap with preview and `applicationInitialization` warmup. Migrations must be expand/contract so both versions work against the same DB during swap and for rollback.
**Code/config**
```bash
az webapp deployment slot swap -g rg -n myapi --slot staging --target-slot production
az webapp config appsettings set -g rg -n myapi --slot staging --slot-settings ASPNETCORE_ENVIRONMENT=Staging
```

**Interview traps**
- Confusing Front Door / App Gateway / Traffic Manager / Load Balancer.
- Claiming Service Bus is exactly-once.
- Storing connection secrets in app settings when MI is possible.
- Forgetting SAS tokens are bearer credentials; prefer user-delegation SAS / RBAC.
- Managed identity token cache and propagation delays after role assignment (minutes).

**Cheat sheet**
- MI + DefaultAzureCredential + RBAC = no secrets.
- Service Bus: queue vs topic; DLQ; idempotency; outbox.
- Front Door global L7; App Gateway regional L7; LB L4; TM DNS.
- Azure SQL: failover groups for DR, elastic pools for many DBs, serverless for spiky/dev.
- Slots: sticky settings, warmup, backward-compatible schema.

---

## Topic 4: CI/CD (GitHub Actions & Azure DevOps)

**Study notes**
- CI: build + test on every push/PR; CD: automated delivery/deployment of the *same* build artifact across environments (build once, deploy many).
- GitHub Actions: workflows in `.github/workflows/*.yml`; jobs run on runners; `environments` support required reviewers (approvals), secrets and variables scoped per environment; `needs` for dependencies; OIDC (`azure/login` with `id-token: write`) avoids stored cloud secrets.
- Azure DevOps: `azure-pipelines.yml`; stages → jobs → steps; `environment:` in a deployment job gives approvals/checks; variable groups, Key Vault-linked groups, service connections (prefer workload identity federation).
- Cache NuGet, run tests with `--no-build`, publish test results & coverage, scan dependencies (`dotnet list package --vulnerable`).

### 4.1 [Easy] Write a GitHub Actions CI workflow for a .NET solution.
**Answer**: Checkout, setup-dotnet, restore, build, test, upload artifact.
**Explanation**: `dotnet test --no-build` reuses the build. Use `concurrency` to cancel superseded runs.
**Code/config**
```yaml
name: ci
on:
  push: { branches: [main] }
  pull_request:
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '8.0.x', cache: true, cache-dependency-path: '**/packages.lock.json' }
      - run: dotnet restore
      - run: dotnet build -c Release --no-restore
      - run: dotnet test -c Release --no-build --logger trx --collect:"XPlat Code Coverage"
      - run: dotnet publish src/MyApi -c Release -o out --no-build
      - uses: actions/upload-artifact@v4
        with: { name: api, path: out }
```
(`cache: true` requires a lock file, i.e. `RestorePackagesWithLockFile`; otherwise omit cache.)

### 4.2 [Follow-up] Add deployment to Azure with environments, approvals, and no stored secrets.
**Answer**: Use a deploy job bound to `environment: production` (configure required reviewers in GitHub), authenticate via OIDC federated credential.
**Explanation**: Federated credentials trust the GitHub repo/environment subject; no client secret to rotate.
**Code/config**
```yaml
  deploy-prod:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/download-artifact@v4
        with: { name: api, path: out }
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/webapps-deploy@v3
        with: { app-name: myapi, package: out }
```

### 4.3 [Medium] Equivalent Azure DevOps multi-stage YAML pipeline.
**Answer**: Build stage publishes artifact; deploy stage uses a `deployment` job with `environment` (approvals configured on the environment).
**Explanation**: Approvals/checks live on the Environment resource, not in YAML. `DotNetCoreCLI@2` is the built-in task.
**Code/config**
```yaml
trigger: [main]
pool: { vmImage: ubuntu-latest }
stages:
- stage: Build
  jobs:
  - job: build
    steps:
    - task: UseDotNet@2
      inputs: { packageType: sdk, version: 8.0.x }
    - task: DotNetCoreCLI@2
      inputs: { command: test, projects: '**/*Tests.csproj', arguments: '-c Release' }
    - task: DotNetCoreCLI@2
      inputs: { command: publish, publishWebProjects: true, arguments: '-c Release -o $(Build.ArtifactStagingDirectory)', zipAfterPublish: true }
    - publish: $(Build.ArtifactStagingDirectory)
      artifact: drop
- stage: DeployProd
  dependsOn: Build
  jobs:
  - deployment: deploy
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureWebApp@1
            inputs: { azureSubscription: 'svc-conn', appName: myapi, package: '$(Pipeline.Workspace)/drop/**/*.zip' }
```

### 4.4 [Hard] How do you handle EF Core migrations and secrets in the pipeline?
**Answer**: Generate an idempotent SQL script (`dotnet ef migrations script --idempotent`) or a migration bundle (`dotnet ef migrations bundle`) as a build artifact; apply it in a controlled deploy step with a privileged identity separate from the app's runtime identity. Avoid `Database.Migrate()` at startup in multi-instance production.
**Explanation**: Reviewable scripts allow DBA approval; bundles are self-contained executables. Runtime identity should have least privilege (no DDL). Use expand/contract for zero-downtime.
**Code/config**
```bash
dotnet ef migrations script --idempotent -p src/Data -s src/MyApi -o migrate.sql
dotnet ef migrations bundle -p src/Data -s src/MyApi -o efbundle --self-contained
./efbundle --connection "$CONN"
```

### 4.5 [Tricky] A secret leaked into pipeline logs / a PR from a fork ran with secrets. What happened and how to prevent?
**Answer**: Secrets are masked only when the exact value is printed; transformed (base64) values leak. In GitHub, workflows triggered by fork PRs don't receive secrets (except `pull_request_target`, which is dangerous if it checks out untrusted code).
**Explanation**: Prevent: OIDC instead of long-lived secrets, least-privilege tokens (`permissions:`), pin third-party actions to commit SHAs, environment-scoped secrets with approvals, never echo secrets, rotate if leaked.
**Code/config**
```yaml
permissions:
  contents: read
steps:
  - uses: actions/checkout@<full-commit-sha>
```

**Interview traps**
- Rebuilding per environment instead of promoting one artifact.
- `Database.Migrate()` racing across instances.
- Approvals defined in YAML (ADO: on Environment; GitHub: environment protection rules).
- `pull_request_target` + checkout of PR head.
- Long-lived service principal secrets instead of federated credentials.

**Cheat sheet**
- Build once, promote artifact; environments for approvals/secrets.
- OIDC/workload identity federation > stored secrets.
- Migrations: idempotent script or bundle, separate identity.
- `needs`/`dependsOn` ordering; cache NuGet; publish test results.

---

## Topic 5: Git Workflows

**Study notes**
- Branching models: **trunk-based** (short-lived branches, feature flags; best for CI/CD), **GitHub Flow** (feature branch → PR → main), **GitFlow** (develop/release/hotfix; heavier, suits scheduled releases).
- **Merge** preserves history with a merge commit (or fast-forward); **rebase** rewrites commits onto a new base for linear history; **squash merge** collapses PR into one commit.
- Rule: never rebase shared/public history. Use `--force-with-lease` (not `--force`) after rebasing your own branch.
- **cherry-pick** applies a specific commit elsewhere (hotfix backports); `-x` records origin.
- **reset** moves branch pointer (`--soft/--mixed/--hard`); **revert** creates an inverse commit (safe on shared branches). **reflog** recovers lost commits.

### 5.1 [Easy] Merge vs rebase?
**Answer**: Merge joins histories with a merge commit; rebase replays your commits atop another branch for linear history.
**Explanation**: Rebase gives cleaner history but rewrites SHAs; merge is non-destructive and keeps true timeline.
**Code/config**
```bash
git checkout feature && git rebase main
git checkout main && git merge --no-ff feature
```

### 5.2 [Follow-up] How do you resolve a merge conflict?
**Answer**: Run merge/rebase, open conflicted files (`<<<<<<<`, `=======`, `>>>>>>>`), choose/combine, `git add`, then `git merge --continue` / `git rebase --continue`. Build and test before finishing.
**Explanation**: Conflicts in `.csproj`, EF `ModelSnapshot.cs`, and migrations need care — on snapshot conflicts, usually remove your migration, merge, and re-add it so the snapshot is regenerated.
**Code/config**
```bash
git status
git mergetool
git rebase --continue   # or: git rebase --abort
```

### 5.3 [Medium] Hotfix a production bug already merged to main while main contains unreleased work. How?
**Answer**: Branch from the release tag/commit, fix, deploy, then cherry-pick the fix commit to main.
**Explanation**: Keeps unreleased work out of the hotfix. Tag releases to make this possible.
**Code/config**
```bash
git checkout -b hotfix/1.4.1 v1.4.0
# fix, commit
git tag v1.4.1
git checkout main && git cherry-pick -x <sha>
```

### 5.4 [Hard] You pushed a bad commit/secret to main. Remediate.
**Answer**: Bad logic: `git revert <sha>` (add `-m 1` for merge commits). Secret: rotate it immediately (history rewrite is secondary), then purge with `git filter-repo` or BFG and force-push, coordinating with the team; enable secret scanning/push protection.
**Explanation**: Anyone who cloned already has the secret, so rotation is mandatory.
**Code/config**
```bash
git revert -m 1 <merge-sha>
git reflog   # recover lost work: git reset --hard HEAD@{3}
```

### 5.5 [Tricky] After `git rebase -i` you "lost" commits, and a teammate's pull now has duplicates. Why?
**Answer**: Dropped/squashed commits remain in the reflog (recoverable ~90 days by default for reachable-from-reflog entries). Duplicates occur because you rebased commits teammates had already pulled; their old commits and your rewritten ones both exist.
**Explanation**: Avoid rebasing published branches; if necessary coordinate, and use `git pull --rebase` to reapply local work.
**Code/config**
```bash
git reflog
git branch rescue <sha>
git push --force-with-lease
```

**Interview traps**
- "Rebase is always better" / "never rebase" — both dogmatic.
- `--force` overwriting colleagues' work.
- Confusing `revert` and `reset`.
- Squash merge makes later cherry-pick/ancestry tracking of the branch harder.
- Resolving EF snapshot conflicts by hand-editing blindly.

**Cheat sheet**
- Trunk-based + short branches + flags for CD.
- Rebase local, merge shared; `--force-with-lease`.
- Hotfix: branch from tag, cherry-pick -x back.
- reflog = safety net; revert for shared history.

---

## Topic 6: Infrastructure as Code (Bicep / Terraform)

**Study notes**
- IaC = infrastructure defined in version-controlled, reviewable, repeatable code. Declarative (desired state) vs imperative scripts.
- **Bicep**: Azure-native DSL that compiles to ARM JSON; no state file (Azure Resource Manager is the source of truth); `what-if` previews changes; modules for reuse; deployment modes Incremental (default) / Complete.
- **Terraform**: multi-cloud (HCL); uses **state** (store remotely, e.g., Azure Storage with locking); `plan` → `apply`; providers (`azurerm`); `import` to adopt existing resources.
- Idempotence: re-running yields same result. Drift: real infra diverges from code.

### 6.1 [Easy] What is IaC and why use it?
**Answer**: Defining infra in code for repeatability, review, versioning, consistency between environments, and disaster recovery.
**Explanation**: Eliminates "snowflake" environments and click-ops errors.
**Code/config**
```bash
az deployment group what-if -g rg -f main.bicep -p env=dev
az deployment group create -g rg -f main.bicep -p env=dev
```

### 6.2 [Follow-up] Bicep: App Service + system-assigned identity.
**Answer**: See below.
**Explanation**: Reference outputs (`principalId`) to grant role assignments. Parameters make it environment-agnostic.
**Code/config**
```bicep
param location string = resourceGroup().location
param env string

resource plan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'plan-${env}'
  location: location
  sku: { name: 'P1v3' }
  kind: 'linux'
  properties: { reserved: true }
}

resource app 'Microsoft.Web/sites@2023-01-01' = {
  name: 'myapi-${env}-${uniqueString(resourceGroup().id)}'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: plan.id
    httpsOnly: true
    siteConfig: { linuxFxVersion: 'DOTNETCORE|8.0' }
  }
}

output principalId string = app.identity.principalId
```
(Verify API versions against current docs before use.)

### 6.3 [Medium] Terraform equivalent and the role of state.
**Answer**: Resources in HCL; state maps config to real resources and must be stored remotely with locking, never committed.
**Explanation**: State can contain secrets — restrict access. Parallel applies without locking corrupt state.
**Code/config**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "rg-tfstate"
    storage_account_name = "sttfstate123"
    container_name       = "tfstate"
    key                  = "myapi.tfstate"
  }
}
resource "azurerm_resource_group" "rg" {
  name     = "rg-myapi"
  location = "westeurope"
}
```
```bash
terraform init && terraform plan -out tfplan && terraform apply tfplan
```

### 6.4 [Hard] Bicep vs Terraform — how choose? How do you manage drift and secrets?
**Answer**: Bicep for Azure-only shops (day-0 support for new resource types, no state to manage); Terraform for multi-cloud/existing expertise/ecosystem. Detect drift via `plan`/`what-if` on schedule in CI; avoid secrets in params — use Key Vault references (`getSecret` in Bicep module params) or `@secure()`.
**Explanation**: Restrict manual portal changes (Azure Policy, RBAC), and run IaC deployments through pipelines with PR review of plan output.
**Code/config**
```bicep
@secure()
param sqlAdminPassword string
```

### 6.5 [Tricky] `terraform apply` wants to destroy and recreate your SQL database. Why, and what do you do?
**Answer**: Changing a "ForceNew" attribute (e.g., name/location, some SKU/collation settings) forces replacement. Read the plan carefully; use `lifecycle { prevent_destroy = true }`, `ignore_changes`, or `terraform state mv` for renames/refactors.
**Explanation**: The plan output marks `-/+` (destroy then create) and "forces replacement". Never auto-approve production plans without review.
**Code/config**
```hcl
lifecycle { prevent_destroy = true }
```

**Interview traps**
- Believing Bicep has a state file.
- Committing `.tfstate` or secrets.
- Complete mode in Bicep deleting resources not in template.
- Auto-approving applies.

**Cheat sheet**
- Bicep: no state, `what-if`, modules. Terraform: remote state, plan/apply.
- Secrets via Key Vault/@secure; PR-review the plan.
- Policy + RBAC to prevent drift.

---

## Topic 7: Deployment Strategies

**Study notes**
- **Rolling**: replace instances gradually (K8s default). Cheap; mixed versions live together.
- **Blue-green**: two full environments; switch traffic atomically (App Service slots, DNS/Front Door). Fast rollback; double cost; DB must be compatible with both.
- **Canary**: route small % of traffic to new version, watch metrics, then expand (Front Door weights, Container Apps revisions traffic split, Argo Rollouts/Flagger).
- **Feature flags**: decouple deploy from release (`Microsoft.FeatureManagement`, Azure App Configuration, LaunchDarkly). Remove stale flags.
- **DB migrations**: expand/contract (parallel change): (1) add nullable column/new table (expand), (2) deploy code writing both/reading new, (3) backfill, (4) deploy code using only new, (5) drop old (contract).

### 7.1 [Easy] Blue-green vs canary vs rolling?
**Answer**: Rolling = gradual instance replacement; blue-green = parallel environment + switch; canary = small traffic share first with metric gating.
**Explanation**: Canary limits blast radius; blue-green optimizes rollback speed; rolling optimizes resource efficiency.
**Code/config**
```bash
az containerapp ingress traffic set -n api -g rg --revision-weight api--v1=90 api--v2=10
```

### 7.2 [Follow-up] Implement a feature flag in ASP.NET Core.
**Answer**: Use `Microsoft.FeatureManagement.AspNetCore`.
**Explanation**: Flags can be backed by appsettings or Azure App Configuration for runtime toggling without redeploy.
**Code/config**
```csharp
builder.Services.AddFeatureManagement();   // reads "FeatureManagement" section
// appsettings.json: { "FeatureManagement": { "NewCheckout": false } }

public class CheckoutController(IFeatureManager features) : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Pay() =>
        await features.IsEnabledAsync("NewCheckout") ? NewFlow() : OldFlow();
}
```

### 7.3 [Medium] Rename a column with zero downtime on EF Core + SQL Server.
**Answer**: Don't rename directly. Add new column, dual-write (or trigger), backfill in batches, switch reads, then drop the old column in a later release.
**Explanation**: A direct rename breaks the still-running old version (rolling/blue-green overlap, and rollback). EF's `RenameColumn` runs `sp_rename`, which is instant but not backward-compatible.
**Code/config**
```sql
ALTER TABLE Customers ADD FullName nvarchar(200) NULL;           -- expand
UPDATE TOP (5000) Customers SET FullName = Name WHERE FullName IS NULL;  -- loop batches
-- later release:
ALTER TABLE Customers DROP COLUMN Name;                          -- contract
```

### 7.4 [Hard] Define a canary promotion policy and automatic rollback.
**Answer**: Route e.g. 5% → 25% → 100%, gating each step on SLIs (error rate, p95 latency, saturation) compared with baseline for a soak period; auto-rollback if thresholds breached.
**Explanation**: Needs good telemetry tagged by version (`cloud_RoleInstance`/custom dimension), sufficient traffic for statistical confidence, and backward-compatible data changes. Tools: Argo Rollouts, Flagger, Azure Pipelines gates/Azure Monitor alerts.
**Code/config**
```yaml
# Argo Rollouts (excerpt)
strategy:
  canary:
    steps:
    - setWeight: 5
    - pause: { duration: 10m }
    - setWeight: 25
    - pause: { duration: 10m }
```

### 7.5 [Tricky] Blue-green with a shared DB: what goes wrong?
**Answer**: Green's schema change breaks blue (so rollback fails); long-running transactions/queues consumed by both; caches/sessions (use distributed session); sticky data like in-flight messages processed by different versions.
**Explanation**: Blue-green only isolates compute, not state. Contract-phase changes must wait until rollback is no longer needed.
**Code/config**
```text
Release N:   expand schema  -> deploy green -> swap
Release N+1: stop writing old column -> after bake time: contract (drop)
```

**Interview traps**
- "Blue-green solves DB migrations" — it doesn't.
- Feature flags without cleanup/ownership.
- Canary without metrics or enough traffic.
- Destructive migrations in same release as code.

**Cheat sheet**
- Rolling cheap, blue-green fast rollback, canary low risk.
- Expand/contract for schema; flags decouple deploy/release.
- Rollback plan = backward-compatible data + previous artifact kept.

---

## Topic 8: Monitoring & Alerting

**Study notes**
- Pillars: **logs, metrics, traces** (OpenTelemetry unifies). Observability = ability to ask new questions of the system from its outputs.
- **Application Insights**: requests, dependencies (SQL, HTTP), exceptions, traces, custom events/metrics, Live Metrics, Application Map, availability tests, workspace-based (KQL in Log Analytics). Sampling reduces cost.
- **Prometheus** (pull-based metrics, PromQL) + **Grafana** (dashboards); .NET: `OpenTelemetry` + Prometheus exporter, or `prometheus-net`.
- **SLI** = measured indicator (e.g., % requests < 300 ms and non-5xx); **SLO** = target (99.9% over 30 days); **SLA** = contractual promise with penalties. **Error budget** = 1 − SLO (99.9% ≈ 43.2 min/30 days).
- Alert on symptoms (user impact), not every cause; avoid alert fatigue; golden signals: latency, traffic, errors, saturation.

### 8.1 [Easy] How do you add Application Insights to ASP.NET Core?
**Answer**: Use the Azure Monitor OpenTelemetry distro (current recommended) or the classic `Microsoft.ApplicationInsights.AspNetCore` SDK, with the connection string.
**Explanation**: Connection string (not just instrumentation key) is the supported config.
**Code/config**
```csharp
// Azure.Monitor.OpenTelemetry.AspNetCore
builder.Services.AddOpenTelemetry().UseAzureMonitor();   // reads APPLICATIONINSIGHTS_CONNECTION_STRING

// classic
builder.Services.AddApplicationInsightsTelemetry();
```

### 8.2 [Follow-up] Write a KQL query for failed requests and slow dependencies.
**Answer**: Query `requests` and `dependencies` tables (workspace-based: `AppRequests`, `AppDependencies`).
**Explanation**: Percentiles beat averages. Correlate using `operation_Id`.
**Code/config**
```kusto
requests
| where timestamp > ago(1h)
| summarize total=count(), failed=countif(success == false), p95=percentile(duration, 95) by bin(timestamp, 5m), name
| extend failRate = todouble(failed) / total
| order by timestamp desc

dependencies
| where type == "SQL" and duration > 1000
| project timestamp, name, duration, operation_Id
```

### 8.3 [Medium] Define SLI/SLO for an order API and an alert strategy.
**Answer**: SLI: ratio of non-5xx requests with latency < 500 ms; SLO: 99.9% over 30 days. Alert on error-budget **burn rate** (e.g., 14.4× over 1h and 5m windows → page; 6× over 6h → ticket).
**Explanation**: Burn-rate alerts are multi-window to be both fast and low-noise. Error budget guides release velocity (freeze when exhausted).
**Code/config**
```promql
sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

### 8.4 [Hard] Production latency spiked — your investigation approach.
**Answer**: Scope (which endpoints/regions/versions), check recent deploys/flags, golden signals, App Map for slow dependency, trace a slow operation end-to-end, then DB: `sys.dm_exec_query_stats`, Query Store, blocking, missing indexes, connection pool exhaustion, thread-pool starvation (sync-over-async), GC pauses.
**Explanation**: Mitigate first (rollback/scale/disable flag), root-cause after. Tools: `dotnet-counters`, `dotnet-trace`, `dotnet-dump`.
**Code/config**
```bash
dotnet-counters monitor -p <pid> System.Runtime Microsoft.AspNetCore.Hosting
```

### 8.5 [Tricky] Why do your dashboards show averages fine while users complain?
**Answer**: Averages hide tail latency; sampling/aggregation may drop failures; metrics with high-cardinality labels are dropped or costly; and synthetic checks may not mimic real flows.
**Explanation**: Use p95/p99 histograms, don't sample out errors (adaptive sampling retains failures preferentially in some configs, verify), use RUM/availability tests, and avoid user-ID labels in Prometheus (cardinality explosion).
**Code/config**
```promql
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

**Interview traps**
- Mixing up SLA/SLO/SLI.
- Alerting on CPU only; paging on non-actionable alerts.
- Logging PII/secrets; unbounded log volume cost.
- High-cardinality metric labels.

**Cheat sheet**
- Logs/metrics/traces; golden signals; percentiles not averages.
- SLO − error budget − burn-rate alerts.
- App Insights: KQL, App Map, correlation via operation_Id.
- Mitigate first, root-cause second.

---

## Topic 9: Azure Functions

**Study notes**
- Hosting plans: **Consumption** (scale to zero, pay per execution), **Flex Consumption**, **Premium** (pre-warmed, VNet), **Dedicated** (App Service plan).
- Models: **isolated worker** (recommended; out-of-process, own DI/Program.cs, supported beyond .NET 8) vs in-process (retiring; support ends Nov 2026).
- Triggers: HTTP, Timer, Queue/Service Bus, Blob, Event Hub, Event Grid, Cosmos DB. Bindings reduce boilerplate (input/output).
- **Durable Functions**: stateful orchestrations in code — orchestrator (deterministic, replayed), activities (do work), client; patterns: function chaining, fan-out/fan-in, async HTTP API, monitor, human interaction, eternal entities.
- Cold starts affect Consumption; mitigate with Premium/pre-warmed or Flex "always ready".

### 9.1 [Easy] Write an HTTP-triggered function (isolated worker).
**Answer**: See code.
**Explanation**: DI via `HostBuilder`; `[Function]` attribute names the function.
**Code/config**
```csharp
public class Hello(ILogger<Hello> log)
{
    [Function("Hello")]
    public IActionResult Run([HttpTrigger(AuthorizationLevel.Function, "get")] HttpRequest req)
    {
        log.LogInformation("called");
        return new OkObjectResult("hi");
    }
}
// Program.cs
var host = new HostBuilder().ConfigureFunctionsWebApplication().Build();
host.Run();
```

### 9.2 [Follow-up] Service Bus-triggered function and retry/poison behavior?
**Answer**: Runtime locks the message, invokes the function; success completes it; exception abandons it; after `maxDeliveryCount` it goes to DLQ.
**Explanation**: Handler must be idempotent. Configure `maxConcurrentCalls` in host.json.
**Code/config**
```csharp
[Function("Process")]
public async Task Run([ServiceBusTrigger("orders", Connection = "SbConn")] ServiceBusReceivedMessage msg)
{
    await handler.HandleAsync(msg.Body.ToString());
}
```
`Connection` with identity-based setting: `SbConn__fullyQualifiedNamespace`.

### 9.3 [Medium] Implement fan-out/fan-in with Durable Functions.
**Answer**: Orchestrator starts many activity tasks and awaits `Task.WhenAll`.
**Explanation**: Orchestrators are replayed, so code must be deterministic: no `DateTime.UtcNow`, `Guid.NewGuid()`, direct I/O, or `Task.Delay` — use `context.CurrentUtcDateTime`, `context.CreateTimer`, activities. Use `context.CreateReplaySafeLogger`.
**Code/config**
```csharp
[Function(nameof(Orchestrate))]
public static async Task<int[]> Orchestrate([OrchestrationTrigger] TaskOrchestrationContext ctx)
{
    var items = await ctx.CallActivityAsync<string[]>("GetItems");
    var tasks = items.Select(i => ctx.CallActivityAsync<int>("Process", i));
    return await Task.WhenAll(tasks);
}
```

### 9.4 [Hard] Why does a Function on Consumption exhaust connections / hit SNAT limits, and how do you fix it?
**Answer**: Creating `new HttpClient()` or SQL connections per invocation exhausts sockets/ports. Use `IHttpClientFactory`/a static client, reuse clients (`ServiceBusClient`, `BlobServiceClient` singletons via DI), and pooled SQL connections.
**Explanation**: Instances scale out, so total connections = instances × per-instance usage; downstream DB limits (e.g., SQL max connections) need `maxConcurrentCalls` throttling.
**Code/config**
```csharp
builder.Services.AddHttpClient();
builder.Services.AddSingleton(_ => new ServiceBusClient(ns, new DefaultAzureCredential()));
```

### 9.5 [Tricky] Your orchestrator "runs twice" / behaves nondeterministically. Why?
**Answer**: Replay: the orchestrator function re-executes from the start each time it resumes, using history to skip completed work. Side effects in orchestrator code (logging, I/O, random) occur multiple times or break determinism.
**Explanation**: Put side effects in activities (at-least-once, so make idempotent). Orchestrator code must be deterministic or replay will diverge from history and fail.
**Code/config**
```csharp
var now = ctx.CurrentUtcDateTime;                // not DateTime.UtcNow
await ctx.CreateTimer(now.AddMinutes(5), CancellationToken.None);
```

**Interview traps**
- Claiming in-process model is the future (isolated is).
- `Thread.Sleep`/`DateTime.Now`/HTTP calls inside orchestrators.
- New HttpClient per call.
- Long-running work on Consumption (default timeout 5 min, max 10; Durable or Premium for longer).

**Cheat sheet**
- Isolated worker; DI; singleton clients.
- Triggers/bindings; MI via `Conn__fullyQualifiedNamespace`.
- Durable: deterministic orchestrator, side effects in activities.
- Idempotency always (at-least-once).

---

## Topic 10: Cloud Cost & Performance Basics

**Study notes**
- Cost levers: right-size SKUs, autoscale (and scale in), scale-to-zero (Functions/Container Apps), Reserved Instances/Savings Plans (1–3 yr commit), Spot VMs for interruptible work, Azure Hybrid Benefit (reuse Windows/SQL licenses), storage tiering/lifecycle policies, dev/test shutdown schedules, delete orphaned disks/IPs, log sampling/retention limits.
- Governance: tags, budgets + alerts in Cost Management, Azure Advisor, Azure Policy.
- Performance: caching (Azure Cache for Redis, `IMemoryCache`, output caching/response caching, CDN/Front Door), async I/O, connection pooling, DB indexing/Query Store, pagination, compression, co-locating resources in the same region, `AsNoTracking` for reads, avoid chatty calls.
- Scale up (vertical) vs scale out (horizontal; requires stateless app).

### 10.1 [Easy] Scale up vs scale out?
**Answer**: Up = bigger instance; out = more instances. Out needs statelessness and gives resilience.
**Explanation**: Vertical has a ceiling and usually downtime; horizontal is elastic.
**Code/config**
```bash
az monitor autoscale create -g rg --resource plan --resource-type Microsoft.Web/serverfarms --min-count 2 --max-count 10 --count 2
```

### 10.2 [Follow-up] Where would you add caching in a .NET API?
**Answer**: Browser/CDN (static, cacheable GETs), output caching for endpoints, in-memory for hot small data on one instance, distributed Redis for shared cache; always define expiry and invalidation.
**Explanation**: In-memory caches diverge across instances; distributed cache adds network latency but consistency. Beware cache stampede (use locking/`GetOrCreateAsync` with `HybridCache` in .NET 9).
**Code/config**
```csharp
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = cfg["Redis"]);
builder.Services.AddOutputCache();
app.UseOutputCache();
app.MapGet("/products", GetProducts).CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)));
```

### 10.3 [Medium] Your Azure bill doubled; how do you investigate?
**Answer**: Cost Management → group by resource/tag/service and compare periods; check autoscale max, log ingestion volume (App Insights/Log Analytics), egress, new SKUs, orphaned resources, unexpected DB tier/DTUs.
**Explanation**: Common culprits: verbose logging/high-ingestion telemetry, premium tiers left running, cross-region data transfer, always-on dev environments.
**Code/config**
```kusto
Usage
| where TimeGenerated > ago(30d)
| summarize GB = sum(Quantity) / 1000 by DataType
| order by GB desc
```

### 10.4 [Hard] API is slow only under load although CPU is low. What do you check?
**Answer**: Thread-pool starvation from sync-over-async (`.Result`, `.Wait()`), DB connection pool exhaustion (default max 100), lock contention/blocking in SQL, downstream rate limiting, SNAT port exhaustion, GC (Server vs Workstation, LOH allocations).
**Explanation**: Low CPU + high latency = waiting. Use dotnet-counters (`ThreadPool Queue Length`), SQL wait stats, App Insights dependency durations.
**Code/config**
```csharp
// Bad
var data = repo.GetAsync(id).Result;
// Good
var data = await repo.GetAsync(id, ct);
```

### 10.5 [Tricky] Is autoscaling always the answer to performance problems?
**Answer**: No — it scales the stateless tier but can overwhelm a shared bottleneck (single SQL DB, connection limits), amplify cost, and lag behind spikes.
**Explanation**: Fix the bottleneck (indexes, caching, queue-based load leveling), set max replica limits, and use rate limiting/back-pressure (`Microsoft.AspNetCore.RateLimiting`). Retry storms worsen outages: use exponential backoff + jitter (Polly/`Microsoft.Extensions.Http.Resilience`).
**Code/config**
```csharp
builder.Services.AddRateLimiter(o => o.AddFixedWindowLimiter("api", l => { l.PermitLimit = 100; l.Window = TimeSpan.FromMinutes(1); }));
app.UseRateLimiter();
```

**Interview traps**
- "Just scale out" with a single DB bottleneck.
- Ignoring egress and logging costs.
- Caching without invalidation strategy.
- Reservation purchases without usage baseline.

**Cheat sheet**
- Cost: right-size, autoscale in, reservations, spot, tiering, tags/budgets.
- Perf: async, cache, index, pool, same region, paginate.
- Low CPU + slow ⇒ blocking/pools/dependencies.

---

## Topic 11: Serverless vs Containers vs VMs

**Study notes**
| Aspect | VMs (IaaS) | Containers (AKS/Container Apps/App Service containers) | Serverless (Functions) |
|---|---|---|---|
| Control | Full OS | App + runtime | Code only |
| Ops burden | Highest (patching) | Medium (AKS high, ACA low) | Lowest |
| Startup | Minutes | Seconds | ms–s (cold start) |
| Scaling | Slow, manual/VMSS | Fast, HPA/KEDA | Automatic, to zero |
| Cost model | Pay for uptime | Pay for allocated resources | Pay per execution (consumption) |
| Best for | Legacy, custom OS, licensing | Microservices, portable workloads | Event-driven, spiky, glue code |
| Limits | — | Orchestration complexity | Execution time, cold start, vendor coupling |

### 11.1 [Easy] When pick serverless?
**Answer**: Event-driven, intermittent/spiky workloads, quick time-to-market, small glue tasks.
**Explanation**: Poor fit: long-running, steady high-throughput (can be costlier), low-latency strict, heavy local state.
**Code/config**
```text
Blob upload -> Event Grid -> Function (thumbnail) -> Blob
```

### 11.2 [Follow-up] Why choose containers over App Service code deploy?
**Answer**: Custom dependencies/OS libs, portability across clouds/on-prem, identical dev/prod, sidecars, multi-service.
**Explanation**: Tradeoff: you own the image (patching base images, scanning).
**Code/config**
```bash
az webapp config container set -n myapi -g rg --container-image-name myacr.azurecr.io/myapi:1.0
```

### 11.3 [Medium] Monolith migration: what's your recommendation for a legacy .NET Framework app?
**Answer**: Lift-and-shift to VM/App Service (Windows) first, then incremental modernization (strangler fig) to .NET 8 + containers where there's value.
**Explanation**: .NET Framework runs on Windows only (Windows containers or VMs/App Service Windows). Tools: .NET Upgrade Assistant, Azure Migrate. Don't rewrite everything at once.
**Code/config**
```text
Strangler: Front Door/YARP routes /orders -> new service; rest -> legacy monolith
```

### 11.4 [Hard] Design decision: Container Apps vs AKS for 15 microservices.
**Answer**: Container Apps if no need for custom K8s APIs/operators/advanced networking; it gives scale-to-zero, KEDA, Dapr, revisions with low ops. AKS if the platform team needs fine control, service mesh choice, GPU, custom CRDs, or very large scale.
**Explanation**: Weigh team skills, ops capacity, compliance, portability. Start simple; migrate if outgrown.
**Code/config**
```text
Decision inputs: team K8s skill | compliance | scale | cost | portability
```

### 11.5 [Tricky] "Serverless means no servers, so no capacity planning" — true?
**Answer**: Servers exist; you don't manage them. You still plan for concurrency limits, downstream capacity, quotas, cold starts, and cost at scale.
**Explanation**: Unbounded scale-out can overload SQL or run up cost (denial-of-wallet); set `maxConcurrentCalls`/`WEBSITE_MAX_DYNAMIC_APPLICATION_SCALE_OUT`.
**Code/config**
```json
{ "version": "2.0", "extensions": { "serviceBus": { "maxConcurrentCalls": 16 } } }
```

**Interview traps**
- Treating any option as universally best.
- Ignoring operational ownership and cost at steady load.
- Forgetting .NET Framework is Windows-only.

**Cheat sheet**
- VM = control; containers = portability; serverless = least ops/spiky events.
- Start with the simplest managed option satisfying requirements.
- Strangler pattern for legacy.

---

## Topic 12: Twelve-Factor App

**Study notes** (the 12 factors)
1. **Codebase**: one repo per app, many deploys. 2. **Dependencies**: explicit (NuGet, lock files). 3. **Config**: in environment, not code. 4. **Backing services**: attached resources via URL/config (SQL, queues). 5. **Build, release, run**: strictly separate stages. 6. **Processes**: stateless, share-nothing. 7. **Port binding**: app self-exports via port. 8. **Concurrency**: scale out via processes. 9. **Disposability**: fast startup, graceful shutdown. 10. **Dev/prod parity**. 11. **Logs**: treat as event streams (stdout). 12. **Admin processes**: one-off tasks as same codebase (migrations as Job).

### 12.1 [Easy] Which factor says config goes in the environment, and how does .NET support it?
**Answer**: Factor III. .NET's configuration system layers appsettings.json → appsettings.{Env}.json → user secrets (dev) → environment variables → command line.
**Explanation**: Later providers override earlier ones. Keep secrets out of source control.
**Code/config**
```csharp
var builder = WebApplication.CreateBuilder(args); // defaults include env vars
// ENV: ConnectionStrings__Default=...
var cs = builder.Configuration.GetConnectionString("Default");
```

### 12.2 [Follow-up] What makes an ASP.NET Core app "stateless" and why?
**Answer**: No in-process session/file state; sessions, cache, uploads go to Redis/Blob/DB so any instance can serve any request.
**Explanation**: Enables horizontal scaling and safe restarts. Data Protection keys must be shared across instances (Blob/Key Vault/Redis) or auth cookies break.
**Code/config**
```csharp
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(new Uri(blobUri), new DefaultAzureCredential())
    .ProtectKeysWithAzureKeyVault(new Uri(keyUri), new DefaultAzureCredential());
```

### 12.3 [Medium] Disposability and logs in .NET containers.
**Answer**: Handle SIGTERM via host lifetime (`IHostApplicationLifetime`, `BackgroundService` honoring `CancellationToken`); log to stdout (console logger) and let the platform ship logs.
**Explanation**: Don't write log files inside containers. Use structured logging (Serilog/`ILogger` JSON console).
**Code/config**
```csharp
public class Worker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested) { await DoWorkAsync(ct); await Task.Delay(1000, ct); }
    }
}
builder.Logging.AddJsonConsole();
```

### 12.4 [Hard] Where do migrations (admin processes) run in a 12-factor deployment?
**Answer**: As a one-off job (K8s Job/Container Apps Job/pipeline step/EF bundle) using the same release artifact, before or alongside rollout, not inside each web instance.
**Explanation**: Prevents concurrent migration races and gives clear failure handling.
**Code/config**
```yaml
apiVersion: batch/v1
kind: Job
metadata: { name: migrate }
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrate
        image: myacr.azurecr.io/myapi-migrator:1.0.3
```

### 12.5 [Tricky] Which 12-factor principles do microservices commonly violate in practice?
**Answer**: Shared databases across services (backing-service coupling), environment-specific code branches (dev/prod parity), local file state, config baked into images, long startup (disposability).
**Explanation**: 12-factor targets deployable apps; it doesn't cover everything (e.g., observability, security, API-first) — "beyond 12-factor" adds those.
**Code/config**
```text
Anti-pattern: if (env == "Prod") {...} scattered through code -> use config/flags
```

**Interview traps**
- Saying "environment variables for secrets" is the end state (prefer secret managers/MI).
- Forgetting Data Protection keys in scaled-out apps.
- Logging to files in containers.

**Cheat sheet**
- Config in env; stateless; disposable; logs to stdout; one-off admin jobs; parity.
- Build once → release (artifact + config) → run.

---

# Behavioral & Career

## How to use this section
Use **STAR**: **S**ituation (1–2 sentences) → **T**ask (your responsibility) → **A**ction (what *you* did, 60% of the answer) → **R**esult (measurable + lesson). Keep 2–3 minutes. Prepare 6–8 stories that each map to several questions. Quantify (latency 4s → 300ms, query reads 1.2M → 8K, deploy time 2h → 15m). Never invent details; the templates below use placeholders and example talking points you must replace with your true experience.

## B1. Tell me about yourself (.NET developer)
**Guidance**: Present → Past → Future, 90 seconds, job-relevant.
**STAR/template**: "I'm a [X]-year .NET developer working mostly with C#, ASP.NET Core, EF Core and SQL Server. Currently I [own/build] [domain/system], where I [key impact]. Before that I [relevant earlier experience]. I enjoy [performance tuning/API design/cloud], and I'm looking for [scope] — which is why this role fits."
**Talking points**: Moved a service from .NET Framework to .NET 8; tuned EF queries/indexes; introduced CI/CD; Azure exposure.

## B2. Biggest technical challenge
**Guidance**: Pick a problem with ambiguity and measurable outcome.
**Template**: S: [Report endpoint timing out at peak]. T: [I was asked to fix without schema downtime]. A: [Profiled with App Insights/Query Store, found N+1 and missing covering index, rewrote with projection + `AsNoTracking`, added index online, added load test]. R: [p95 6 s → 400 ms; DB CPU −45%].
**Talking points**: Actual execution plan, key lookup elimination, parameter sniffing, pagination.

## B3. Conflict with a teammate
**Guidance**: Show empathy, data-driven resolution, relationship intact. Not blame.
**Template**: S: [Teammate wanted raw SQL everywhere; I preferred EF]. T: [Align on approach]. A: [Listened, benchmarked both on a representative query, agreed: EF for CRUD, Dapper/SQL for reporting hot paths, documented in ADR]. R: [Faster delivery, consistent guideline, good working relationship].

## B4. A mistake / failure
**Guidance**: Real mistake, own it, show prevention. Avoid "my weakness is perfectionism" style evasions.
**Template**: S: [Deployed a migration that locked a large table]. T: [Restore service]. A: [Rolled back, communicated, killed blocking session, re-ran in batches off-peak]. R: [Recovered in 20 min; added migration review checklist, staging with prod-size data, pipeline gate].
**Talking points**: Non-null column default on big table, `NOLOCK` misuse, forgetting `await`.

## B5. Mentoring junior developers
**Guidance**: Show teaching approach, not just "helped".
**Template**: S: [Junior struggled with async/EF]. A: [Pair programming, small tasks, code-review comments with "why", brown-bag on `IQueryable` vs `IEnumerable`]. R: [They shipped their first feature independently in 6 weeks; PR rework dropped].

## B6. Handling tight deadlines
**Guidance**: Prioritization, scope negotiation, communication, quality guardrails.
**Template**: A: [Split into must-have/should-have, agreed MVP with PM, feature-flagged the rest, kept tests on critical path, daily status]. R: [Delivered on time; remaining scope in next sprint; no Sev-1 bugs].

## B7. Disagreeing with an architect / senior decision
**Guidance**: Disagree and commit; bring evidence; respect authority.
**Template**: S: [Architect proposed microservices for a small domain]. A: [Wrote a short tradeoff doc—team size, deployment cost, data consistency—proposed modular monolith with clear boundaries; presented in design review; accepted final decision]. R: [Compromise: modular monolith with extraction criteria].

## B8. Working with legacy code
**Guidance**: Pragmatic, incremental, test-first around changes.
**Template**: A: [Added characterization tests before changes, used strangler pattern, upgraded .NET Framework 4.x libs to .NET Standard/.NET 8 gradually, replaced ADO.NET with EF/Dapper where safe]. R: [Reduced incident rate; enabled CI].
**Talking points**: Michael Feathers-style seams, Upgrade Assistant, dead code removal.

## B9. Estimation
**Guidance**: Show method: decompose, include unknowns, buffer, track and re-estimate.
**Template**: "I break work into ≤2-day tasks, estimate ranges (best/likely/worst), flag unknowns with a spike, include testing/review/deploy, and communicate confidence. I compare actuals to estimates retrospectively." Example: spike on a third-party API revealed rate limits → adjusted plan early.

## B10. Code review philosophy
**Guidance**: Small PRs, kind, specific, automated checks handle style.
**Talking points**: Review for correctness, readability, tests, security (SQL injection, authorization), performance (N+1, `async` misuse), EF migrations; "nit:" labels; ask questions; approve with follow-ups; respond to reviews gracefully.

## B11. Production incident story
**Guidance**: Detect → mitigate → root cause → prevent; blameless.
**Template**: S: [Orders API 500s after release]. T: [On-call]. A: [Checked alerts/App Insights, correlated to deploy, rolled back via slot swap in 10 min, found connection pool exhaustion from missing `await using`/sync-over-async, fixed, wrote postmortem]. R: [MTTR 25 min; added load test and alert on pool usage].

## B12. Why are you leaving?
**Guidance**: Positive, forward-looking; no bad-mouthing.
**Template**: "I've learned a lot at [company], especially [X]. I'm looking for [larger scale/cloud-native work/ownership/growth path], which this role offers."

## B13. Strengths
**Guidance**: 2–3 strengths with evidence. **Template**: "Debugging and performance tuning — e.g., [story]. Also clear communication with non-technical stakeholders."

## B14. Weaknesses
**Guidance**: Real, non-critical, with active mitigation. **Template**: "I used to over-engineer early designs; now I write the simplest version, define extension points only when requirements appear, and use design reviews. Example: [YAGNI decision]."

## B15. Salary negotiation
**Guidance**: Research market range; let them name first if possible; give a range anchored on your target; discuss total compensation; be professional, don't lie about current pay; negotiate after an offer.
**Template**: "Based on my research for this level and location and my experience in [skills], I'm looking at [X–Y]. I'm flexible depending on the overall package and growth. What's the range for this role?"
**Counter-offer**: "I'm excited. Considering [impact/skills], could we get to [Z]?" Consider bonus, equity, learning budget, remote, notice period.

## B16. Describe a time you improved performance
**Template**: S: [Nightly batch took 4 h]. A: [Found row-by-row processing; moved to set-based SQL / `ExecuteUpdate`/bulk copy, batching, proper indexes]. R: [4 h → 18 min].

## B17. Describe a time you made a design decision with tradeoffs
**Template**: Pick CQRS/caching/outbox/queue. State options, criteria, decision, outcome. "We chose outbox over distributed transactions because …; trade-off was eventual consistency."

## B18. Tell me about a time you learned a new technology quickly
**Template**: S: [Had to adopt Azure Service Bus]. A: [Docs, proof-of-concept, asked an expert, wrote an internal guide]. R: [Shipped in 3 weeks].

## B19. How do you handle ambiguous requirements?
**Template**: A: [Ask clarifying questions, write assumptions, prototype/wireframe, short feedback loops, document decisions]. R: [Avoided rework].

## B20. Time you took ownership beyond your role
**Template**: A: [Noticed flaky tests/slow pipeline; owned fix, parallelized tests, cached NuGet]. R: [Pipeline 25 → 9 min].

## B21. How do you stay current?
**Talking points**: .NET blog/release notes, GitHub repos, conference talks (.NET Conf), Microsoft Learn, side projects, community; apply selectively, evaluate with spikes.

## B22. Dealing with a difficult stakeholder / changing requirements
**Template**: A: [Clarified underlying need, showed impact on timeline, offered options, documented agreement]. R: [Aligned expectations].

## B23. How do you ensure quality?
**Talking points**: Test pyramid (unit/integration with Testcontainers or `WebApplicationFactory`), CI gates, code review, static analysis, observability, feature flags, canary releases.

## B24. Tell me about a time you disagreed with a product decision
**Template**: A: [Quantified tech debt/risk, proposed alternative, agreed to timebox]. R: [Compromise].

## B25. Proudest achievement
**Template**: Choose something with impact and team credit; quantify.

## B26. Where do you see yourself in 3–5 years?
**Guidance**: Growth toward senior/lead/architect depth; tie to company needs, not job-hopping.

## B27. Do you have questions for us? (see Interview Strategy)

## B28. Tell me about working under pressure / on-call
**Template**: Calm, communicate status, mitigate first, escalate, document, follow-up improvements.

## B29. Handling feedback
**Template**: S: [Received feedback PRs too large]. A: [Adopted smaller PRs, stacked changes]. R: [Review time halved].

## B30. Disagreement over technical debt vs features
**Template**: A: [Quantified cost (incidents, dev hours), bundled debt with feature work, 20% capacity]. R: [Lower incidents].

**Interview traps (behavioral)**
- Saying "we" for everything — clarify *your* contribution.
- Blaming others; no lessons learned; no numbers.
- Over-long stories; no result.
- Inventing facts that fall apart in follow-ups.

**Cheat sheet**
- STAR; quantify; own mistakes; focus on learning.
- Prepare 6–8 flexible stories: performance, incident, conflict, failure, leadership, learning, ambiguity, ownership.

---

# Interview Strategy

## S1. Answering technical questions: Define → Example → Tradeoff
1. **Define** in one sentence ("A deadlock is two transactions each holding a lock the other needs").
2. **Example** concrete, ideally from experience or a small code/SQL snippet.
3. **Tradeoff/when not to use** ("Retrying on error 1205 is the fix; consistent lock ordering reduces them; snapshot isolation trades tempdb for fewer locks").
4. Check in: "Want me to go deeper into X?"
Think aloud, clarify requirements, state assumptions, mention testing/observability.

## S2. Questions you don't know
- Say so honestly; then reason from adjacent knowledge: "I haven't used X directly; I know Y which solves a similar problem — I'd expect…".
- Describe how you'd find out (docs, profiler, execution plan, a quick spike).
- Never bluff on specifics (API names/flags); a wrong confident answer hurts more than "I'm not sure".
- Come back later if you recall.

## S3. Asking good questions
- Team/process: "How are releases done? What does on-call look like? How do you handle tech debt?"
- Tech: ".NET version/upgrade policy? Testing strategy? Observability stack? Data layer (EF/Dapper), DB size and pain points?"
- Growth/role: "What does success look like in 6 months? How are performance and promotion evaluated?"
- Avoid: salary/leave first-round, anything easily Googled.

## S4. Whiteboard / live-coding tips
- Restate problem; clarify inputs, edge cases (null, empty, duplicates, large input).
- Outline approach and complexity before coding; start with simple correct solution, then optimize.
- Use meaningful names, small methods; talk while typing; test with examples and edge cases; handle `CancellationToken`/async if relevant.
- Know common structures: `Dictionary`, `HashSet`, `List`, `Queue`, `Stack`, LINQ (`GroupBy`, `Join`, `Aggregate`), `StringBuilder`.
- For SQL: state schema assumptions; use CTEs/window functions (`ROW_NUMBER`, `LAG`); mention indexes.
- If stuck: ask for a hint, narrow to a sub-problem. Compile/mentally run your code.

## S5. System-design framework (45 min)
1. **Requirements** (5 min): functional, non-functional (scale, latency, availability, consistency, security), constraints.
2. **Estimates**: users, QPS, data size/growth, read/write ratio.
3. **API & data model**: endpoints, entities, SQL vs NoSQL, indexes, partitioning.
4. **High-level architecture**: client → CDN/Front Door → API (ASP.NET Core) → cache (Redis) → DB (Azure SQL) → queue (Service Bus) → workers; storage (Blob).
5. **Deep dive**: bottleneck (DB, hot keys), consistency (outbox, idempotency), caching, pagination, auth (Entra ID/OAuth2/JWT), rate limiting.
6. **Reliability**: retries/backoff/circuit breaker, DLQ, health probes, multi-AZ/region, backups, RTO/RPO.
7. **Observability & operations**: logs/metrics/traces, SLOs, alerts, CI/CD, IaC, deployments.
8. **Tradeoffs & evolution**: what you'd change at 10× scale; cost.
Example practice systems: URL shortener, order processing with payments, notification service, file upload pipeline, rate limiter, chat, e-commerce catalog search.

## S6. 4-Week .NET Study Schedule
**Week 1 – Language & runtime**: C# (value/reference types, generics, delegates/events, LINQ, async/await, `Task` internals, `IDisposable`, records, pattern matching, nullable reference types, span basics), memory/GC, collections complexity, exceptions, SOLID/design patterns. Daily: 10 coding problems + review.
**Week 2 – ASP.NET Core & EF Core & SQL**: middleware pipeline, DI lifetimes, minimal APIs/controllers, auth (JWT/policies), validation, filters, configuration/options, logging, health checks; EF tracking vs no-tracking, loading strategies, N+1, migrations, concurrency tokens, transactions; SQL joins, window functions, indexes (clustered/nonclustered, covering), execution plans, isolation levels, deadlocks, normalization.
**Week 3 – Architecture & Cloud/DevOps**: REST design, CQRS, outbox, messaging, caching, resiliency (Polly), microservices vs monolith; this file's Docker/K8s/Azure/CI-CD/IaC/monitoring topics; build and deploy a small app end-to-end.
**Week 4 – Practice & polish**: 2 system-design mocks, 3 mock behavioral sessions (record yourself), 2 live-coding sessions, resume/projects polish, 15 behavioral stories rehearsed, company-specific research, rapid-revision sheet daily; light review last two days, rest well.

## S7. Expectations checklist by level
**Junior**
- C#: OOP, collections, LINQ basics, async/await basics, exceptions, `using`.
- .NET: build ASP.NET Core CRUD API, DI basics, config, Git basics.
- EF: CRUD, migrations, relationships, `Include`.
- SQL: SELECT/JOIN/GROUP BY, PK/FK, basic indexes, transactions concept.
**Mid**
- C#: generics, delegates, deferred execution, `IEnumerable` vs `IQueryable`, async pitfalls, immutability, unit-testing with mocks.
- .NET: middleware, auth, logging, options, background services, REST best practices, CI/CD, Docker basics.
- EF: tracking, N+1, projection, concurrency, raw SQL, transactions, query splitting, migration strategy.
- SQL: window functions, CTEs, indexing strategy, reading execution plans, isolation, stored procedures, pagination.
**Senior**
- C#: memory/GC, `Span<T>`, `ValueTask`, concurrency primitives, performance profiling, API design.
- .NET: architecture (clean/modular monolith, CQRS), resiliency, distributed systems issues (idempotency, outbox, eventual consistency), security (OWASP), observability, cloud design.
- EF: performance tuning, compiled queries, bulk ops (`ExecuteUpdate`/`ExecuteDelete`), multi-tenancy, interceptors, when to bypass EF.
- SQL: Query Store, parameter sniffing, partitioning, deadlock analysis, HA/DR, locking/blocking, schema evolution with zero downtime.
- Behavior: mentors, drives design reviews, owns incidents.
**Lead / Staff**
- Sets technical direction and standards, ADRs, roadmap/tech-debt strategy, cross-team alignment, hiring/mentoring, capacity/estimation, risk management, build-vs-buy, cost/FinOps, SLO ownership, communicates with executives, balances delivery with quality/people.

## S8. Resume & portfolio project ideas
- **E-commerce API**: ASP.NET Core, EF Core + SQL Server, JWT, Redis cache, outbox + Service Bus, Docker, GitHub Actions, Bicep, deployed to Container Apps with App Insights.
- **Order processing with Durable Functions** (saga/orchestration, retries, compensation).
- **Multi-tenant SaaS** (global query filters, row-level security in SQL).
- **Performance lab**: reproduce N+1, parameter sniffing, deadlock; document fixes with benchmarks (BenchmarkDotNet).
- **Real-time dashboard** with SignalR + background service.
- **Migration case study**: .NET Framework → .NET 8 via strangler.
- Resume tips: 1–2 pages, impact bullets ("Reduced report latency 85% by …"), tech stack section, links to GitHub, tailor keywords, avoid listing every tool.
- README for each project: architecture diagram, decisions, how to run, tests, CI badge.

## S9. Final rapid-revision sheet
**C#/.NET**: value vs reference; `struct` vs `class`; `async` returns `Task`, avoid `.Result`/`async void`; `ConfigureAwait(false)` in libraries; DI lifetimes (singleton/scoped/transient; no scoped in singleton); middleware order (exception → HTTPS → static → routing → CORS → auth → authz → endpoints); `IEnumerable` deferred execution; `IQueryable` translates to SQL; `record` value equality; GC generations 0/1/2 + LOH.
**EF Core**: `AsNoTracking` for reads; `Include` vs projection; N+1; `SaveChanges` is a transaction; concurrency via rowversion; split queries for cartesian explosion; `ExecuteUpdate/Delete`; migrations scripts in prod; `DbContext` scoped, not thread-safe.
**SQL Server**: clustered index = data order (one per table); nonclustered + INCLUDE columns for covering; SARGable predicates (no function on column); key lookups; parameter sniffing; isolation levels (RCSI/Snapshot reduce blocking); deadlock = lock ordering; `OFFSET/FETCH` or keyset pagination; window functions; `NOLOCK` = dirty reads.
**Docker/K8s**: multi-stage, layer cache, non-root, 8080; Deployment/Service/probes; requests vs limits; liveness cheap; HPA needs requests.
**Azure**: MI + Key Vault + RBAC; Service Bus at-least-once + DLQ; Front Door global vs App Gateway regional; failover groups; slots.
**CI/CD**: build once, deploy many; OIDC; environments + approvals; migrations via script/bundle.
**Git**: rebase local, merge shared; cherry-pick -x; revert vs reset; reflog.
**IaC**: Bicep (no state, what-if) vs Terraform (state, plan).
**Deploy**: rolling/blue-green/canary/flags; expand/contract.
**Monitoring**: logs/metrics/traces; SLI/SLO/SLA; burn-rate alerts; percentiles.
**Functions**: isolated worker; durable orchestrator deterministic; singleton clients.
**12-factor**: config in env, stateless, disposable, logs to stdout, admin jobs.
**Behavioral**: STAR; quantify; own mistakes; ask questions; negotiate with data.
**Day-of**: sleep, water, test setup/internet, arrive early, listen fully, structure answers, thank the interviewers and follow up in 24 h.
