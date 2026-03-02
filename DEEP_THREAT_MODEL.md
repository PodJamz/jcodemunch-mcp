# Deep Threat Model and Security Review

## Scope

This review covers the runtime and storage paths for:
- Local indexing (`index_folder`)
- GitHub indexing (`index_repo`)
- Index persistence and symbol retrieval (`IndexStore`)
- Optional AI summarization

## Assets

- Local filesystem confidentiality/integrity
- Cached indexed content under `~/.code-index/` (or `CODE_INDEX_PATH`)
- API credentials (`GITHUB_TOKEN`, `ANTHROPIC_API_KEY`)
- MCP tool response integrity for downstream agents

## Trust boundaries

1. **Untrusted repository content** (file paths, file bytes, symbol metadata derived from code).
2. **Trusted local host filesystem** where cache is stored.
3. **External APIs** (GitHub + optional Anthropic).
4. **MCP client/agent input** for tool arguments.

## Data flow summary

1. User/agent invokes tool (e.g., `index_repo`, `index_folder`).
2. Files discovered and filtered.
3. Parser extracts symbol metadata + byte offsets.
4. Index and raw file cache persisted under index store.
5. Retrieval tools read index metadata + raw bytes for symbols.

## Threat analysis by STRIDE-style categories

### 1) Tampering / path traversal in cache writes and reads

**Threat**: Untrusted file paths (from remote tree entries or malformed index metadata) could traverse outside cache directory and overwrite/read arbitrary local files.

**Prior state**: `IndexStore.save_index`, `incremental_save`, and `get_symbol_content` joined paths without validating containment.

**Fix implemented**:
- Added `_safe_content_path()` in `IndexStore` to resolve and enforce descendants of repository content dir.
- Enforced in raw-file writes, deletes, and symbol-content reads.
- Unsafe paths now raise `ValueError` for writes and return `None` for reads.

**Tests added**:
- `test_save_index_rejects_path_traversal_in_raw_files`
- `test_get_symbol_content_rejects_traversal_symbol_file`

**Residual risk**: If an attacker can directly edit cache JSON on disk, they can still corrupt metadata (availability/integrity of index contents), but traversal reads are blocked.

### 2) Information disclosure via secret indexing

**Threat**: Sensitive credentials accidentally indexed and exposed via retrieval tools.

**Current controls**:
- Secret filename pattern exclusion and binary detection.
- `.gitignore` + skip patterns + size caps.

**Residual risk**:
- Content-based secret scanning is not implemented; non-obviously named secret files may still be indexed.

### 3) DoS/resource exhaustion

**Threat**: Large repos or adversarial files cause CPU/memory/IO pressure.

**Current controls**:
- File count cap (500)
- File size cap (500KB default)
- Concurrency semaphore for GitHub fetches.

**Residual risk**:
- Parser-level pathological inputs may still degrade performance.

### 4) Spoofing and supply-chain concerns

**Threat**: Reliance on external APIs and dependencies introduces trust in remote services/packages.

**Current controls**:
- Optional API use; graceful fallback for summarization.
- No hidden outbound channels beyond expected feature calls.

**Residual risk**:
- Dependency compromise risk remains (standard Python ecosystem concern).

### 5) Repudiation / auditability

**Threat**: Difficult forensic tracing of who indexed what and when.

**Current controls**:
- `indexed_at`, language/file/symbol counts persisted.

**Residual risk**:
- No signed logs or immutable audit trail.

## Security conclusions

- No obvious malware logic identified (no C2, persistence installers, or credential exfil loops in reviewed paths).
- A real traversal-hardening gap existed in cache path handling and is now remediated with tests.
- The project demonstrates strong baseline defensive filtering; the largest remaining improvements are content-level secret scanning and stronger provenance controls.

## Recommended next hardening steps

1. Add optional content-based secret scanning before cache persistence.
2. Add strict owner/name sanitization for index filenames (defense in depth).
3. Add explicit maximum total bytes indexed per run.
4. Add structured security events (JSON logs) for skipped/unsafe paths.
5. Add fuzz tests for path and symbol metadata edge cases.
