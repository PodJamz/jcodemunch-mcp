# Repository Overview and Security Audit Notes

## What this project does

`jcodemunch-mcp` is an MCP server that indexes source code repositories/folders and exposes retrieval tools for AI agents. It parses supported languages into symbols (functions, classes, methods, constants, etc.), stores symbol metadata with byte offsets, and enables targeted lookup (`search_symbols`, `get_symbol`, `get_file_outline`, etc.) instead of loading entire files.

## How to use it (quick path)

1. Install:
   - `pip install git+https://github.com/jgravelle/jcodemunch-mcp.git`
2. Start/configure in an MCP client with command `jcodemunch-mcp`.
3. Index code:
   - GitHub repo: `index_repo { "url": "owner/repo" }`
   - Local folder: `index_folder { "path": "/path/to/project" }`
4. Retrieve precisely:
   - `get_repo_outline`, `get_file_outline`, `search_symbols`, `get_symbol`, `search_text`.

## Security posture (high-level)

The codebase includes explicit defensive controls for path traversal, symlink escape, secret-file exclusion, binary detection, and file-size/file-count limits. Local indexing validates file paths before discovery and again before reads. Remote indexing uses GitHub APIs and optional token auth.

## Manual malicious-code review summary

A manual static review found no obvious malware behavior (e.g., ransomware patterns, credential exfiltration loops, hidden persistence installers, command-and-control beacons, or obfuscated payload execution).

Observed external interactions are expected for product behavior:
- GitHub API fetches for repository indexing.
- Optional Anthropic API calls for AI-generated summaries.
- Local git command used only to read HEAD hash.

This is not a formal security certification; users should still review dependencies and run in least-privilege environments.
