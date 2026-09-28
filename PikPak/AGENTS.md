# PikPak Agent Manager (AGENTS.md)

This project provides an autonomous agent manager for PikPak cloud drive operations (browsing, cloud downloading, large-scale file classification, directory organization, quota monitoring, and maintenance).

---

## 🚨 Critical Security & Privacy Rules (Highest Priority)

1. **Strictly Prohibit Exposing Environment & Secrets in Output**:
   - **NEVER** print, echo, or log the contents of `.env`, tokens, API keys, or authorization credentials in chat messages, terminal outputs, error messages, or generated code.
   - **NEVER** display the full string of `PIKPAK_TOKEN`.
   - If token status must be verified, output only safe, sanitized indicators (e.g., "Configured / Not configured" or prefix with ellipsis `eyJhbG...`).
2. **Credential Isolation & Zero Hardcoding**:
   - Always load sensitive credentials via local `.env` or system environment variables. Never hardcode credentials into scripts or persistent files.
   - Never redirect sensitive outputs to tracked or persistent files.
3. **Strict Disposable Script Policy (Tmp-Only & Immediate Cleanup)**:
   - **Destination Restriction**: All temporary, one-off, or scratch scripts MUST ONLY be created under the system temporary directory (`/tmp/`). Strictly prohibit generating disposable scripts in the project directory, workspace root, or tracked folders.
   - **Immediate Post-Execution Cleanup**: Disposable scripts and intermediate dump files MUST be deleted immediately upon execution completion (`rm -f /tmp/<script>`). Leaving leftover scratch scripts is strictly forbidden.
4. **Irreversible Action Confirmation & Dry-Run**:
   - For single destructive operations: always list target files and ask for explicit user confirmation before executing (bulk delete, empty trash, directory purge).
   - For batch classifications (> 50 files): **MUST execute a Dry-Run first**, providing a summary plan before performing any batch move or rename.

---

## ⚖️ Dual-Mode Strategy: MCP First vs. Python Batch Pipeline

To balance latency, token efficiency, and high-throughput execution, the agent strictly adopts a **tiered execution strategy**:

### Mode A: MCP Native Tools (Interactive & Single-Task Operations)
- **Applicable Scenarios**: Daily user questions, quota checks, single link offline downloads, browsing current folder, checking specific file details, moving/renaming 1~10 files.
- **Workflow**:
  - Prioritize native MCP function calls (`list_files`, `create_download_task`, `get_quota`, `move_file`, `rename_file`) when `pikpak` MCP server is active.
  - Zero terminal overhead, structured validation, instant user feedback.

### Mode B: Python Batch Pipeline (Mass File Classification & Heavy Tasks)
- **Applicable Scenarios**: Batch organizing folders, classifying hundreds or thousands of files, recursive full-drive scans, disk space distribution analytics.
- **Execution Architecture (Three-Stage Pipeline)**:
  1. **Fast Paged Ingestion & Local Cache**:
     - Pull remote file metadata via `GET /drive/v1/files` using cursor `page_token` loop.
     - Persist snapshot into local `.cache/` (never dump thousands of raw file objects into chat context).
  2. **Rule-Based Pre-Classification (95% Local / 5% LLM)**:
     - Group files by extension, video quality, anime/series regex, date, or tags purely in local Python memory (< 1ms).
     - Only send unresolved edge cases to the model for semantic judgment.
  3. **High-Concurrency Execution & Manifest Logging**:
     - Use `ThreadPoolExecutor` (5~10 workers) to batch dispatch cloud file moves/renames (`PATCH /drive/v1/files/{id}`).
     - Record `manifest.json` (source ID, original parent, new parent, timestamp) to support one-click rollback if needed.

---

## Dev Environment & Setup Commands

- **Python Version**: Python 3.8+ (standard library `urllib.request` preferred).
- **Environment Configuration**: Ensure `.env` exists in the project root with `PIKPAK_TOKEN=<token>`.
- **MCP Configuration**: Mounted in `~/.gemini/config/mcp_config.json` targeting `https://pikpak.ai/mcp`.
- **Smoke Test & Verification**:
  ```bash
  python3 -c "
  import os, urllib.request, json
  token = os.environ.get('PIKPAK_TOKEN')
  if not token and os.path.exists('.env'):
      for line in open('.env'):
          if line.startswith('PIKPAK_TOKEN='):
              token = line.strip().split('=', 1)[1].strip('\"\' ')
  req = urllib.request.Request('https://api-drive.mypikpak.com/drive/v1/about', headers={'Authorization': f'Bearer {token}'})
  with urllib.request.urlopen(req, timeout=10) as r:
      d = json.loads(r.read())
      print('Status: OK | Limit:', d.get('quota', {}).get('limit'))
  "
  ```
- **Inspect Reference Documentation**:
  - Task Error Codes & Self-Healing: See `TROUBLESHOOTING.md`
  - CLI Operation Cheatsheet: See `.agents/skills/pikpak-cli/SKILL.md`

---

## Architecture & Core Endpoints

| Capability | Target / Protocol | Description & Conventions |
| :--- | :--- | :--- |
| **PikPak MCP** | `https://pikpak.ai/mcp` | SSE Model Context Protocol endpoint for interactive AI tools |
| **Quota & Account** | `GET https://api-drive.mypikpak.com/drive/v1/about` | Storage capacity, trash usage, download quotas |
| **File Listing** | `GET https://api-drive.mypikpak.com/drive/v1/files` | Root directory ID is empty string `""` (never `/` or `root`) |
| **Cloud Download** | `POST https://api-drive.mypikpak.com/drive/v1/files` | Offline magnet / URL tasks (requires `cloud_download` scope) |
| **File Move / Rename** | `PATCH https://api-drive.mypikpak.com/drive/v1/files/{id}` | Update `parent_id` (move) or `name` (rename) without re-uploading |

---

## Execution Guidelines & Conventions

1. **Root Directory ID**:
   - Always represent root as `""` (empty string). Passing `/` or `root` will fail or return empty results.
2. **Quota Awareness & 25% Rule**:
   - Respect the **25% Connected Apps Quota Rule**: If downstream speed throttles to 100 KB/s or uploads fail, report quota state to user rather than spamming retries.
3. **Safe User Feedback**:
   - Output concise, tabular summaries for file lists and task results.
   - Convert byte sizes into human-readable units (MB, GB, TB).
