---
description: A comprehensive guide for developing MCP (Model Context Protocol) clients in various languages, based on official quickstart documentation.
author: Cline (adapted from MCP Docs)
version: 1.0
tags: ["mcp", "client-development", "python", "typescript", "java", "kotlin", "csharp", "guide", "quickstart"]
globs: ["*"]
---

# MCP Client Development Guide

> Get started building your own client that can integrate with all MCP servers.

In this tutorial, you'll learn how to build an LLM-powered chatbot client that connects to MCP servers. It helps to have gone through the server quickstart that guides you through the basics of building your first server.

---

## Python Client Development

[You can find the complete code for this tutorial here.](https://github.com/modelcontextprotocol/quickstart-resources/tree/main/mcp-client-python)

### System Requirements

Before starting, ensure your system meets these requirements:

*   Mac or Windows computer
*   Latest Python version installed
*   Latest version of `uv` installed

### Setting Up Your Environment

First, create a new Python project with `uv`:

```bash
# Create project directory
uv init mcp-client
cd mcp-client

# Create virtual environment
uv venv

# Activate virtual environment
# On Windows:
.venv\Scripts\activate
# On Unix or MacOS:
source .venv/bin/activate

# Install required packages
uv add mcp anthropic python-dotenv

# Remove boilerplate files
# On Windows:
del main.py
# On Unix or MacOS:
rm main.py

# Create our main file
touch client.py
```

### Setting Up Your API Key

You'll need an Anthropic API key from the [Anthropic Console](https://console.anthropic.com/settings/keys).

Create a `.env` file to store it:

```bash
# Create .env file
touch .env
```

Add your key to the `.env` file:

```bash
ANTHROPIC_API_KEY=<your key here>
```

Add `.env` to your `.gitignore`:

```bash
echo ".env" >> .gitignore
```

> **Warning:**
> Make sure you keep your `ANTHROPIC_API_KEY` secure!

### Creating the Client

#### Basic Client Structure

First, let's set up our imports and create the basic client class:

```python
import asyncio
from typing import Optional
from contextlib import AsyncExitStack
import sys # Added import

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv()  # load environment variables from .env

class MCPClient:
    def __init__(self):
        # Initialize session and client objects
        self.session: Optional[ClientSession] = None
        self.exit_stack = AsyncExitStack()
        self.anthropic = Anthropic()
    # methods will go here
```

#### Server Connection Management

Next, we'll implement the method to connect to an MCP server:

```python
    async def connect_to_server(self, server_script_path: str):
        """Connect to an MCP server

        Args:
            server_script_path: Path to the server script (.py or .js)
        """
        is_python = server_script_path.endswith('.py')
        is_js = server_script_path.endswith('.js')
        if not (is_python or is_js):
            raise ValueError("Server script must be a .py or .js file")

        command = "python" if is_python else "node"
        server_params = StdioServerParameters(
            command=command,
            args=[server_script_path],
            env=None
        )

        stdio_transport = await self.exit_stack.enter_async_context(stdio_client(server_params))
        self.stdio, self.write = stdio_transport
        self.session = await self.exit_stack.enter_async_context(ClientSession(self.stdio, self.write))

        await self.session.initialize()

        # List available tools
        response = await self.session.list_tools()
        tools = response.tools
        print("\nConnected to server with tools:", [tool.name for tool in tools])
```

#### Query Processing Logic

Now let's add the core functionality for processing queries and handling tool calls:

```python
    async def process_query(self, query: str) -> str:
        """Process a query using Claude and available tools"""
        messages = [
            {
                "role": "user",
                "content": query
            }
        ]

        response = await self.session.list_tools()
        available_tools = [{
            "name": tool.name,
            "description": tool.description,
            "input_schema": tool.inputSchema
        } for tool in response.tools]

        # Initial Claude API call
        api_response = self.anthropic.messages.create( # Renamed to avoid conflict
            model="claude-3-5-sonnet-20241022",
            max_tokens=1000,
            messages=messages,
            tools=available_tools
        )

        # Process response and handle tool calls
        final_text = []

        assistant_message_content = []
        for content in api_response.content: # Use renamed variable
            if content.type == 'text':
                final_text.append(content.text)
                assistant_message_content.append(content)
            elif content.type == 'tool_use':
                tool_name = content.name
                tool_args = content.input

                # Execute tool call
                result = await self.session.call_tool(tool_name, tool_args)
                final_text.append(f"[Calling tool {tool_name} with args {tool_args}]")

                assistant_message_content.append(content)
                messages.append({
                    "role": "assistant",
                    "content": assistant_message_content
                })
                messages.append({
                    "role": "user",
                    "content": [
                        {
                            "type": "tool_result",
                            "tool_use_id": content.id,
                            "content": result.content
                        }
                    ]
                })

                # Get next response from Claude
                next_api_response = self.anthropic.messages.create( # Renamed
                    model="claude-3-5-sonnet-20241022",
                    max_tokens=1000,
                    messages=messages,
                    tools=available_tools
                )

                final_text.append(next_api_response.content[0].text) # Use renamed

        return "\n".join(final_text)
```

#### Interactive Chat Interface

Now we'll add the chat loop and cleanup functionality:

```python
    async def chat_loop(self):
        """Run an interactive chat loop"""
        print("\nMCP Client Started!")
        print("Type your queries or 'quit' to exit.")

        while True:
            try:
                query = input("\nQuery: ").strip()

                if query.lower() == 'quit':
                    break

                response_text = await self.process_query(query) # Renamed
                print("\n" + response_text)

            except Exception as e:
                print(f"\nError: {str(e)}")

    async def cleanup(self):
        """Clean up resources"""
        await self.exit_stack.aclose()
```

#### Main Entry Point

Finally, we'll add the main execution logic:

```python
async def main():
    if len(sys.argv) < 2:
        print("Usage: python client.py <path_to_server_script>")
        sys.exit(1)

    client = MCPClient()
    try:
        await client.connect_to_server(sys.argv[1])
        await client.chat_loop()
    finally:
        await client.cleanup()

if __name__ == "__main__":
    # import sys # Already imported at the top
    asyncio.run(main())
```

You can find the complete `client.py` file [here.](https://gist.github.com/zckly/f3f28ea731e096e53b39b47bf0a2d4b1)

### Key Components Explained

#### 1. Client Initialization
*   The `MCPClient` class initializes with session management and API clients
*   Uses `AsyncExitStack` for proper resource management
*   Configures the Anthropic client for Claude interactions

#### 2. Server Connection
*   Supports both Python and Node.js servers
*   Validates server script type
*   Sets up proper communication channels
*   Initializes the session and lists available tools

#### 3. Query Processing
*   Maintains conversation context
*   Handles Claude's responses and tool calls
*   Manages the message flow between Claude and tools
*   Combines results into a coherent response

#### 4. Interactive Interface
*   Provides a simple command-line interface
*   Handles user input and displays responses
*   Includes basic error handling
*   Allows graceful exit

#### 5. Resource Management
*   Proper cleanup of resources
*   Error handling for connection issues
*   Graceful shutdown procedures

### Common Customization Points

1.  **Tool Handling**
    *   Modify `process_query()` to handle specific tool types
    *   Add custom error handling for tool calls
    *   Implement tool-specific response formatting
2.  **Response Processing**
    *   Customize how tool results are formatted
    *   Add response filtering or transformation
    *   Implement custom logging
3.  **User Interface**
    *   Add a GUI or web interface
    *   Implement rich console output
    *   Add command history or auto-completion

### Running the Client

To run your client with any MCP server:

```bash
uv run client.py path/to/server.py # python server
uv run client.py path/to/build/index.js # node server
```

> **Note:**
> If you're continuing the weather tutorial from the server quickstart, your command might look something like this: `python client.py .../quickstart-resources/weather-server-python/weather.py`

The client will:

1.  Connect to the specified server
2.  List available tools
3.  Start an interactive chat session where you can:
    *   Enter queries
    *   See tool executions
    *   Get responses from Claude

Here's an example of what it should look like if connected to the weather server from the server quickstart:

![Python Client CLI Example](https://mintlify.s3.us-west-1.amazonaws.com/mcp/images/client-claude-cli-python.png)

### How It Works

When you submit a query:

1.  The client gets the list of available tools from the server
2.  Your query is sent to Claude along with tool descriptions
3.  Claude decides which tools (if any) to use
4.  The client executes any requested tool calls through the server
5.  Results are sent back to Claude
6.  Claude provides a natural language response
7.  The response is displayed to you

### Best practices

1.  **Error Handling**
    *   Always wrap tool calls in try-catch blocks
    *   Provide meaningful error messages
    *   Gracefully handle connection issues
2.  **Resource Management**
    *   Use `AsyncExitStack` for proper cleanup
    *   Close connections when done
    *   Handle server disconnections
3.  **Security**
    *   Store API keys securely in `.env`
    *   Validate server responses
    *   Be cautious with tool permissions

### Troubleshooting

#### Server Path Issues
*   Double-check the path to your server script is correct
*   Use the absolute path if the relative path isn't working
*   For Windows users, make sure to use forward slashes (/) or escaped backslashes (\\) in the path
*   Verify the server file has the correct extension (.py for Python or .js for Node.js)

Example of correct path usage:
```bash
# Relative path
uv run client.py ./server/weather.py

# Absolute path
uv run client.py /Users/username/projects/mcp-server/weather.py

# Windows path (either format works)
uv run client.py C:/projects/mcp-server/weather.py
uv run client.py C:\\projects\\mcp-server\\weather.py
```

#### Response Timing
*   The first response might take up to 30 seconds to return
*   This is normal and happens while:
    *   The server initializes
    *   Claude processes the query
    *   Tools are being executed
*   Subsequent responses are typically faster
*   Don't interrupt the process during this initial waiting period

#### Common Error Messages
If you see:
*   `FileNotFoundError`: Check your server path
*   `Connection refused`: Ensure the server is running and the path is correct
*   `Tool execution failed`: Verify the tool's required environment variables are set
*   `Timeout error`: Consider increasing the timeout in your client configuration

### Testing with a Simple Echo Server

Testing your MCP client's core functionality (like connection, tool listing, and tool invocation) can be greatly simplified by using a local "echo" server. This server would typically have a single tool, for example, `echo_tool`, that simply returns any input it receives.

**Benefits:**
*   **Isolated Testing:** Test your client's logic without dependencies on external services or complex server setups.
*   **Quick Verification:** Easily confirm that your client can connect, list tools, send requests, and process responses.
*   **Debugging:** Simplify debugging of request/response handling and data serialization.

**Example Echo Server (Python with FastMCP):**
A simple echo server can be created using a framework like FastMCP.
```python
# echo_server.py
from fastmcp import FastMCP, ToolContext

mcp = FastMCP()

@mcp.tool(
    name="echo_tool",
    description="Echoes back the input message.",
    input_schema={
        "type": "object",
        "properties": {"message": {"type": "string", "description": "The message to echo."}},
        "required": ["message"],
    }
)
async def echo_tool(ctx: ToolContext, message: str):
    # await ctx.info(f"Echoing message: {message}") # Optional logging
    return {"echoed_message": message}

if __name__ == "__main__":
    mcp.run()
```
Save this as `echo_server.py` in your project.

**Connecting Your Client:**
Modify your client's connection command to point to this local server script:
```bash
uv run client.py path/to/your/echo_server.py
```

**Testing Client Logic:**
Once connected, your client will list `echo_tool`. You can then send a query that your LLM (e.g., Claude) would interpret to use this tool. For example, a query like "Use echo_tool to echo 'hello world'" should trigger the tool. You can then verify that your client correctly handles the tool call and receives the echoed response (e.g., `{'echoed_message': 'hello world'}`). This helps confirm the client's request-response cycle with the MCP server.

---

## Node.js (TypeScript) Client Development

[You can find the complete code for this tutorial here.](https://github.com/modelcontextprotocol/quickstart-resources/tree/main/mcp-client-typescript)

### System Requirements

Before starting, ensure your system meets these requirements:

*   Mac or Windows computer
*   Node.js 17 or higher installed
*   Latest version of `npm` installed
*   Anthropic API key (Claude)

### Setting Up Your Environment

First, let's create and set up our project:

**MacOS/Linux:**
```bash
# Create project directory
mkdir mcp-client-typescript
cd mcp-client-typescript

# Initialize npm project
npm init -y

# Install dependencies
npm install @anthropic-ai/sdk @modelcontextprotocol/sdk dotenv

# Install dev dependencies
npm install -D @types/node typescript

# Create source file
touch index.ts
```

**Windows:**
```powershell
# Create project directory
md mcp-client-typescript
cd mcp-client-typescript

# Initialize npm project
npm init -y

# Install dependencies
npm install @anthropic-ai/sdk @modelcontextprotocol/sdk dotenv

# Install dev dependencies
npm install -D @types/node typescript

# Create source file
new-item index.ts
```

Update your `package.json` to set `type: "module"` and a build script:

```json
{
  "type": "module",
  "scripts": {
    "build": "tsc && chmod 755 build/index.js"
  }
}
```

Create a `tsconfig.json` in the root of your project:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["index.ts"],
  "exclude": ["node_modules"]
}
```

### Setting Up Your API Key

You'll need an Anthropic API key from the [Anthropic Console](https://console.anthropic.com/settings/keys).

Create a `.env` file to store it:

```bash
echo "ANTHROPIC_API_KEY=<your key here>" > .env
```

Add `.env` to your `.gitignore`:

```bash
echo ".env" >> .gitignore
```

> **Warning:**
> Make sure you keep your `ANTHROPIC_API_KEY` secure!

### Creating the Client

#### Basic Client Structure

First, let's set up our imports and create the basic client class in `index.ts`:

```typescript
import { Anthropic } from "@anthropic-ai/sdk";
import {
  MessageParam,
  Tool,
} from "@anthropic-ai/sdk/resources/messages/messages.mjs";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";
import readline from "readline/promises";
import dotenv from "dotenv";

dotenv.config();

const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
if (!ANTHROPIC_API_KEY) {
  throw new Error("ANTHROPIC_API_KEY is not set");
}

class MCPClient {
  private mcp: Client;
  private anthropic: Anthropic;
  private transport: StdioClientTransport | null = null;
  private tools: Tool[] = [];

  constructor() {
    this.anthropic = new Anthropic({
      apiKey: ANTHROPIC_API_KEY,
    });
    this.mcp = new Client({ name: "mcp-client-cli", version: "1.0.0" });
  }
  // methods will go here
}
```

#### Server Connection Management

Next, we'll implement the method to connect to an MCP server:

```typescript
  async connectToServer(serverScriptPath: string) {
    try {
      const isJs = serverScriptPath.endsWith(".js");
      const isPy = serverScriptPath.endsWith(".py");
      if (!isJs && !isPy) {
        throw new Error("Server script must be a .js or .py file");
      }
      const command = isPy
        ? process.platform === "win32"
          ? "python"
          : "python3"
        : process.execPath;

      this.transport = new StdioClientTransport({
        command,
        args: [serverScriptPath],
      });
      await this.mcp.connect(this.transport);

      const toolsResult = await this.mcp.listTools();
      this.tools = toolsResult.tools.map((tool) => {
        return {
          name: tool.name,
          description: tool.description,
          input_schema: tool.inputSchema,
        };
      });
      console.log(
        "Connected to server with tools:",
        this.tools.map(({ name }) => name)
      );
    } catch (e) {
      console.log("Failed to connect to MCP server: ", e);
      throw e;
    }
  }
```

#### Query Processing Logic

Now let's add the core functionality for processing queries and handling tool calls:

```typescript
  async processQuery(query: string) {
    const messages: MessageParam[] = [
      {
        role: "user",
        content: query,
      },
    ];

    const response = await this.anthropic.messages.create({
      model: "claude-3-5-sonnet-20241022",
      max_tokens: 1000,
      messages,
      tools: this.tools,
    });

    const finalText = [];

    for (const content of response.content) {
      if (content.type === "text") {
        finalText.push(content.text);
      } else if (content.type === "tool_use") {
        const toolName = content.name;
        const toolArgs = content.input as { [x: string]: unknown } | undefined;

        const result = await this.mcp.callTool({
          name: toolName,
          arguments: toolArgs,
        });
        finalText.push(
          `[Calling tool ${toolName} with args ${JSON.stringify(toolArgs)}]`
        );

        messages.push({ // This should be assistant's tool use message
          role: "assistant",
          content: [content] // Original tool_use content block
        });
        messages.push({ // This is the user's tool_result message
          role: "user",
          content: [{ // Content must be an array for tool_result
            type: "tool_result",
            tool_use_id: content.id,
            content: result.content as string // Assuming result.content is string or string[]
          }],
        });
        
        const nextResponse = await this.anthropic.messages.create({ // Renamed
          model: "claude-3-5-sonnet-20241022",
          max_tokens: 1000,
          messages,
          tools: this.tools, // Pass tools again
        });

        finalText.push(
          nextResponse.content[0].type === "text" ? nextResponse.content[0].text : ""
        );
      }
    }

    return finalText.join("\n");
  }
```

#### Interactive Chat Interface

Now we'll add the chat loop and cleanup functionality:

```typescript
  async chatLoop() {
    const rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout,
    });

    try {
      console.log("\nMCP Client Started!");
      console.log("Type your queries or 'quit' to exit.");

      while (true) {
        const message = await rl.question("\nQuery: ");
        if (message.toLowerCase() === "quit") {
          break;
        }
        const responseText = await this.processQuery(message); // Renamed
        console.log("\n" + responseText);
      }
    } finally {
      rl.close();
    }
  }

  async cleanup() {
    await this.mcp.close();
  }
```

#### Main Entry Point

Finally, we'll add the main execution logic:

```typescript
async function main() {
  if (process.argv.length < 3) {
    console.log("Usage: node build/index.js <path_to_server_script>"); // Updated usage
    return;
  }
  const mcpClient = new MCPClient();
  try {
    await mcpClient.connectToServer(process.argv[2]);
    await mcpClient.chatLoop();
  } finally {
    await mcpClient.cleanup();
    process.exit(0);
  }
}

main();
```

### Running the Client

To run your client with any MCP server:

```bash
# Build TypeScript
npm run build

# Run the client
node build/index.js path/to/server.py # python server
node build/index.js path/to/build/index.js # node server
```

> **Note:**
> If you're continuing the weather tutorial from the server quickstart, your command might look something like this: `node build/index.js .../quickstart-resources/weather-server-typescript/build/index.js`

**The client will:**

1.  Connect to the specified server
2.  List available tools
3.  Start an interactive chat session where you can:
    *   Enter queries
    *   See tool executions
    *   Get responses from Claude

### How It Works

When you submit a query:

1.  The client gets the list of available tools from the server
2.  Your query is sent to Claude along with tool descriptions
3.  Claude decides which tools (if any) to use
4.  The client executes any requested tool calls through the server
5.  Results are sent back to Claude
6.  Claude provides a natural language response
7.  The response is displayed to you

### Best practices

1.  **Error Handling**
    *   Use TypeScript's type system for better error detection
    *   Wrap tool calls in try-catch blocks
    *   Provide meaningful error messages
    *   Gracefully handle connection issues
2.  **Security**
    *   Store API keys securely in `.env`
    *   Validate server responses
    *   Be cautious with tool permissions

### Troubleshooting

#### Server Path Issues
*   Double-check the path to your server script is correct
*   Use the absolute path if the relative path isn't working
*   For Windows users, make sure to use forward slashes (/) or escaped backslashes (\\) in the path
*   Verify the server file has the correct extension (.js for Node.js or .py for Python)

Example of correct path usage:
```bash
# Relative path
node build/index.js ./server/build/index.js

# Absolute path
node build/index.js /Users/username/projects/mcp-server/build/index.js

# Windows path (either format works)
node build/index.js C:/projects/mcp-server/build/index.js
node build/index.js C:\\projects\\mcp-server\\build\\index.js
```

#### Response Timing
*   The first response might take up to 30 seconds to return
*   This is normal and happens while:
    *   The server initializes
    *   Claude processes the query
    *   Tools are being executed
*   Subsequent responses are typically faster
*   Don't interrupt the process during this initial waiting period

#### Common Error Messages
If you see:
*   `Error: Cannot find module`: Check your build folder and ensure TypeScript compilation succeeded
*   `Connection refused`: Ensure the server is running and the path is correct
*   `Tool execution failed`: Verify the tool's required environment variables are set
*   `ANTHROPIC_API_KEY is not set`: Check your .env file and environment variables
*   `TypeError`: Ensure you're using the correct types for tool arguments

### Testing with a Simple Echo Server

Testing your MCP client's core functionality can be streamlined using a local "echo" server. This server would feature a basic tool, like `echo_tool`, which returns its input.

**Benefits:**
*   **Isolated Testing:** Validate client logic independently of complex external services.
*   **Quick Verification:** Confirm connection, tool listing, request sending, and response processing.
*   **Debugging:** Simplify troubleshooting of request/response handling.

**Example Echo Server (TypeScript with FastMCP):**
You can create a simple echo server using a framework like FastMCP.
```typescript
// echoServer.ts
import { FastMCP, ToolContext, z } from 'fastmcp'; // Ensure FastMCP is installed

const mcp = new FastMCP();

mcp.tool({
  name: 'echo_tool',
  description: 'Echoes back the input message.',
  inputSchema: z.object({
    message: z.string().describe('The message to echo.'),
  }),
  execute: async (args, ctx: ToolContext) => {
    // await ctx.log.info(`Echoing message: ${args.message}`); // Optional logging
    return { echoed_message: args.message };
  },
});

if (require.main === module) {
  mcp.run(); // Or your specific run command for FastMCP TS
}
```
Save this as `echoServer.ts`, compile it (e.g., `tsc echoServer.ts`), and ensure its output (e.g., `build/echoServer.js`) is runnable.

**Connecting Your Client:**
Update your client's connection command to target this local server:
```bash
# First build your echo server if needed (e.g., tsc echoServer.ts)
# Then run the client
node build/index.js path/to/your/build/echoServer.js
```

**Testing Client Logic:**
After connecting, your client should list `echo_tool`. A query like "Use echo_tool to send back 'test message'" (if using an LLM to drive tool use) should trigger the `echo_tool`. Verify your client correctly processes the tool call and receives the echoed response (e.g., `{'echoed_message': 'test message'}`). This confirms the client-server interaction loop.

---

## Java Client Development

> **Note:**
> This is a quickstart demo based on Spring AI MCP auto-configuration and boot starters.
> To learn how to create sync and async MCP Clients manually, consult the [Java SDK Client](/sdk/java/mcp-client) documentation.

This example demonstrates how to build an interactive chatbot that combines Spring AI's Model Context Protocol (MCP) with the [Brave Search MCP Server](https://github.com/modelcontextprotocol/servers-archived/tree/main/src/brave-search). The application creates a conversational interface powered by Anthropic's Claude AI model that can perform internet searches through Brave Search, enabling natural language interactions with real-time web data.
[You can find the complete code for this tutorial here.](https://github.com/spring-projects/spring-ai-examples/tree/main/model-context-protocol/web-search/brave-chatbot)

### System Requirements

Before starting, ensure your system meets these requirements:

*   Java 17 or higher
*   Maven 3.6+
*   npx package manager
*   Anthropic API key (Claude)
*   Brave Search API key

### Setting Up Your Environment

1.  Install npx (Node Package eXecute):
    First, make sure to install [npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
    and then run:
    ```bash
    npm install -g npx
    ```
2.  Clone the repository:
    ```bash
    git clone https://github.com/spring-projects/spring-ai-examples.git
    cd spring-ai-examples/model-context-protocol/web-search/brave-chatbot 
    # Corrected path
    ```
3.  Set up your API keys:
    ```bash
    export ANTHROPIC_API_KEY='your-anthropic-api-key-here'
    export BRAVE_API_KEY='your-brave-api-key-here'
    ```
4.  Build the application:
    ```bash
    ./mvnw clean install
    ```
5.  Run the application using Maven:
    ```bash
    ./mvnw spring-boot:run
    ```

> **Warning:**
> Make sure you keep your `ANTHROPIC_API_KEY` and `BRAVE_API_KEY` keys secure!

### How it Works

The application integrates Spring AI with the Brave Search MCP server through several components:

#### MCP Client Configuration

1.  Required dependencies in `pom.xml`:
    ```xml
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-mcp-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-starter-model-anthropic</artifactId>
    </dependency>
    ```
2.  Application properties (`application.yml`):
    ```yml
    spring:
      ai:
        mcp:
          client:
            enabled: true
            name: brave-search-client
            version: 1.0.0
            type: SYNC
            request-timeout: 20s
            stdio:
              root-change-notification: true
              servers-configuration: classpath:/mcp-servers-config.json
            toolcallback:
              enabled: true
        anthropic:
          api-key: ${ANTHROPIC_API_KEY}
    ```
    This activates the `spring-ai-starter-mcp-client` to create one or more `McpClient`s based on the provided server configuration.
    The `spring.ai.mcp.client.toolcallback.enabled=true` property enables the tool callback mechanism, that automatically registers all MCP tool as spring ai tools.
    It is disabled by default.

3.  MCP Server Configuration (`mcp-servers-config.json`):
    ```json
    {
      "mcpServers": {
        "brave-search": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-brave-search"],
          "env": {
            "BRAVE_API_KEY": "<PUT YOUR BRAVE API KEY>"
          }
        }
      }
    }
    ```

#### Chat Implementation

The chatbot is implemented using Spring AI's `ChatClient` with MCP tool integration:

```java
var chatClient = chatClientBuilder
    .defaultSystem("You are useful assistant, expert in AI and Java.")
    .defaultToolCallbacks((Object[]) mcpToolAdapter.toolCallbacks())
    .defaultAdvisors(new MessageChatMemoryAdvisor(new InMemoryChatMemory()))
    .build();
```

> **Warning:**
> Breaking change: From SpringAI 1.0.0-M8 onwards, use `.defaultToolCallbacks(...)` instead of `.defaultTool(...)` to register MCP tools.

Key features:
*   Uses Claude AI model for natural language understanding
*   Integrates Brave Search through MCP for real-time web search capabilities
*   Maintains conversation memory using `InMemoryChatMemory`
*   Runs as an interactive command-line application

#### Build and run

```bash
./mvnw clean install
java -jar ./target/ai-mcp-brave-chatbot-0.0.1-SNAPSHOT.jar
```
or
```bash
./mvnw spring-boot:run
```

The application will start an interactive chat session where you can ask questions. The chatbot will use Brave Search when it needs to find information from the internet to answer your queries.

The chatbot can:
*   Answer questions using its built-in knowledge
*   Perform web searches when needed using Brave Search
*   Remember context from previous messages in the conversation
*   Combine information from multiple sources to provide comprehensive answers

#### Advanced Configuration

The MCP client supports additional configuration options:
*   Client customization through `McpSyncClientCustomizer` or `McpAsyncClientCustomizer`
*   Multiple clients with multiple transport types: `STDIO` and `SSE` (Server-Sent Events)
*   Integration with Spring AI's tool execution framework
*   Automatic client initialization and lifecycle management

For WebFlux-based applications, you can use the WebFlux starter instead:
```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-mcp-client-webflux-spring-boot-starter</artifactId>
</dependency>
```
This provides similar functionality but uses a WebFlux-based SSE transport implementation, recommended for production deployments.

### Testing with a Simple Echo Server

For testing your Java MCP client, especially when using Spring AI, you can create a minimal local MCP server. This "echo" server would expose a simple tool (e.g., `echo_tool`) that returns its input.

**Benefits:**
*   **Focused Client Testing:** Test client connection, tool discovery, and invocation logic without external dependencies.
*   **Rapid Iteration:** Quickly verify client-side handling of MCP interactions.
*   **Simplified Debugging:** Isolate issues in the client's request/response cycle.

**Example Echo Server (Conceptual for Spring AI):**
If you're using Spring AI, you could create a simple Spring Boot application with an MCP server configuration that defines an `echo_tool`.
```java
// Conceptual EchoTool.java
// package com.example.echoserver;
// import org.springframework.ai.model.function.FunctionCallback;
// import org.springframework.context.annotation.Bean;
// import org.springframework.context.annotation.Configuration;
// import java.util.function.Function;
//
// @Configuration
// public class EchoToolConfig {
//     @Bean
//     public FunctionCallback echoFunction() {
//         return FunctionCallback.builder("echo_tool")
//             .withDescription("Echoes back the input message.")
//             .withInputType(EchoRequest.class) // Define EchoRequest record/class
//             .withResponseProcessor(response -> ((EchoRequest) response).message())
//             .withFunction(request -> "Echo: " + request.message()) // Simplified logic
//             .build();
//     }
// }
// record EchoRequest(String message) {}
```
You would then configure your `mcp-servers-config.json` to point to this local Spring Boot echo server. For a non-Spring AI Java client, you'd implement a basic MCP server using the chosen Java MCP SDK.

**Connecting Your Client:**
Adjust your client's server configuration (e.g., in `mcp-servers-config.json` or programmatically) to connect to your local echo server. If the echo server is a Spring Boot app, you'd run it and configure your client to connect to its MCP endpoint.

**Testing Client Logic:**
With the client connected to the echo server, it should discover the `echo_tool`. You can then craft a prompt for your LLM (if applicable) or directly make a tool call via your client's API to use `echo_tool`. Verify that the client receives the expected echoed response, confirming the basic MCP communication flow.

---

## Kotlin Client Development

[You can find the complete code for this tutorial here.](https://github.com/modelcontextprotocol/kotlin-sdk/tree/main/samples/kotlin-mcp-client)

### System Requirements

Before starting, ensure your system meets these requirements:

*   Java 17 or higher
*   Anthropic API key (Claude)

### Setting up your environment

First, let's install `java` and `gradle` if you haven't already.
You can download `java` from [official Oracle JDK website](https://www.oracle.com/java/technologies/downloads/).
Verify your `java` installation:
```bash
java --version
```

Now, let's create and set up your project:

**MacOS/Linux:**
```bash
# Create a new directory for our project
mkdir kotlin-mcp-client
cd kotlin-mcp-client

# Initialize a new kotlin project
gradle init
```

**Windows:**
```powershell
# Create a new directory for our project
md kotlin-mcp-client
cd kotlin-mcp-client
# Initialize a new kotlin project
gradle init
```

After running `gradle init`, you will be presented with options for creating your project.
Select **Application** as the project type, **Kotlin** as the programming language, and **Java 17** as the Java version.

Alternatively, you can create a Kotlin application using the [IntelliJ IDEA project wizard](https://kotlinlang.org/docs/jvm-get-started.html).

After creating the project, add the following dependencies:

**build.gradle.kts:**
```kotlin
val mcpVersion = "0.4.0" // Check for latest version
val slf4jVersion = "2.0.9"
val anthropicVersion = "0.8.0" // Check for latest version

dependencies {
    implementation("io.modelcontextprotocol:kotlin-sdk:$mcpVersion")
    implementation("org.slf4j:slf4j-nop:$slf4jVersion") // Or your preferred SLF4J binding
    implementation("com.anthropic:anthropic-java:$anthropicVersion")
}
```

**build.gradle (Groovy):**
```groovy
def mcpVersion = '0.4.0' // Check for latest version
def slf4jVersion = '2.0.9'
def anthropicVersion = '0.8.0' // Check for latest version
dependencies {
    implementation "io.modelcontextprotocol:kotlin-sdk:$mcpVersion"
    implementation "org.slf4j:slf4j-nop:$slf4jVersion" // Or your preferred SLF4J binding
    implementation "com.anthropic:anthropic-java:$anthropicVersion"
}
```

Also, add the following plugins to your build script:

**build.gradle.kts:**
```kotlin
plugins {
    id("com.github.johnrengelman.shadow") version "8.1.1"
}
```

**build.gradle (Groovy):**
```groovy
plugins {
    id 'com.github.johnrengelman.shadow' version '8.1.1'
}
```

### Setting up your API key

You'll need an Anthropic API key from the [Anthropic Console](https://console.anthropic.com/settings/keys).

Set up your API key:
```bash
export ANTHROPIC_API_KEY='your-anthropic-api-key-here'
```

> **Warning:**
> Make sure you keep your `ANTHROPIC_API_KEY` secure!

### Creating the Client

#### Basic Client Structure

First, let's create the basic client class in `src/main/kotlin/your/package/App.kt` (or similar):

```kotlin
package your.package // Replace with your package name

import com.anthropic.AnthropicOkHttpClient
import com.anthropic.api.Tool
import com.anthropic.api.ToolUnion
import com.anthropic.api.MessageParam
import com.anthropic.api.MessageCreateParams
import com.anthropic.api.Model
import com.anthropic.api.TextContent
import com.fasterxml.jackson.core.type.TypeReference
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.databind.JsonNode // Import JsonNode
import io.modelcontextprotocol.client.Client
import io.modelcontextprotocol.client.stdio.StdioClientTransport
import io.modelcontextprotocol.common.Implementation
import io.modelcontextprotocol.common.JsonObject // Assuming this is your JsonObject
import kotlinx.coroutines.runBlocking
import okio.buffer
import okio.sink
import okio.source
import com.anthropic.api.JsonValue // Import JsonValue

class MCPClient : AutoCloseable {
    private val anthropic = AnthropicOkHttpClient.fromEnv()
    private val mcp: Client = Client(clientInfo = Implementation(name = "mcp-client-cli", version = "1.0.0"))
    private lateinit var tools: List<ToolUnion>

    // methods will go here

    override fun close() {
        runBlocking {
            mcp.close()
            anthropic.close()
        }
    }
    // Helper function
    private fun JsonObject?.toJsonValue(): JsonValue {
        if (this == null) return JsonValue.from(emptyMap<String, Any>()) // Handle null case
        val mapper = ObjectMapper()
        val node: JsonNode = mapper.readTree(this.toString())
        return JsonValue.fromJsonNode(node)
    }
}
```

#### Server connection management

Next, we'll implement the method to connect to an MCP server:

```kotlin
    suspend fun connectToServer(serverScriptPath: String) {
        try {
            val command = buildList {
                when (serverScriptPath.substringAfterLast(".")) {
                    "js" -> add("node")
                    "py" -> add(if (System.getProperty("os.name").lowercase().contains("win")) "python" else "python3")
                    "jar" -> addAll(listOf("java", "-jar"))
                    else -> throw IllegalArgumentException("Server script must be a .js, .py or .jar file")
                }
                add(serverScriptPath)
            }

            val process = ProcessBuilder(command).start()
            val transport = StdioClientTransport(
                input = process.inputStream.asSource().buffered(),
                output = process.outputStream.asSink().buffered()
            )

            mcp.connect(transport)

            val toolsResult = mcp.listTools()
            tools = toolsResult?.tools?.map { tool ->
                ToolUnion.ofTool(
                    Tool.builder()
                        .name(tool.name)
                        .description(tool.description ?: "")
                        .inputSchema(
                            Tool.InputSchema.builder()
                                .type(tool.inputSchema.type.toJsonValue()) // Use helper
                                .properties(tool.inputSchema.properties.toJsonValue()) // Use helper
                                .putAdditionalProperty("required", tool.inputSchema.required.toJsonValue()) // Use helper
                                .build()
                        )
                        .build()
                )
            } ?: emptyList()
            println("Connected to server with tools: ${tools.joinToString(", ") { it.tool().get().name() }}")
        } catch (e: Exception) {
            println("Failed to connect to MCP server: $e")
            throw e
        }
    }
```

#### Query processing logic

Now let's add the core functionality for processing queries and handling tool calls:

```kotlin
    private val messageParamsBuilder: MessageCreateParams.Builder = MessageCreateParams.builder()
        .model(Model.CLAUDE_3_5_SONNET_20241022)
        .maxTokens(1024)

    suspend fun processQuery(query: String): String {
        val messages = mutableListOf(
            MessageParam.builder()
                .role(MessageParam.Role.USER)
                .content(query)
                .build()
        )

        val response = anthropic.messages().create(
            messageParamsBuilder
                .messages(messages)
                .tools(tools)
                .build()
        )

        val finalText = mutableListOf<String>()
        response.content().forEach { content ->
            when {
                content.isText -> finalText.add(content.text().getOrNull()?.text() ?: "")

                content.isToolUse -> {
                    val toolUse = content.toolUse().get()
                    val toolName = toolUse.name()
                    val toolArgs: Map<String, Any?>? = toolUse._input()?.convert(object : TypeReference<Map<String, Any?>>() {})


                    val result = mcp.callTool(
                        name = toolName,
                        arguments = toolArgs?.mapValues { it.value.toString() } ?: emptyMap() // Convert to Map<String, String>
                    )
                    finalText.add("[Calling tool $toolName with args $toolArgs]")
                    
                    messages.add(MessageParam.builder().role(MessageParam.Role.ASSISTANT).content(listOf(content)).build())
                    messages.add(
                        MessageParam.builder()
                            .role(MessageParam.Role.USER)
                            .content(listOf(com.anthropic.api.Content.builder().type("tool_result").toolUseId(toolUse.id()).content(
                                result?.content?.joinToString("\n") { if (it is TextContent) it.text ?: "" else "" } ?: ""
                            ).build()))
                            .build()
                    )

                    val aiResponse = anthropic.messages().create(
                        messageParamsBuilder
                            .messages(messages)
                            .tools(tools) // Pass tools again
                            .build()
                    )

                    finalText.add(aiResponse.content().firstOrNull()?.text()?.getOrNull()?.text() ?: "")
                }
            }
        }
        return finalText.joinToString("\n")
    }
```

#### Interactive chat

We'll add the chat loop:

```kotlin
    suspend fun chatLoop() {
        println("\nMCP Client Started!")
        println("Type your queries or 'quit' to exit.")

        while (true) {
            print("\nQuery: ")
            val message = readlnOrNull() ?: break // Changed to readlnOrNull
            if (message.lowercase() == "quit") break
            val responseText = processQuery(message) // Renamed
            println("\n$responseText")
        }
    }
```

#### Main entry point

Finally, we'll add the main execution function:

```kotlin
// In your App.kt or Main.kt
fun main(args: Array<String>) = runBlocking {
    if (args.isEmpty()) {
        println("Usage: java -jar <your_path>/build/libs/kotlin-mcp-client-all.jar <path_to_server_script>") // Updated usage
        return@runBlocking
    }
    val serverPath = args.first()
    val client = MCPClient()
    client.use {
        it.connectToServer(serverPath) // Changed to 'it'
        it.chatLoop() // Changed to 'it'
    }
}
```

### Running the client

To run your client with any MCP server:

```bash
./gradlew shadowJar # Use shadowJar to create an executable jar

# Run the client
java -jar build/libs/<your-jar-name>-all.jar path/to/server.jar # jvm server
java -jar build/libs/<your-jar-name>-all.jar path/to/server.py # python server
java -jar build/libs/<your-jar-name>-all.jar path/to/build/index.js # node server
```

> **Note:**
> If you're continuing the weather tutorial from the server quickstart, your command might look something like this: `java -jar build/libs/kotlin-mcp-client-all.jar .../samples/weather-stdio-server/build/libs/weather-stdio-server-all.jar`

**The client will:**

1.  Connect to the specified server
2.  List available tools
3.  Start an interactive chat session where you can:
    *   Enter queries
    *   See tool executions
    *   Get responses from Claude

### How it works

Here's a high-level workflow schema:

```mermaid
---
config:
    theme: neutral
---
sequenceDiagram
    actor User
    participant Client
    participant Claude
    participant MCP_Server as MCP Server
    participant Tools

    User->>Client: Send query
    Client->>MCP_Server: Get available tools
    Client->>Claude: Send query with tool descriptions
    Claude-->>Client: Decide tool execution
    Client->>MCP_Server: Request tool execution
    MCP_Server->>Tools: Execute chosen tools
    Tools-->>MCP_Server: Return results
    MCP_Server-->>Client: Send results
    Client->>Claude: Send tool results
    Claude-->>Client: Provide final response
    Client-->>User: Display response
```

When you submit a query:

1.  The client gets the list of available tools from the server
2.  Your query is sent to Claude along with tool descriptions
3.  Claude decides which tools (if any) to use
4.  The client executes any requested tool calls through the server
5.  Results are sent back to Claude
6.  Claude provides a natural language response
7.  The response is displayed to you

### Best practices

1.  **Error Handling**
    *   Leverage Kotlin's type system to model errors explicitly
    *   Wrap external tool and API calls in `try-catch` blocks when exceptions are possible
    *   Provide clear and meaningful error messages
    *   Handle network timeouts and connection issues gracefully
2.  **Security**
    *   Store API keys and secrets securely in `local.properties`, environment variables, or secret managers
    *   Validate all external responses to avoid unexpected or unsafe data usage
    *   Be cautious with permissions and trust boundaries when using tools

### Troubleshooting

#### Server Path Issues
*   Double-check the path to your server script is correct
*   Use the absolute path if the relative path isn't working
*   For Windows users, make sure to use forward slashes (/) or escaped backslashes (\\) in the path
*   Make sure that the required runtime is installed (java for Java, npm for Node.js, or uv for Python)
*   Verify the server file has the correct extension (.jar for Java, .js for Node.js or .py for Python)

Example of correct path usage:
```bash
# Relative path
java -jar build/libs/client-all.jar ./server/build/libs/server-all.jar

# Absolute path
java -jar build/libs/client-all.jar /Users/username/projects/mcp-server/build/libs/server-all.jar

# Windows path (either format works)
java -jar build/libs/client-all.jar C:/projects/mcp-server/build/libs/server-all.jar
java -jar build/libs/client-all.jar C:\\projects\\mcp-server\\build\\libs\\server-all.jar
```

#### Response Timing
*   The first response might take up to 30 seconds to return
*   This is normal and happens while:
    *   The server initializes
    *   Claude processes the query
    *   Tools are being executed
*   Subsequent responses are typically faster
*   Don't interrupt the process during this initial waiting period

#### Common Error Messages
If you see:
*   `Connection refused`: Ensure the server is running and the path is correct
*   `Tool execution failed`: Verify the tool's required environment variables are set
*   `ANTHROPIC_API_KEY is not set`: Check your environment variables

### Testing with a Simple Echo Server

To effectively test your Kotlin MCP client, consider implementing a basic local "echo" server. This server would offer a single tool, such as `echo_tool`, designed to simply return any input it receives.

**Benefits:**
*   **Targeted Client Validation:** Test your client's connection, tool discovery, and invocation mechanisms in isolation.
*   **Efficient Iteration:** Quickly confirm client-side handling of MCP interactions without external complexities.
*   **Streamlined Debugging:** Isolate and resolve issues within the client's request/response lifecycle more easily.

**Example Echo Server (Conceptual Kotlin):**
You can implement a minimal MCP server using the Kotlin MCP SDK.
```kotlin
// conceptual_echo_server.kt
// package com.example.mcp.echo
//
// import io.modelcontextprotocol.server.* // Use actual imports from Kotlin MCP SDK
// import kotlinx.coroutines.runBlocking
//
// // Define input/output structures if your SDK uses them
// data class EchoInput(val message: String)
// data class EchoOutput(val echoed_message: String)
//
// fun main() = runBlocking {
//     val server = StdioServer { // Or your SDK's server builder
//         tool<EchoInput, EchoOutput>(
//             name = "echo_tool",
//             description = "Echoes back the input message.",
//             inputSchema = EchoInput::class, // Or schema definition
//         ) { input ->
//             EchoOutput(input.message)
//         }
//     }
//     server.run() // Start the server
// }
```
Compile this into a JAR (e.g., using `./gradlew shadowJar`).

**Connecting Your Client:**
Modify your client's connection command to point to this local echo server JAR:
```bash
java -jar build/libs/<your-client-jar-name>-all.jar path/to/your/echo_server-all.jar
```

**Testing Client Logic:**
Once your client connects, it should list `echo_tool`. If your client uses an LLM, a query like "Use echo_tool to echo 'Kotlin test'" should activate the tool. Verify that your client correctly handles the tool call and receives the echoed response (e.g., `{"echoed_message": "Kotlin test"}`). This confirms the fundamental client-server communication.

---

## C# Client Development

[You can find the complete code for this tutorial here.](https://github.com/modelcontextprotocol/csharp-sdk/tree/main/samples/QuickstartClient)

### System Requirements

Before starting, ensure your system meets these requirements:

*   .NET 8.0 or higher
*   Anthropic API key (Claude)
*   Windows, Linux, or MacOS

### Setting up your environment

First, create a new .NET project:
```bash
dotnet new console -n QuickstartClient
cd QuickstartClient
```

Then, add the required dependencies to your project:
```bash
dotnet add package ModelContextProtocol --prerelease
dotnet add package Anthropic.SDK
dotnet add package Microsoft.Extensions.Hosting
dotnet add package Microsoft.Extensions.AI
```

### Setting up your API key

You'll need an Anthropic API key from the [Anthropic Console](https://console.anthropic.com/settings/keys).
```bash
dotnet user-secrets init
dotnet user-secrets set "ANTHROPIC_API_KEY" "<your key here>"
```

### Creating the Client

#### Basic Client Structure

First, let's setup the basic client class in the file `Program.cs`:
```csharp
using Anthropic.SDK;
using Microsoft.Extensions.AI;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using ModelContextProtocol.Client;
using ModelContextProtocol.Protocol.Transport; // Ensure this namespace is correct for StdioClientTransport
using System.IO; // For Directory, File
using System; // For Console, StringComparison, NotSupportedException

var builder = Host.CreateApplicationBuilder(args);

builder.Configuration
    .AddEnvironmentVariables()
    .AddUserSecrets<Program>();
```
This creates the beginnings of a .NET console application that can read the API key from user secrets.

Next, we'll setup the MCP Client:
```csharp
var (command, arguments) = GetCommandAndArguments(args);

var clientTransport = new StdioClientTransport(new()
{
    Name = "Demo Server",
    Command = command,
    Arguments = arguments,
});

await using var mcpClient = await McpClientFactory.CreateAsync(clientTransport);

var tools = await mcpClient.ListToolsAsync();
foreach (var tool in tools)
{
    Console.WriteLine($"Connected to server with tools: {tool.Name}");
}
```

Add this function at the end of the `Program.cs` file:
```csharp
static (string command, string[] arguments) GetCommandAndArguments(string[] args)
{
    if (args.Length == 0) throw new ArgumentException("Server script path not provided.");
    var scriptPath = args[0]; // Assuming the script path is the first argument
    string[] scriptArgs = args.Skip(1).ToArray(); // Remaining args are for the script itself

    return scriptPath switch
    {
        var script when script.EndsWith(".py") => ("python", [script, ..scriptArgs]),
        var script when script.EndsWith(".js") => ("node", [script, ..scriptArgs]),
        var script when Directory.Exists(script) || (File.Exists(script) && script.EndsWith(".csproj")) => ("dotnet", ["run", "--project", script, "--no-build", ..scriptArgs]),
        _ => throw new NotSupportedException("An unsupported server script was provided. Supported scripts are .py, .js, or .csproj")
    };
}
```
This creates a MCP client that will connect to a server that is provided as a command line argument. It then lists the available tools from the connected server.

### Testing with a Simple Echo Server

For testing your C# MCP client, creating a minimal local "echo" server is highly beneficial. This server would expose a basic tool, for instance, `echo_tool`, which simply returns any input it receives.

**Benefits:**
*   **Focused Client Validation:** Test client connection, tool discovery, and invocation logic without relying on external services.
*   **Rapid Iteration Cycles:** Quickly verify how your client handles MCP interactions.
*   **Simplified Debugging:** Isolate issues in the client's request/response handling more effectively.

**Example Echo Server (Conceptual C#):**
You can implement a simple MCP server using the C# MCP SDK.
```csharp
// Conceptual EchoServerProject/Program.cs
// using ModelContextProtocol.Server; // Use actual imports from C# MCP SDK
// using System.Threading.Tasks;
//
// public class EchoTool : McpTool<EchoTool.Input, EchoTool.Output>
// {
//     public override string Name => "echo_tool";
//     public override string Description => "Echoes back the input message.";
//     public override object InputSchema => new { type = "object", properties = new { message = new { type = "string" } }, required = new[] { "message" } };
//
//     public class Input { public string Message { get; set; } }
//     public class Output { public string EchoedMessage { get; set; } }
//
//     public override Task<Output> ExecuteAsync(Input input, McpToolContext context)
//     {
//         // await context.Logger.LogInformationAsync($"Echoing: {input.Message}"); // Optional logging
//         return Task.FromResult(new Output { EchoedMessage = input.Message });
//     }
// }
//
// public class EchoServer
// {
//     public static async Task Main(string[] args)
//     {
//         var server = new McpServerBuilder() // Or your SDK's server builder
//             .AddTool(new EchoTool())
//             .BuildStdioServer();
//         await server.RunAsync(); // Start the server
//     }
// }
```
Build this as a separate console project (e.g., `EchoServerProject.csproj`).

**Connecting Your Client:**
Modify your client's startup command to point to this local echo server project:
```bash
dotnet run -- path/to/your/EchoServerProject/EchoServerProject.csproj
```

**Testing Client Logic:**
After your client connects, it should discover the `echo_tool`. If your client integrates with an LLM, a query like "Use echo_tool to echo 'C# test'" should trigger the tool. Verify that your client correctly processes the tool call and receives the echoed response (e.g., a JSON object like `{"echoedMessage": "C# test"}`). This confirms the basic client-server communication flow.

#### Query processing logic

Now let's add the core functionality for processing queries and handling tool calls:
```csharp
using var anthropicSdkClient = new AnthropicClient(new APIAuthentication(builder.Configuration["ANTHROPIC_API_KEY"]));
var chatClient = anthropicSdkClient.Messages
    .AsBuilder()
    .UseFunctionInvocation(mcpClient) // Pass McpClient for tool execution
    .Build();


var options = new ChatOptions
{
    MaxOutputTokens = 1000,
    ModelId = "claude-3-5-sonnet-20241022",
    Tools = [.. tools.Select(t => t.ToTool())] // Convert MCP tools to Anthropic SDK tools
};

Console.ForegroundColor = ConsoleColor.Green;
Console.WriteLine("MCP Client Started!");
Console.ResetColor();

PromptForInput();
while(Console.ReadLine() is string query && !"exit".Equals(query, StringComparison.OrdinalIgnoreCase))
{
    if (string.IsNullOrWhiteSpace(query))
    {
        PromptForInput();
        continue;
    }

    await foreach (var message in chatClient.GetStreamingResponseAsync(query, options))
    {
        Console.Write(message);
    }
    Console.WriteLine();

    PromptForInput();
}

static void PromptForInput()
{
    Console.WriteLine("Enter a command (or 'exit' to quit):");
    Console.ForegroundColor = ConsoleColor.Cyan;
    Console.Write("> ");
    Console.ResetColor();
}
```

### Key Components Explained

#### 1. Client Initialization
*   The client is initialized using `McpClientFactory.CreateAsync()`, which sets up the transport type and command to run the server.

#### 2. Server Connection
*   Supports Python, Node.js, and .NET servers.
*   The server is started using the command specified in the arguments.
*   Configures to use stdio for communication with the server.
*   Initializes the session and available tools.

#### 3. Query Processing
*   Leverages [Microsoft.Extensions.AI](https://learn.microsoft.com/dotnet/ai/ai-extensions) for the chat client.
*   Configures the `IChatClient` to use automatic tool (function) invocation.
*   The client reads user input and sends it to the server.
*   The server processes the query and returns a response.
*   The response is displayed to the user.

### Running the Client

To run your client with any MCP server:
```bash
dotnet run -- path/to/server.csproj # dotnet server
dotnet run -- path/to/server.py # python server
dotnet run -- path/to/server.js # node server
```

> **Note:**
> If you're continuing the weather tutorial from the server quickstart, your command might look something like this: `dotnet run -- path/to/QuickstartWeatherServer`.

The client will:
1.  Connect to the specified server
2.  List available tools
3.  Start an interactive chat session where you can:
    *   Enter queries
    *   See tool executions
    *   Get responses from Claude
4.  Exit the session when done

Here's an example of what it should look like it connected to a weather server quickstart:

![.NET Client Quickstart](https://mintlify.s3.us-west-1.amazonaws.com/mcp/images/quickstart-dotnet-client.png)

---

## Next steps

*   **Example servers:** Check out our gallery of official MCP servers and implementations.
*   **Clients:** View the list of clients that support MCP integrations.
*   **Building MCP with LLMs:** Learn how to use LLMs like Claude to speed up your MCP development.
*   **Core architecture:** Understand how MCP connects clients, servers, and LLMs.
*   **MCP Specification:** Read the official [Model Context Protocol specification](https://modelcontextprotocol.io/specification/2025-06-18.md).

(Note: Links from original CardGroup would need to be actual URLs if this were live documentation.)
