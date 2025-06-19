---
description: Defines the protocol and steps for developing MCP (Model Context Protocol) servers, with a focus on using the FastMCP framework.
author: https://github.com/nickbaumann98
version: 1.1
tags: ["mcp", "development", "protocol", "server", "integration"]
globs: ["*"]
---
# MCP Server Development Protocol

⚠️ CRITICAL: DO NOT USE attempt_completion BEFORE TESTING ⚠️

## Step 1: Planning (PLAN MODE)
- What problem does this tool solve?
- What API/service will it use?
- What are the authentication requirements?
  □ Standard API key
  □ OAuth (requires separate setup script)
  □ Other credentials

## Step 2: Implementation (ACT MODE)
1. Bootstrap
   - For web services, JavaScript integration, or Node.js environments:
     ```bash
     npm install fastmcp
     # Consider using the fastmcp-boilerplate: https://github.com/punkpeye/fastmcp-boilerplate
     # Or follow the quickstart in the FastMCP (TypeScript) README:
     # https://github.com/punkpeye/fastmcp/blob/main/README.md
     ```
   - For data science, ML workflows, or Python environments:
     ```bash
     # Using uv (recommended)
     uv pip install fastmcp
     # Or with pip
     pip install fastmcp
     # Follow the quickstart in the FastMCP (Python) README:
     # https://github.com/jlowin/fastmcp/blob/main/README.md
     ```

2. Core Implementation
   - Use FastMCP SDK (TypeScript: [punkpeye/fastmcp](https://github.com/punkpeye/fastmcp), Python: [jlowin/fastmcp](https://github.com/jlowin/fastmcp))
   - Implement comprehensive logging
     - TypeScript (for web/JS projects):
       ```typescript
       console.error('[Setup] Initializing server...');
       console.error('[API] Request to endpoint:', endpoint);
       console.error('[Error] Failed with:', error);
       ```
     - Python (for data science/ML projects):
       ```python
       import logging
       logging.error('[Setup] Initializing server...')
       logging.error(f'[API] Request to endpoint: {endpoint}')
       logging.error(f'[Error] Failed with: {str(error)}')
       # FastMCP (TypeScript) provides context-based logging:
       # execute: async (args, { log }) => { log.info("Message", args); }
       # FastMCP (Python) also provides context-based logging:
       # async def my_tool(ctx: Context, arg1: str): await ctx.info("Message")
       ```
   - Add type definitions (FastMCP often handles schema generation from type hints or libraries like Zod for TypeScript)
   - Handle errors with context (FastMCP provides `UserError` for user-facing errors)
   - Implement rate limiting if needed

3. Configuration
   - Get credentials from user if needed
   - Add to MCP settings:
     - For TypeScript (FastMCP) projects:
       ```json
       {
         "mcpServers": {
           "my-fastmcp-ts-server": {
             // Example using tsx for development
             "command": "npx",
             "args": ["tsx", "path/to/your/server.ts"],
             // Or for a built server:
             // "command": "node",
             // "args": ["path/to/build/index.js"],
             "env": {
               "API_KEY": "your_api_key_here"
             },
             "disabled": false,
             "autoApprove": []
           }
         }
       }
       ```
     - For Python (FastMCP) projects:
       ```json
       // In settings.json for the MCP client (e.g., Cline)
       {
         "mcpServers": {
           "my-fastmcp-py-server": {
             "command": "python", // Or your venv python path
             "args": ["path/to/your/server.py"], // Your FastMCP server script
             "env": {
               "API_KEY": "your_api_key_here"
             },
             "disabled": false,
             "autoApprove": []
           }
         }
       }
       # For local execution/testing, you'd typically run:
       # fastmcp run path/to/your/server.py
       ```
     - Note: The `mcp install` command is part of the official MCP SDK CLI. For FastMCP, you configure the client (like Cline) to run your server script directly.

## Step 3: Testing (BLOCKER ⛔️)

Tip: You can restart MCP servers by toucing the file `~/Library/Application Support/Code - Insiders/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json` without any actual changes.

<thinking>
BEFORE using attempt_completion, I MUST verify:
□ Have I tested EVERY tool?
□ Have I confirmed success from the user for each test?
□ Have I documented the test results?

If ANY answer is "no", I MUST NOT use attempt_completion.
</thinking>

1. Test Each Tool (REQUIRED)
   - **Unit Tests:**
     □ Write unit tests for individual functions, classes, and components within the MCP server to ensure they behave as expected in isolation.
   - **Integration Tests:**
     □ Test each tool with valid inputs.
     □ Verify output format is correct.
     □ **End-to-End MCP Workflow:**
       - Run the MCP server.
       - Implement or use an existing MCP Client.
       - Programmatically use the MCP Client to connect to your running server.
       - Call each of the server's tools via the client's programmatic API.
       - Assert that the responses and effects of the tools are correct.
   ⚠️ DO NOT PROCEED UNTIL ALL TOOLS ARE THOROUGHLY TESTED WITH BOTH UNIT AND INTEGRATION TESTS.

## Step 4: Completion
❗ STOP AND VERIFY:
□ Every tool has been tested with valid inputs
□ Output format is correct for each tool

Only after ALL tools have been tested can attempt_completion be used.

## Key Requirements
- ✓ Must use MCP SDK
- ✓ Must have comprehensive logging
- ✓ Must test each tool individually with unit and integration tests
- ✓ Must handle errors gracefully
- ⛔️ NEVER skip testing before completion

## Further Documentation

For the most up-to-date and detailed information on developing MCP servers with FastMCP, please refer to the official repositories and documentation:

- **FastMCP (TypeScript)**:
    - GitHub Repository: [https://github.com/punkpeye/fastmcp](https://github.com/punkpeye/fastmcp)
    - README (for quickstart and features): `https://raw.githubusercontent.com/punkpeye/fastmcp/main/README.md`
- **FastMCP (Python)**:
    - GitHub Repository: [https://github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp)
    - Official Documentation: [https://gofastmcp.com](https://gofastmcp.com)
    - README (for quickstart and overview): `https://raw.githubusercontent.com/jlowin/fastmcp/main/README.md`
- **MCP Specification**:
    - The official Model Context Protocol specification can be found at: [https://modelcontextprotocol.io/specification/2025-06-18.md](https://modelcontextprotocol.io/specification/2025-06-18.md)

You can use Cline's `fetch` tool or GitHub tools to access the raw README content from the URLs above if needed.
