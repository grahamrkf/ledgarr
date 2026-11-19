# Ledgarr — Software Design Document

**Status:** Draft  
**Version:** 0.1  
**License:** MIT

## 1. Purpose

Ledgarr is a Docker-first web application for auditing, assessing, comparing, and maintaining self-hosted media libraries. It consolidates technical inspection and policy-based maintenance into one interface while leaving specialist responsibilities with the services designed to perform them.

Ledgarr should help an administrator answer: What is in the library? Is it consistent? Which files fall short of the desired standard? What needs attention? What is safe to change, and what would the change do?

## 2. Product goals

1. Provide a hierarchical and searchable view of media libraries and their technical properties.
2. Identify quality outliers and assess files against configurable targets.
3. Reuse Sonarr/Radarr data and scoring rather than reimplementing their custom-format engines.
4. Provide useful track, naming, integrity, and playback-compatibility audits.
5. Offer a graphical way to configure and manage Recyclarr without replacing its synchronization engine.
6. Coordinate optional processing tools while retaining a safe, auditable operation workflow.
7. Integrate with media servers and companion applications using least privilege.
8. Remain maintainable and approachable for self-hosted users.

## 3. Target users

- Home-media server administrators.
- Users maintaining large movie and television libraries.
- Users operating multiple Sonarr/Radarr instances, including separate HD/UHD or anime libraries.
- Users who want consistent audio/subtitle policies and actionable upgrade reports.
- Administrators who want to understand actual playback/transcoding behaviour across client devices.

## 4. Product principles and system boundaries

- **Arr owns Arr:** Sonarr/Radarr remain authoritative for their media records, quality profiles, custom-format matching/scoring, release selection, and download workflows.
- **Recyclarr owns synchronization:** Ledgarr provides a GUI for configuring, validating, previewing, and invoking Recyclarr; it does not recreate TRaSH synchronization logic.
- **Media servers own playback:** Jellyfin/Plex/Emby remain responsible for serving and transcoding media. Ledgarr audits configuration and observed behaviour.
- **Specialist processors remain specialist:** Tdarr/Unmanic may execute processing jobs. Ledgarr can also provide a limited native FFmpeg/MKVToolNix backend.
- **Least privilege:** integrations are read-only by default; modifications and deeper container inspection require explicit configuration.
- **Safe by default:** scan and assessment are non-destructive. Every file operation is planned, reviewed, validated, and recorded.

## 5. Core modules

### 5.1 Library inventory and hierarchy

Present libraries as a hierarchy:

- Library.
- Movie or series.
- Season.
- Episode.
- Individual media file.

Support multiple library roots and multiple Arr instances. Display file path, size, duration, container, video/audio/subtitle streams, and linked external IDs where available.

### 5.2 Quality assessment and outlier detection

Users define Ledgarr assessment profiles such as HD-1080p, UHD-2160p, Anime-1080p, or Remux-2160p. These profiles are assessment targets and need not change Sonarr/Radarr download profiles.

For each item, show whether it is below, at, or above the target. Summaries should identify season/library consistency and outliers—for example, episodes whose reported custom-format scores or technical characteristics differ substantially from the rest.

Use Arr-reported custom-format scores and matched formats when the relevant API resource exposes them. Do not silently invent or recalculate an Arr score. Distinguish:
- Score reported by Arr.
- Ledgarr technical assessment.
- Score unavailable/not supplied by the connected Arr version or endpoint.

### 5.3 Comparison and technical inspection

Provide file-level detail and multi-file comparison. Align equivalent tracks across files, identify missing or different streams, and allow drill-down to the individual file. This functionality should reuse the Continuity Ledger media-analysis/track engine where practical.

### 5.4 Naming and organization audit

Check naming and folder structure against configurable templates, including TRaSH-inspired, P2P/Scene, simple media-server, and custom formats. Identify likely errors and warnings such as missing metadata, incorrect season/episode numbering, mismatched folder placement, collisions, and inconsistent extensions.

Use metadata from Arr or user-confirmed external identification. Do not guess unknown release group, edition, source, or episode order. Support manual correction and alternate episode orders where applicable.

### 5.5 Track policies and management

Policies may specify preferred audio/subtitle languages, required accessibility/commentary tracks, default/forced behaviour, and preferred audio variants. Audit compliance and identify candidates for correction.

Supported track edits should include language/name/flags/order and add/extract/remove operations where the container permits. Use the Continuity Ledger engine or equivalent shared services. Preserve originals and require operation review.

### 5.6 Audio variant generation

A policy may request an additional track when a preferred compatible variant is missing. Example: generate English AAC stereo from an existing English multichannel source while retaining the original lossless 5.1/7.1 track.

Policies can define source language, target codec, channel layout, bitrate, naming, default flag, and approval mode. Never synthesize a different language from an unrelated source. Skip files that already meet the policy. Generation is an optional processing workflow, not an unrestricted transcoding feature.

### 5.7 Recyclarr configuration manager

Provide a guided interface to:
- Browse human-readable TRaSH templates, custom formats, groups, and quality profiles.
- Search and resolve identifiers automatically.
- Assign configurations to multiple Sonarr/Radarr instances.
- Adjust supported settings and scores.
- Generate and inspect YAML, including advanced raw-YAML editing.
- Validate configuration and show dry-run/proposed changes.
- Run Recyclarr synchronization and present logs/results.
- Maintain configuration history and restore prior versions.
- Distinguish Recyclarr-managed settings from manually managed settings.

Recyclarr remains the execution/synchronization engine. Ledgarr must not silently overwrite managed configuration or claim that a preview is a completed sync.

### 5.8 Media-server playback compatibility and transcoding health

Connect to Jellyfin initially, with Plex/Emby considered later. Assess compatibility against selected client profiles and explain likely Direct Play, remux/direct stream, audio transcode, or full video transcode outcomes.

Audit relevant server configuration and environment, including acceleration method, FFmpeg version, GPU/device mapping, driver visibility, hardware decode/encode support, tone mapping, temporary transcode path, and permissions where available.

Separate configuration inspection from verified capability. A configured GPU does not prove that a complete decode/filter/encode pipeline works. Provide guided verification or inspect real playback/transcode sessions where possible. Flag likely CPU fallbacks, subtitle burn-in, HDR tone-mapping requirements, and repeated compatibility problems.

### 5.9 Integrations and ecosystem

Initial priorities:
- Sonarr and Radarr: multiple instances, library records, file paths, profiles, scores, and carefully scoped rename/refresh actions.
- Recyclarr: configuration GUI and execution.
- Jellyfin: library/server/playback/transcoding audit.

Optional integrations:
- Bazarr: subtitle coverage audit and links/triggered searches.
- Tdarr and Unmanic: dispatch candidate processing jobs and verify results.
- Plex and Emby: library/playback compatibility.
- Prowlarr, Seerr/Jellyseerr, Tautulli, Kometa, Maintainerr: contextual links or narrowly scoped integrations, not duplicated feature sets.
- Lidarr/music and book libraries are future expansion, not initial-release requirements.

## 6. User experience

### Main navigation

- Dashboard.
- Libraries.
- Quality & Outliers.
- Comparisons.
- Tracks & Policies.
- Naming & Organization.
- Processing.
- Recyclarr.
- Integrations.
- Audit History and Settings.

### Dashboard

Show actionable summaries: scan status, library counts, quality outliers, missing episodes/files, policy violations, pending operations, processing status, and integration health. Avoid overwhelming the user with low-value alerts.

### Review-first operations

Every operation should show affected files, source and destination, expected changes, warnings, backup strategy, and validation plan. Batch operations must enumerate the actual matching files/tracks affected.

## 7. Permissions and integration levels

### Default

- API access to configured services.
- Read-only media mounts for scanning.
- No Docker socket.
- No GPU device access.
- No unrestricted host filesystem access.

### Enhanced

- Restricted Docker socket proxy exposing only required read-only inspection endpoints.
- Read-only Compose/config mounts where needed.
- Read-only container log/config inspection.

### Advanced

- Optional host hardware diagnostic helper returning limited CPU/GPU/driver/device information.
- Explicitly configured write access for approved file operations.
- No privileged container and no unrestricted Docker socket as a convenience shortcut.

A read-only mount of the Docker socket does not make Docker API access read-only. Do not mount the raw Docker socket directly into Ledgarr. Use a narrowly restricted proxy or helper.

## 8. Non-functional requirements

- Docker Compose deployment with persistent configuration/database/operation data.
- Browser-based responsive interface.
- Background workers for scans, probes, and media operations.
- REST API and WebSocket/event channel for progress and status.
- Support read-only operation and multiple library roots.
- Clear permissions, audit logs, and recoverable operations.
- No assumption that all external API versions expose identical data.
- Graceful handling of offline services, inaccessible files, and missing tools.
- Secrets stored outside source-controlled configuration and masked in logs.

## 9. Success criteria

The initial useful release should connect to at least one Arr instance and one media library, inventory files, display technical metadata, compare quality/outliers, and produce a reliable report without modifying files. Subsequent releases can add track operations, Recyclarr GUI, and playback/transcoding diagnostics behind explicit integration permissions.

## 10. Initial delivery phases

1. **Foundation:** Docker deployment, authentication, library scanning, metadata database, hierarchical UI, audit history.
2. **Arr and quality:** multiple Arr instances, profile mapping, reported scores, quality assessment, outliers.
3. **Comparison and maintenance:** shared track engine, track policies, naming audits, safe operation queue, integrity checks.
4. **Recyclarr:** template/ID browser, YAML generation, validation, preview, sync, history.
5. **Playback and processing:** Jellyfin diagnostics, client compatibility, audio policies, optional Tdarr/Unmanic/native processing.
6. **Expansion:** Plex/Emby, scheduled audits, notifications, additional media types and integrations.
