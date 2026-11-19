# Ledgarr — Technical Design Document

**Status:** Draft  
**Version:** 0.1

## 1. Architecture overview

Ledgarr is a Docker-first web application. The browser communicates with a backend API; the backend owns filesystem access, integrations, media probing, database operations, and job orchestration.

```text
Browser UI (Vue 3 + TypeScript)
          │ HTTPS / REST / WebSocket
          ▼
Backend API (Python + FastAPI)
          │
          ├── Domain services and policy engine
          ├── Integration adapters (Arr, Recyclarr, Jellyfin, etc.)
          ├── Media analysis / Continuity Ledger engine
          ├── Operation planner and audit log
          └── Job queue / workers
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
    SQLite      Media tools   External services
                FFprobe       Sonarr/Radarr
                FFmpeg        Recyclarr
                MKVToolNix    Jellyfin/Plex/Emby
                              Tdarr/Unmanic/Bazarr
```

Keep the first implementation simple: a modular monolith with a separate background worker process if needed. Avoid microservices until scaling or isolation demonstrates a real need.

## 2. Proposed technology

- **Frontend:** Vue 3 + TypeScript; component library such as Vuetify may be evaluated.
- **Backend:** Python + FastAPI.
- **Persistence:** SQLite initially, with a repository layer to permit PostgreSQL later if multi-user/concurrency needs justify it.
- **Background work:** a durable database-backed job queue initially; choose a dedicated queue broker only when requirements demand it.
- **Media metadata:** FFprobe, with optional MediaInfo.
- **Media operations:** MKVToolNix and FFmpeg, ideally through a shared Continuity Ledger package.
- **Deployment:** Docker Compose.
- **API:** REST for resources/actions and WebSockets or server-sent events for progress.
- **Testing:** pytest, frontend unit/component tests, integration tests with mock services and small media fixtures.

## 3. Suggested repository layout

```text
ledgarr/
├── frontend/
│   ├── src/
│   └── package.json
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── domain/
│   │   ├── services/
│   │   ├── integrations/
│   │   ├── workers/
│   │   ├── persistence/
│   │   └── main.py
│   └── tests/
├── deploy/
│   ├── docker/
│   └── examples/
├── docs/
├── docker-compose.yml
└── README.md
```

## 4. Backend modules

### API layer

Owns authentication, request validation, API versioning, pagination, and serialization. It should not contain media-processing logic.

### Library service

Manages configured library roots, file discovery, hierarchy, scan state, and reconciliation between disk, Arr, and media-server records.

### Media analysis service

Calls the shared Continuity Ledger domain/application package for probing, normalized stream metadata, track matching, and file comparison. Ledgarr-specific code adds library context and persistence.

### Quality assessment service

Maps files to Ledgarr assessment profiles and reports below/at/above target. Stores source provenance for every score or assessment:
- Arr-reported custom-format score and matched formats, when available.
- Ledgarr-computed technical checks.
- Missing/unavailable external score.

Do not claim that a Ledgarr estimate is an Arr score. API response shapes differ by Arr version and endpoint; implement version-aware adapters and tests.

### Integration adapters

Each external application has an isolated adapter with connection settings, capability discovery, API version handling, health checks, and explicit permission scopes. A failed integration should not prevent local file scanning.

### Recyclarr service

Stores structured configuration and secrets references, generates YAML, validates against the supported Recyclarr schema/version, runs the Recyclarr executable/container, captures logs, and records config revisions. Recyclarr is the sync engine; Ledgarr is the UI/orchestrator.

### Policy engine

Evaluates track, naming, quality, and playback rules. Policies should be versioned, explainable, and testable. Each finding should include the rule, observed facts, target, severity, and recommended action.

### Operation planner/executor

Builds explicit plans for rename, track edits, file processing, and refresh actions. Plans have review status, permissions, validation steps, and execution results. The executor must use safe temporary-output/commit semantics and retain audit history.

### Worker system

Runs scans, probes, comparisons, external API polling, Recyclarr jobs, and media processing outside HTTP request handlers. Jobs need progress, cancellation where safe, retries for transient integration errors, and durable status.

## 5. Data model

Suggested initial entities:

- `User` / `AccessToken` or equivalent authentication records.
- `Integration` and `IntegrationCapability`.
- `Library` and `LibraryRoot`.
- `MediaItem` (movie/show/season/episode abstraction).
- `MediaFile` (path, size, timestamps, container, probe state).
- `Stream` (normalized track metadata plus raw probe data).
- `ArrInstance`, `ArrMediaRecord`, `QualityProfile`, `CustomFormatScoreSnapshot`.
- `AssessmentProfile`, `AssessmentResult`, `Policy`, `PolicyFinding`.
- `ComparisonSession`, `TrackGroup`, `TrackMatch`.
- `OperationPlan`, `OperationStep`, `OperationResult`.
- `Job`, `JobEvent`, `AuditEvent`.
- `RecyclarrConfigRevision` and `SyncRun`.
- `PlaybackClientProfile`, `PlaybackObservation`, `CompatibilityAssessment`.

Persist provenance, timestamps, source service/version, and confidence where relevant. Avoid duplicating all external service data when a cached snapshot or external ID is sufficient.

## 6. Filesystem and scanning model

- Media paths are configured explicitly through volume mounts.
- Default mounts should be read-only.
- Scan jobs discover candidate files and probe them asynchronously.
- Track file identity using path, size, modification time, and optional hashes.
- Handle symlinks and network mounts according to explicit settings.
- Detect inaccessible roots and distinguish missing files from temporarily unavailable mounts.
- Avoid hashing large media files by default.
- Reconcile file inventory against Arr/media-server records without assuming any one service has a complete view.

For Docker, store application state in persistent volumes (for example `/config` or `/data`) and mount media at stable in-container paths. Document UID/GID handling. Write access should be opt-in and scoped to only the directories that need modification.

## 7. Integration design

### Sonarr and Radarr

- Support multiple instances with independent base URLs, API keys, and labels.
- Read library records, file paths, profile assignments, quality information, custom-format data, and available file-level score data.
- Capability-detect API versions and endpoints.
- Treat unavailable custom-format scores as unavailable, not zero.
- Use native rename/refresh endpoints when possible to keep Arr databases consistent.
- Require explicit permission before issuing rename or other write actions.

### Recyclarr

- Keep generated configuration separate from user secrets.
- Support schema/version awareness, includes, environment-variable references, and multiple instances as implementation scope permits.
- Show a diff or dry-run result before sync.
- Record the exact configuration revision and Recyclarr version used for each run.
- Do not silently overwrite manually maintained settings.

### Jellyfin / Plex / Emby

- Start with Jellyfin.
- Use service APIs for system information, playback sessions, and accessible configuration.
- Do not assume every transcoding setting is exposed through the public API.
- Optional read-only config/Compose mounts or restricted Docker inspection may supplement API data.
- Distinguish configured acceleration from verified decode/filter/encode capability.
- Avoid direct access to the media server's GPU devices for ordinary auditing.
- Use observed playback/transcode sessions or guided test playback to verify real behaviour.

### Docker inspection

Permission levels:

1. **Basic:** service API access only; no Docker socket, GPU device, or host filesystem access.
2. **Enhanced:** restricted Docker socket proxy exposing only required read-only inspection endpoints; optional read-only Compose/config mounts.
3. **Advanced:** optional narrow host diagnostic helper for CPU/GPU/driver information.

Never mount the raw Docker socket directly into Ledgarr as a shortcut. A read-only filesystem mount of the socket does not make the Docker API read-only and may effectively grant host-level control. Do not run Ledgarr privileged.

### Tdarr / Unmanic / Bazarr and others

Use optional adapters. Ledgarr identifies candidates and policy requirements, submits jobs only when explicitly enabled, and verifies resulting files. It should not recreate each application's plugin system or specialized management interface.

## 8. Playback compatibility model

A compatibility assessment is an estimate based on the selected client profile, container, codecs, subtitle handling, network constraints, and server configuration. Classify likely outcomes as:

- Direct Play.
- Remux/container conversion.
- Direct Stream/audio-only conversion.
- Full video transcode.

Identify likely causes: unsupported container/codec, audio incompatibility, subtitle burn-in, HDR tone mapping, bitrate limits, and client-specific capability constraints. Compare estimates with actual playback observations where available. Clearly label estimates versus observed outcomes.

## 9. API and event design

Use versioned REST endpoints, for example:

- `/api/v1/libraries`
- `/api/v1/media-items`
- `/api/v1/files/{id}`
- `/api/v1/comparisons`
- `/api/v1/assessment-profiles`
- `/api/v1/policies`
- `/api/v1/findings`
- `/api/v1/operations`
- `/api/v1/integrations`
- `/api/v1/recyclarr/configurations`
- `/api/v1/jobs`

Use WebSockets or server-sent events for job progress and status changes. Keep API schemas typed and generated from backend models where practical. Do not expose raw secrets in API responses.

## 10. Security and permissions

- Require authentication for network-accessible deployments.
- Support reverse-proxy deployment and document trusted-proxy configuration.
- Store API keys and credentials securely; mask them in logs and UI.
- Use least-privilege API keys where supported.
- Default media mounts to read-only.
- Scope write mounts to selected roots and enable only for approved workflows.
- No unrestricted Docker socket or privileged mode.
- Validate all paths and subprocess arguments.
- Treat filenames, media metadata, and external API content as untrusted.
- Audit configuration changes and file operations.
- Provide backup/restore guidance for database and configuration.
- Consider CSRF, CORS, session security, and rate limiting according to the chosen authentication model.

## 11. Operation safety

Use the same transaction-like pattern as the shared Continuity Ledger engine:

1. Build a plan and identify affected files.
2. Revalidate file state and permissions.
3. Check destination collisions and free space.
4. Process to temporary output.
5. Validate streams and expected metadata.
6. Commit atomically where possible.
7. Preserve backup/rollback information.
8. Refresh Arr/media-server records when required.
9. Record full audit result.

External processors may have their own execution semantics; Ledgarr should record job IDs, inputs, outputs, and verification results rather than pretending it controls their internals.

## 12. Deployment

Initial Compose deployment should include:
- `ledgarr` API/web service.
- Optional `worker` service using the same application image.
- Persistent configuration/database volume.
- Explicit media mounts.
- Optional network attachment to the user's media-service network.

Example conceptual configuration:

```yaml
services:
  ledgarr:
    image: ghcr.io/example/ledgarr:latest
    ports:
      - "8787:8787"
    volumes:
      - ./config:/config
      - /srv/media:/media:ro
    environment:
      - TZ=America/Edmonton
      - PUID=1000
      - PGID=1000
    restart: unless-stopped
```

This is illustrative, not a finalized Compose file. The eventual image may use a different internal user/permission model. Do not include API keys in a committed Compose example.

## 13. Observability and operations

- Structured application logs with secret/path redaction options.
- Job-level progress and error summaries.
- Integration health checks and last-success timestamps.
- Audit trail for settings, policies, and operations.
- Metrics for scan throughput, queue depth, and failures where useful.
- Exportable diagnostic bundle with secrets excluded.

## 14. Testing strategy

- Unit tests for policy evaluation, score provenance, path normalization, and comparison integration.
- Contract tests against mocked Sonarr/Radarr/Jellyfin/Recyclarr APIs and multiple versions.
- Integration tests with small sample libraries and media files.
- Tests for read-only enforcement and denied write operations.
- Docker integration tests for mount permissions and restricted proxy behaviour.
- Failure tests for unavailable services, stale files, interrupted jobs, full disks, and invalid outputs.
- End-to-end tests for the key inventory → assessment → review → operation workflow.

## 15. Open technical decisions

- Authentication strategy for single-user and multi-user deployments.
- SQLite concurrency limits and migration point to PostgreSQL, if needed.
- Durable job queue implementation.
- Exact supported Arr API versions and file-level custom-format score sources.
- Recyclarr version/schema support policy.
- Whether Continuity Ledger is imported as a Python package or run as a worker service.
- First supported media-server APIs beyond Jellyfin.
- Exact semantics for operation rollback after external service refreshes.
