# Ledgarr

**A Docker-first web application for inspecting, auditing, assessing, and maintaining self-hosted media libraries.**

Ledgarr provides a central view of media-library quality and consistency. It is intended to complement—not replace—Sonarr, Radarr, Recyclarr, Jellyfin, Plex, Emby, Bazarr, Tdarr, and similar specialist applications.

## Highlights

- Hierarchical library browsing: libraries, movies/shows, seasons, episodes, and files.
- Quality-profile assessment and outlier detection across seasons and libraries.
- Read Arr quality profiles, custom formats, and file-level scores where the APIs expose them; clearly distinguish unavailable scores from Ledgarr's own technical assessment.
- Multiple Sonarr/Radarr instances and separate Ledgarr assessment profiles.
- Naming and organization audits, missing/untracked-file checks, duplicate detection, and integrity reports.
- Track-policy audits for languages, subtitles, commentary, accessibility tracks, and preferred audio variants.
- Optional audio-generation workflows that add compatible tracks while preserving originals.
- Jellyfin/Plex/Emby playback compatibility estimates and transcoding-health diagnostics.
- Recyclarr GUI assistance for TRaSH configuration, ID discovery, YAML generation, validation, preview, sync, and history.
- Optional integrations with Bazarr, Tdarr, Unmanic, and other specialist tools.
- Reviewable operation queue, backups, validation, audit history, and scheduled scans.

## Design principles

- Docker-first deployment with persistent configuration and database storage.
- Browser is the interface; media access and processing happen on the backend.
- Read-only integrations by default and least-privilege container permissions.
- Arr remains authoritative for its profiles, custom-format scoring, and library records.
- Recyclarr remains the synchronization engine for TRaSH configuration.
- External processors remain optional backends; Ledgarr owns policy, candidate selection, orchestration, and verification.
- No destructive changes without explicit authorization and a recoverable operation plan.

See [Software Design](SOFTWARE_DESIGN.md) and [Technical Design](TECHNICAL_DESIGN.md).
