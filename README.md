# VisScan

> **A full-stack DevSecOps scanning platform — web dashboard + automated GitLab CI pipeline**

VisScan is an open-source platform that lets developers trigger, monitor, and audit security scans for their containerised services through a web UI. When a scan is started, a job is enqueued via RabbitMQ, a TypeScript worker picks it up and triggers a dedicated GitLab CI pipeline on a self-hosted runner, and results flow back in real time through webhooks and a status poller. Every scan runs Gitleaks (secret detection), Semgrep (SAST), and Trivy (container + OS vulnerabilities) — with a **Verified Release Gate** that blocks promotion of any image carrying unresolved CRITICAL findings.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)
![GitLab CI](https://img.shields.io/badge/GitLab%20CI-FC6D26?logo=gitlab&logoColor=white)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?logo=rabbitmq&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?logo=kubernetes&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15-4169E1?logo=postgresql&logoColor=white)

---

## Architecture

VisScan is composed of four runtime components that work together:

| Component | Technology | Role |
|---|---|---|
| **Web App** | Next.js 16 + tRPC + NextAuth | Dashboard, scan management, admin panel |
| **Worker** | Node.js (tsx) | RabbitMQ consumer; triggers GitLab CI pipelines |
| **GitLab CI Pipeline** | GitLab CI + self-hosted runner | Clones repo, compiles, builds, scans, gates release |
| **Database** | PostgreSQL 15 + Prisma ORM | Persists users, projects, scan history, findings |

```mermaid
flowchart TD
    %% ── USER LAYER ──────────────────────────────────────────────────────
    U([User / Developer])
    U -- "POST /api/scan/start\n(scanMode, serviceId, imageTag)" --> A

    %% ── WEB APPLICATION (Next.js) ────────────────────────────────────────
    subgraph WEB ["Web Application  •  Next.js 16 + tRPC"]
        A[scan/start API\nValidate quota · resolve credentials\nCreate ScanHistory QUEUED]
        A -- "publishScanJob()\ntype: SCAN_AND_BUILD | SCAN_ONLY\npriority 1–10" --> PUB

        subgraph QUEUE_LIB ["lib/queue/publisher.ts"]
            PUB[RabbitMQ Publisher\namqplib]
        end

        WH["/api/webhook\nReceive per-stage updates\nMerge findings · update DB"]
        STREAM["/api/scan/stream\nSSE — live status to browser"]
    end

    %% ── MESSAGE BROKER ──────────────────────────────────────────────────
    subgraph MQ ["RabbitMQ  •  Priority Queues"]
        BQ["scan_jobs_build\n(SCAN_AND_BUILD jobs)\nprefetch 5"]
        SQ["scan_jobs_scan\n(SCAN_ONLY jobs)\nprefetch 5"]
        DLQ["scan_jobs_dlq\n(Dead Letter)"]
    end

    PUB -- "SCAN_AND_BUILD" --> BQ
    PUB -- "SCAN_ONLY" --> SQ
    BQ -. "undeliverable" .-> DLQ
    SQ -. "undeliverable" .-> DLQ

    %% ── WORKER ──────────────────────────────────────────────────────────
    subgraph WKR ["Worker  •  worker/index.ts  (tsx)"]
        CON["buildChannel.consume()\nscanChannel.consume()"]
        WAIT["Concurrency Gate\nmax 5 parallel per mode\npoll DB every 5s"]
        TRIG["triggerGitLab()\nPOST /projects/:id/trigger/pipeline\nform-urlencoded variables"]
        POLL["Status Poller\npollRunningScans()\nevery 10 s"]
        ART["fetchReportArtifacts()\nDownload JSON reports\nfrom GitLab Jobs API"]
    end

    BQ --> CON
    SQ --> CON
    CON --> WAIT
    WAIT -- "slot available" --> TRIG
    TRIG -- "pipelineId" --> DB
    POLL -- "GET /pipelines/:id" --> GL_API
    POLL -- "on SUCCESS" --> ART
    ART -- "trivy/semgrep/gitleaks JSON" --> DB

    %% ── GITLAB CI PIPELINE ──────────────────────────────────────────────
    subgraph GL ["GitLab CI Pipeline  •  .gitlab-ci.yml  (8 stages)"]
        direction TB
        S0["① maintenance\nupdate_trivy_db\n(nightly cache warmer)"]
        S1["② setup\nfetch_and_detect\nClone repo · auto-detect stack\n(Node/Go/Python/Java-Maven/Java-Gradle/Docker)"]
        S2A["③ security_audit\ngitleaks_scan\n(secret detection)"]
        S2B["③ security_audit\nsemgrep_scan\n(SAST — p/ci ruleset)"]
        S3["④ compile\ncompile_node | compile_go\ncompile_java_maven | compile_java_gradle\ncompile_python\n(SKIP if SCAN_ONLY)"]
        S4["⑤ build_artifact\nbuild_and_push\nKaniko daemon-less build\n→ Docker Hub :temp-{pipelineId}"]
        S5A["⑥ container_scan\ntrivy_scan\nCRITICAL/HIGH/MEDIUM · cached DB"]
        S5B["⑥ container_scan\ngenerate_sbom\nCycloneDX SBOM via Trivy"]
        S6["⑦ release\npush_to_hub  ← MANUAL GATE\nBlocks if CRITICAL > 0\nCosign image signing (optional)"]
        S7["⑧ cleanup\nDelete :temp-{pipelineId}\nSend final webhook"]
    end

    TRIG -- "API trigger\n+variables" --> GL_API["GitLab API"]
    GL_API --> S0
    S0 --> S1
    S1 --> S2A & S2B & S3
    S2A & S2B --> S4
    S3 --> S4
    S4 --> S5A --> S5B --> S6 --> S7

    %% ── WEBHOOKS: PIPELINE BACK TO APP ──────────────────────────────────
    S2A -- "gitleaks-report.json\n(multipart POST)" --> WH
    S2B -- "semgrep-report.json\n(multipart POST)" --> WH
    S5A -- "status: SUCCESS\n(JSON POST)" --> WH
    S7  -- "status: SUCCESS | FAILED\n(JSON POST)" --> WH
    WH  -- "merge findings\ncalc vuln counts\nupdate ScanHistory" --> DB
    WH  -- "CRITICAL > 0 → FAILED_SECURITY" --> DB

    %% ── VERIFIED RELEASE GATE ────────────────────────────────────────────
    S6 -- "grep CRITICAL\nin trivy-report.json" --> GATE{Verified\nRelease Gate}
    GATE -- "✅ 0 criticals\ncrane cp :temp → :tag" --> PUSHED["Image published\n:user-tag"]
    GATE -- "❌ criticals found\nexit 1" --> BLOCKED["Pipeline BLOCKED\n:temp deleted in cleanup"]

    %% ── DATABASE ────────────────────────────────────────────────────────
    subgraph DB_BOX ["PostgreSQL 15  •  Prisma ORM"]
        DB[("ScanHistory\nProjectService\nProjectGroup\nUser / Credential\nAuditLog")]
    end

    WEB --> DB_BOX
    WKR --> DB_BOX

    %% ── SSE BACK TO BROWSER ──────────────────────────────────────────────
    DB -- "status change" --> STREAM
    STREAM -- "real-time\nstage progress" --> U

    %% ── STYLING ──────────────────────────────────────────────────────────
    style GATE fill:#1a1a2e,stroke:#e94560,color:#fff
    style PUSHED fill:#16213e,stroke:#0f9b58,color:#fff
    style BLOCKED fill:#16213e,stroke:#e94560,color:#fff
    style S6 fill:#1a1a2e,stroke:#e94560,color:#fff
    style MQ fill:#1a1a1a,stroke:#FF6600,color:#fff
    style WKR fill:#1a1a1a,stroke:#3178C6,color:#fff
    style GL fill:#1a1a1a,stroke:#FC6D26,color:#fff
```

---

## Key Features

**Dual scan modes.** `SCAN_AND_BUILD` clones the repo, auto-detects the stack (Node, Go, Python, Java/Maven, Java/Gradle, or existing Dockerfile), compiles if necessary, builds with Kaniko, then scans the resulting container image. `SCAN_ONLY` runs Gitleaks and Semgrep on source code without building — useful for quick secret and SAST checks on any commit.

**Auto-stack detection.** The `fetch_and_detect` stage inspects the cloned repository for `package.json`, `go.mod`, `pom.xml`, `build.gradle`, `requirements.txt`, or a `Dockerfile` and sets the correct compile and build path automatically.

**Daemon-less Kaniko builds.** Container images are built using Kaniko with no Docker daemon, making them safe to run inside rootless Kubernetes pods. Named `--chown` flags are patched to numeric UID/GID at build time for full Kaniko compatibility.

**Dual-lane priority queue.** RabbitMQ maintains two separate priority queues — `scan_jobs_build` and `scan_jobs_scan` — each with `prefetch(5)` to allow up to 5 concurrent jobs per lane on an 8-core runner. Jobs support priority levels 1–10.

**Verified Release Gate.** The `push_to_hub` job is manual and checks the Trivy report for CRITICAL severity findings before promoting the temporary image (`:temp-{pipelineId}`) to the user's chosen tag. If any criticals exist, the job exits 1 and the cleanup stage deletes the temporary image.

**SBOM generation.** After every successful container build, `generate_sbom` produces a CycloneDX-format Software Bill of Materials via Trivy, stored as a 30-day pipeline artifact.

**Cosign image signing.** An optional `sign_image` stage signs the promoted image with Sigstore Cosign if `COSIGN_PRIVATE_KEY` is configured as a CI/CD variable.

**Real-time scan progress.** The web app streams pipeline job updates (stage name, status, duration) back to the browser via Server-Sent Events, sourced from both the RabbitMQ worker poller and GitLab webhooks.

**Webhook + poller dual-path.** Scan results arrive through two independent paths: GitLab CI jobs `POST` their reports to `/api/webhook` as multipart form data (with the raw JSON report file attached), while the worker's poller queries the GitLab Jobs API every 10 seconds as a fallback. Both paths merge findings and are idempotent — duplicate events are dropped.

**Zombie detection.** The poller marks any scan still `RUNNING` after 180 minutes as `FAILED` automatically to prevent stuck records.

**Per-user quota and admin panel.** Admins can set per-user project quotas, approve or bulk-approve pending users (CMU EntraID OAuth), override Dockerfiles, and view the full scan history across all users.

---

## Screenshots

### Dashboard
![Dashboard](public/landing/dashboard.png)

### Scan Pipeline — Live Progress
![Scan Pipeline](public/landing/scan-pipeline.png)

### Scan Results
![Scan Results](public/landing/scan-result.png)

### Scan History & Comparison
![Scan History](public/landing/scan-history.png)

### Compare Scans
![Compare Scans](public/landing/compare-scan.png)

### Docker Template Override
![Docker Template](public/landing/docker-template.png)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Next.js 16, React 18, Tailwind CSS, tRPC, TanStack Query |
| Auth | NextAuth v4, CMU EntraID (OAuth2), credential encryption (AES) |
| Backend API | Next.js Route Handlers, Zod validation, audit logging |
| Worker | Node.js + tsx, amqplib, Axios |
| Message Broker | RabbitMQ 3 (priority queues + dead-letter queue) |
| Database | PostgreSQL 15, Prisma ORM |
| CI Pipeline | GitLab CI/CD, self-hosted runner, 8 stages |
| Container Build | Kaniko v1.23 (daemon-less, rootless Kubernetes) |
| SAST | Semgrep 1.100 (p/ci ruleset) |
| Secret Detection | Gitleaks v8.18 (configurable via `.gitleaks.toml`) |
| Container Scan | Trivy 0.53 (cached DB via nightly maintenance job) |
| SBOM | Trivy (CycloneDX format) |
| Image Signing | Sigstore Cosign (optional) |
| Image Registry | Docker Hub |
| Observability | OpenTelemetry (OTLP HTTP exporter) |
| Deployment | Docker Compose (web + worker + db + rabbitmq) |

---

## Project Structure

```
VisScan/
├── app/                        # Next.js App Router
│   ├── api/
│   │   ├── scan/start/         # POST — enqueue a scan job
│   │   ├── scan/[id]/stream/   # GET  — SSE live pipeline status
│   │   ├── webhook/            # POST — receives GitLab CI stage reports
│   │   ├── admin/              # Admin: users, quotas, scan history
│   │   └── auth/               # NextAuth + CMU EntraID proxy
│   ├── dashboard/              # Main project/service dashboard
│   ├── scan/                   # Scan pages: build, scan-only, history, compare
│   └── admin/                  # Admin UI: users, history, settings, templates
│
├── worker/
│   └── index.ts                # RabbitMQ consumer + GitLab trigger + status poller
│
├── lib/
│   ├── queue/
│   │   ├── publisher.ts        # RabbitMQ publisher (amqplib)
│   │   └── types.ts            # ScanJob, JobResult, queue name constants
│   ├── prisma.ts               # Prisma client singleton
│   ├── auth.ts                 # NextAuth config
│   ├── crypto.ts               # AES credential encryption/decryption
│   ├── quotaManager.ts         # Per-user scan quota enforcement
│   └── scanConfig.ts           # Scan retention and cleanup config
│
├── components/
│   ├── pipeline/               # PipelineStepper, FindingsTable, LogsPanel, etc.
│   └── ScanLiveWatcher.tsx     # SSE consumer for real-time stage updates
│
├── prisma/
│   └── schema.prisma           # DB schema: User, Credential, ProjectGroup,
│                               #   ProjectService, ScanHistory, AuditLog
│
├── docker/
│   ├── Dockerfile              # Multi-stage Next.js production image
│   ├── Dockerfile.worker       # Worker image (tsx runtime)
│   └── docker-compose.prod.yml # web + worker + db + rabbitmq
│
└── .gitlab-ci.yml              # 8-stage CI pipeline (maintenance→cleanup)
```

---

## Getting Started

### Prerequisites

| Tool | Version |
|---|---|
| Node.js | 20 |
| Docker + Docker Compose | v2 |
| PostgreSQL | 15 (or use the compose stack) |
| RabbitMQ | 3 (or use the compose stack) |
| GitLab project | With a self-hosted runner registered |

### 1. Clone and configure

```bash
git clone https://github.com/Unlxii/VisScan.git
cd VisScan
cp .env.example .env
# Fill in DATABASE_URL, GITLAB_PROJECT_ID, GITLAB_TRIGGER_TOKEN,
# GITLAB_TOKEN, RABBITMQ_URL, NEXTAUTH_SECRET, CMU EntraID credentials
```

### 2. Start infrastructure

```bash
cd docker
docker compose -f docker-compose.prod.yml up -d db rabbitmq
```

### 3. Run database migrations and seed admin

```bash
npx prisma db push
npx tsx scripts/create-admin.ts
```

### 4. Start the web app and worker

```bash
# Development
npm run dev          # Next.js web app (port 3000)
npm run worker:dev   # Worker with hot-reload

# Production (Docker)
docker compose -f docker/docker-compose.prod.yml up -d
```

### 5. Register your GitLab CI runner

The pipeline in `.gitlab-ci.yml` is triggered by the worker via the GitLab Trigger API. Register a self-hosted runner on the GitLab project, then set `GITLAB_PROJECT_ID` and `GITLAB_TRIGGER_TOKEN` in your `.env`.

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | ✅ | PostgreSQL connection string |
| `RABBITMQ_URL` | ✅ | RabbitMQ AMQP URL (e.g. `amqp://guest:guest@localhost:5672`) |
| `GITLAB_PROJECT_ID` | ✅ | GitLab project ID hosting `.gitlab-ci.yml` |
| `GITLAB_TRIGGER_TOKEN` | ✅ | GitLab pipeline trigger token |
| `GITLAB_TOKEN` | ✅ | Personal access token for Jobs API (artifact download, status polling) |
| `GITLAB_API_URL` | ✅ | GitLab API base URL (`https://gitlab.com/api/v4`) |
| `NEXTAUTH_SECRET` | ✅ | Random secret for NextAuth session encryption |
| `NEXTAUTH_URL` | ✅ | Public base URL of the web app |
| `ENCRYPTION_KEY` | ✅ | AES key for encrypting stored Git/Docker credentials |
| `CMU_ENTRAID_CLIENT_ID` | Optional | CMU EntraID OAuth client ID |
| `GITLAB_WEBHOOK_SECRET` | Optional | Shared secret for verifying CI webhooks |
| `COSIGN_PRIVATE_KEY` | Optional | Sigstore Cosign private key for image signing |
| `ADMIN_PASSWORD` | Optional | Password for the seeded admin account |

---

## CI Pipeline Stage Reference

| Stage | Job(s) | Runs when | Description |
|---|---|---|---|
| `maintenance` | `update_trivy_db` | Schedule / manual | Downloads latest Trivy vulnerability DB into shared cache |
| `setup` | `fetch_and_detect` | Always | Clones target repo, auto-detects stack, exports `STACK`, `CONTEXT_PATH`, `FINAL_IMAGE_NAME` |
| `security_audit` | `gitleaks_scan`, `semgrep_scan` | Always | Secret detection and SAST — run in parallel, results POSTed to webhook |
| `compile` | `compile_node/go/python/java_*` | `SCAN_AND_BUILD` only | Compiles source; Java JAR cached between stages |
| `build_artifact` | `build_and_push` | `SCAN_AND_BUILD` only | Kaniko build → push `:temp-{pipelineId}` to Docker Hub |
| `container_scan` | `trivy_scan`, `generate_sbom` | `SCAN_AND_BUILD` only | CVE scan + CycloneDX SBOM generation |
| `release` | `push_to_hub` (manual), `sign_image` | `SCAN_AND_BUILD` only | **Verified Release Gate** — blocks if CRITICAL > 0; optionally signs with Cosign |
| `cleanup` | *(unnamed)* | Always | Deletes `:temp` image; sends final SUCCESS/FAILED webhook |

---

## Related Repository

| Repo | Description |
|---|---|
| [VisScan-gitlab-ci-template](https://github.com/Unlxii/VisScan-gitlab-ci-template) | Reusable `.gitlab-ci.yml` include template — integrate VisScan into any project's pipeline |

---

## License

MIT © [Ronnachai Sitthichoksathit](https://github.com/Unlxii) & [Kittiwat Yasarawan](https://github.com/Fittokung)
