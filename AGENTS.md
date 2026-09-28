# Drive-Agent (AGENTS.md)

This project provides an autonomous, vendor-agnostic agent manager specialized in cloud storage organization, directory restructuring, media library classification (Plex/Emby/Jellyfin standards), junk/ad cleaning, quota monitoring, and resource lifecycle management across unified cloud storage backends (WebDAV, AList, OpenList, CloudDrive, Rclone, S3-compatible, and proprietary cloud drives).

---

## 🚨 Critical Security & Risk Guardrails (Highest Priority)

1. **Zero-Exposure Policy for Credentials & Secrets**:
   - **NEVER** print, echo, or log tokens, API keys, WebDAV passwords, cookies, authorization headers, or `.env` content in chat messages, terminal outputs, error logs, or code.
   - When inspecting authentication or storage status, output only safe, sanitized indicators (e.g., `Configured (Length: 32)` or masked prefix `eyJhbGciOi...`).
   - All credentials must be loaded from local environment variables or `.env`. Hardcoding credentials into persistent or scratch scripts is strictly prohibited.

2. **Strict Disposable Script Policy (Tmp-Only & Immediate Teardown)**:
   - **Target Directory**: All temporary, one-off, or scratch scripts MUST ONLY be created under the system temporary directory (`/tmp/`).
   - Strictly prohibit generating disposable scripts in the workspace root, project directories, or tracked folders.
   - **Immediate Cleanup**: Disposable scripts and intermediate dump files MUST be deleted immediately upon execution completion (`rm -f /tmp/<script>`). Leaving leftover scratch scripts is strictly forbidden.

3. **Destructive Action Guardrails & Dry-Run Protocol**:
   - **Explicit User Confirmation**: For destructive operations (permanent deletion, emptying trash, purging directories), always list target files and ask for explicit user confirmation before executing.
   - **Mandatory Dry-Run**: For any batch reorganization, moving, or renaming exceeding 20 files, the agent **MUST execute a Dry-Run first**, displaying a structured preview table of proposed changes before applying modifications.

4. **Manifest Logging & 100% Rollback Guarantee**:
   - Any batch move, rename, or file modification MUST record an execution manifest (`manifests/manifest_<timestamp>.json`).
   - The manifest records: `item_id`, `original_path`, `new_path`, `action`, `timestamp`, and `status`.
   - Every batch operation must be reversible using an automated rollback routine.

---

## ⚖️ Dual-Mode Execution Strategy: Native Tools vs. Batch Pipeline

To balance latency, token efficiency, and high-throughput execution across heterogeneous cloud providers, the agent strictly adopts a **tiered execution strategy**:

```
                       [Incoming User Request]
                                  │
                  Is batch size > 20 files OR 
                  requires recursive scanning?
                                 / \
                                /   \
                        [No]   /     \   [Yes]
                              ▼       ▼
                     ┌──────────────────┐    ┌───────────────────────────────────┐
                     │ Mode A: Native   │    │ Mode B: Python Batch Pipeline     │
                     │ MCP / CLI Tools  │    │ 1. Paged Ingestion & Local Cache  │
                     │ (1 ~ 20 files)   │    │ 2. Hybrid Rule Classification     │
                     │ Instant feedback │    │ 3. Concurrency + Manifest Logging │
                     └──────────────────┘    └───────────────────────────────────┘
```

### Mode A: Native Tools & Direct Operations (Interactive & Targeted)
- **Applicable Scenarios**: Daily user inquiries, quota checks, browsing a single directory, single-link offline downloads, moving/renaming 1~20 files, regex renaming, removing empty folders, inspecting single file metadata.
- **Workflow**:
  - Prioritize native storage MCP function calls or lightweight CLI commands (`rclone`, `openlist`, `webdav`, etc.):
    - **Listing & Search**: `list`, `stat`, `search`, `storage_status`
    - **Reorganization**: `move`, `rename`, `batch_rename`, `mkdir`
    - **Housekeeping**: `remove_empty_dirs`, `remove` (after confirmation)
  - Zero terminal overhead, structured validation, instant user feedback.

### Mode B: Python Batch Pipeline (Mass Classification & Heavy Tasks)
- **Applicable Scenarios**: Organizing folders with hundreds or thousands of files, recursive full-drive scans, media library restructuring, bulk junk/ad cleaning, deduplication.
- **Three-Stage Architecture**:
  1. **Fast Paged Ingestion & Local Cache**:
     - Traverse remote directories via cursor/page token loops or recursive listing.
     - Persist directory snapshot into `.cache/` (never dump thousands of raw file objects into LLM chat context).
  2. **Hybrid Rule-Based Pre-Classification (95% Local / 5% LLM)**:
     - Execute 95% of classification purely in local Python memory (< 1ms per file) using regular expressions (anime/TV series pattern matching, movie title/year, video resolution tags, file extensions).
     - Only send unresolved edge cases (ambiguous titles, multi-part collections) to the model for semantic reasoning.
  3. **High-Concurrency Execution & Manifest Logging**:
     - Dispatch cloud file moves/renames with controlled concurrency (`ThreadPoolExecutor` with 5~8 workers) calling storage APIs or MCP.
     - Implement exponential backoff and jitter for API rate limits (`429 Too Many Requests`).
     - Commit changes to `manifests/manifest_<timestamp>.json` upon completion to guarantee rollback capability.

---

## 📁 File Classification & Naming Standards

The agent organizes cloud drives according to standard media server formats (Plex / Emby / Jellyfin / Infuse) and structured data hierarchies:

### 1. Media Library Structure
- **Movies**:
  - Format: `Movies/{Movie Title} ({Year})/{Movie Title} ({Year}) [{Resolution}][{Codec}].{ext}`
  - Example: `Movies/Inception (2010)/Inception (2010) [1080p][H265].mkv`
  - Subtitle pairing: Subtitles must match video filename (e.g., `Inception (2010) [1080p][H265].zh-CN.srt`).
- **TV Shows & Anime**:
  - Format: `TV Shows/{Show Title}/Season {SS}/{Show Title} - S{SS}E{EE} - {Episode Title}.{ext}`
  - Example: `TV Shows/Breaking Bad/Season 01/Breaking Bad - S01E01 - Pilot.mp4`
  - Anime format: Standardize multi-part releases, SPs, and OVAs into `Season 01` or `Season 00 (Specials)`.

### 2. General Data Hierarchy
- `_Inbox/`: Newly downloaded, unclassified, or staging files.
- `Documents/{Category}/{Year}/`: Work, finance, study, and electronic books (`.pdf`, `.epub`, `.mobi`).
- `Software/{OS: Windows|macOS|Linux|Android}/`: Installers and application binaries (`.exe`, `.dmg`, `.apk`, `.deb`).
- `Archives/{Category}/`: Compressed archives (`.zip`, `.7z`, `.rar`, `.tar.gz`, `.iso`).
- `Torrents/`: Seed files (`.torrent`) and torrent metadata.

### 3. Advertising & Junk Cleaning Rules
- **Automatic Deletion Targets** (after Dry-Run preview):
  - Known ad text files: `*扫码*.txt`, `*关注*.txt`, `*最新网址*.url`, `*防屏蔽*.html`, `*www.*.url`.
  - Empty directories: Directories containing zero files after moves (`remove_empty_dirs`).
  - Zero-byte files: Incomplete or broken zero-byte downloads.
- **Protected Files**: NEVER delete files matching standard subtitle formats (`.srt`, `.ass`, `.vtt`, `.sub`), nfo metadata (`.nfo`), or album artwork (`poster.jpg`, `fanart.jpg`).

---

## 🛠️ Unified Storage Abstraction Layer

The agent interacts with underlying cloud storage through standardized operational primitives:

| Universal Primitive | Typical Backends (MCP / CLI / REST) | Description & Best Practice |
| :--- | :--- | :--- |
| **`fs_list`** | MCP `fsList` / `rclone lsjson` / REST `GET /files` | Fetch directory contents with pagination/cursor support |
| **`fs_stat`** | MCP `fsGet` / `rclone stat` / REST `GET /meta` | Inspect single file size, hash, and MIME type |
| **`fs_move`** | MCP `fsMove` / `rclone moveto` / REST `PATCH` | Relocate file or directory without re-uploading |
| **`fs_rename`** | MCP `fsRename` / `fsBatchRename` / REST `PATCH` | In-place renaming, regex pattern replacement |
| **`fs_mkdir`** | MCP `fsMkdir` / `rclone mkdir` / REST `POST /dir` | Recursive directory creation (parent-safe) |
| **`fs_remove`** | MCP `fsRemove` / `rclone delete` / REST `DELETE` | Move to trash or purge (guarded by dry-run) |
| **`fs_prune_empty`**| MCP `fsRemoveEmptyDirectory` / `rclone rmdirs` | Prune empty parent folders post-classification |
| **`fs_quota`** | MCP `fsStorageStatus` / `rclone about` / REST `/quota` | Query total, used, and remaining drive capacity |

---

## 🔄 Manifest Specification & Rollback Protocol

Every batch operation generates a manifest file in `manifests/`:

```json
{
  "manifest_id": "manifest_20260928_220000",
  "created_at": "2026-09-28T22:00:00+08:00",
  "total_items": 42,
  "successful": 42,
  "failed": 0,
  "operations": [
    {
      "item_id": "file_123456",
      "action": "move",
      "original_path": "/_Inbox/[SubGroup] Show - 01 [1080p].mp4",
      "target_path": "/TV Shows/Show/Season 01/Show - S01E01 [1080p].mp4",
      "status": "success"
    }
  ]
}
```

### Automated Rollback Protocol
To revert any batch operation:
1. Load the target manifest from `manifests/`.
2. Generate an inverted execution plan: swap `target_path` and `original_path`.
3. Check for destination path conflicts before moving.
4. Execute reversal via worker pool or native tools and write a rollback audit log.

---

## 📋 Execution Guidelines & Interaction Norms

1. **Path Handling**:
   - Always use normalized forward slashes `/`.
   - The virtual root directory is uniformly represented as `/`.
   - Strip leading/trailing trailing whitespace and invalid filesystem characters (`:`, `*`, `?`, `"`, `<`, `>`, `|`) when targeting cross-platform drives.
2. **Rate Limit & Backoff**:
   - Respect upstream storage API quotas; if encountering `429 Too Many Requests` or `503 Service Unavailable`, apply exponential backoff with jitter and notify the user rather than aggressively retrying.
3. **Safe & Concise User Output**:
   - Provide concise tabular summaries for listings, task progress, and dry-run previews:
     ```markdown
     | Original Path | Target Path | Action | Rule |
     | :--- | :--- | :--- | :--- |
     | /_Inbox/Inception.2010.mkv | /Movies/Inception (2010)/Inception (2010).mkv | Move | Movie Match |
     ```
   - Format byte sizes with human-readable units (`KB`, `MB`, `GB`, `TB`).
   - Display progress counters for batch processing (e.g., `Processing 25/120 files (20.8%)`).
