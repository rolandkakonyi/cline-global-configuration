---
Date: 2025-06-20
TaskRef: "Update Rules/mcp-server-development-protocol.md for FastMCP"

Learnings:
- Updated MCP server development protocol to use FastMCP for both TypeScript and Python.
- TypeScript FastMCP: uses `npm install fastmcp`, repository at `punkpeye/fastmcp`.
- Python FastMCP: uses `uv pip install fastmcp`, repository at `jlowin/fastmcp`, documentation at `gofastmcp.com`.
- FastMCP provides context-based logging (e.g., `ctx.info()`, `log.info()`) and often handles schema generation.
- Configuration in `cline_mcp_settings.json` for FastMCP servers involves directly calling the server script (e.g., `tsx server.ts` or `python server.py`).
- Added a "Further Documentation" section to the rule, pointing to official FastMCP resources.

Difficulties:
- Fetching large README files from GitHub using the `fetch` tool required multiple calls with `start_index` due to content truncation. This is a recurring pattern for large remote content.

Successes:
- Successfully integrated new information about FastMCP into the existing `mcp-server-development-protocol.md` rule.
- Maintained the structure of the rule while updating relevant sections for accuracy.

Improvements_Identified_For_Consolidation:
- General pattern: Strategy for fetching large files in chunks using the `fetch` tool with `start_index` parameter.
- Project-Specific (Cline Rules): Key details for FastMCP (TS & Python) bootstrapping, logging, configuration, and documentation links.
---
