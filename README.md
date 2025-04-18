# Guide to Creating an MCP Client-Host-Server Setup

This guide outlines how to create a system with a host ("Claude Desktop"), a client (FastAPI server with OAuth and SQLite CRUD), and an SSE-based MCP server exposing prompts, resources, and tools, as per the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) specification. It also includes instructions for connecting your MCP server to Claude Desktop.

## Overview of MCP Architecture
MCP defines a Client-Host-Server architecture:
- **Host**: Manages multiple client instances, coordinates AI/LLM integration, enforces security, and handles context aggregation.
- **Clients**: Created by the host, each maintains a one-to-one connection with a server, handling protocol negotiation and message routing.
- **Servers**: Expose resources, tools, and prompts via MCP primitives, operating independently with security constraints.

In this setup:
- **Host ("Claude Desktop")**: A desktop application integrating a FastAPI server for user interactions and an MCP client for server communication.
- **Client**: The FastAPI server, part of the host, handling user requests and database operations, using the MCP client for server interactions.
- **Server**: An SSE-based MCP server providing prompts, resources, and tools.

## Design Principles
MCP emphasizes:
1. **Easy Server Build**: Servers focus on specific capabilities, with the host handling orchestration.
2. **High Composability**: Modular components combine seamlessly via a shared protocol.
3. **Server Isolation**: Only necessary context is shared, with the host controlling cross-server interactions.
4. **Progressive Feature Addition**: Minimal core protocol with negotiated capabilities for extensibility.

## Step 1: Setting Up the MCP Server
The MCP server uses Server-Sent Events (SSE) over HTTP to stream messages, implementing JSON-RPC for MCP primitives.

### Requirements
- **Framework**: Python with FastAPI for HTTP and SSE support.
- **Primitives**:
  - **Prompts**: User-controlled templates (e.g., slash commands, menu options).
  - **Resources**: Contextual data like files or database schemas, identified by URIs ([RFC 3986](https://tools.ietf.org/html/rfc3986)).
  - **Tools**: Executable functions like database queries or API calls.
- **Transport**: Streamable HTTP with SSE ([MCP Transports](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports#streamable-http)).
- **Capabilities**: Declare `resources`, `tools`, and `completions` capabilities during initialization.

### Implementation
1. **Create the FastAPI Server**:
   - Set up a FastAPI application with an SSE endpoint (e.g., `/mcp`).
   - Use `fastapi.responses.StreamingResponse` for SSE streaming.
2. **Implement JSON-RPC Methods**:
   - **Resources** ([MCP Resources](https://modelcontextprotocol.io/specification/2025-03-26/server/resources)):
     - `resources/list`: List available resources with URIs and metadata.
     - `resources/read`: Retrieve resource content by URI.
     - Support subscriptions (`resources/subscribe`) and notifications (`notifications/resources/updated`).
   - **Tools** ([MCP Tools](https://modelcontextprotocol.io/specification/2025-03-26/server/tools)):
     - `tools/list`: List tools with names, descriptions, and input schemas.
     - `tools/call`: Execute a tool with provided parameters.
     - Support notifications (`notifications/tools/list_changed`).
   - **Prompts**: Implement methods like `prompts/list` or `prompts/execute` (assumed, as specifics are unclear).
   - **Completion API** ([MCP Completion](https://modelcontextprotocol.io/specification/2025-03-26/server/utilities/completion)):
     - `completion/complete`: Provide autocompletion for prompt arguments or resource URIs.
3. **Handle SSE**:
   - Stream JSON-RPC responses and notifications to connected clients.
   - Ensure resumability and redelivery for robust streaming.
4. **Security**:
   - Validate tool inputs and sanitize outputs.
   - Implement access controls and rate limiting.
   - Validate `Origin` headers for HTTP requests.

### Example Resource and Tool Definitions
| **Type**   | **Example**                                                                 |
|------------|-----------------------------------------------------------------------------|
| Resource   | URI: `file://localhost/documents/report.txt`, MIME: `text/plain`            |
| Tool       | Name: `query_db`, Input Schema: `{"type": "string", "query": "SELECT..."}`  |

## Step 2: Setting Up the Host ("Claude Desktop")
The host is a desktop application integrating a FastAPI server and an MCP client.

### Requirements
- **Environment**: Python-based desktop application (e.g., using Tkinter or PyQt for GUI, or run as a console app).
- **Components**:
  - **FastAPI Server**: Handles user requests with OAuth authentication and SQLite CRUD.
  - **MCP Client**: Connects to the MCP server via HTTP/SSE.
- **Functionality**: Coordinate user requests, local data operations, and server interactions.

### Implementation
1. **FastAPI Server**:
   - **Setup**:
     - Use FastAPI with `fastapi.security.OAuth2AuthorizationCodeBearer` for OAuth.
     - Connect to a local SQLite database using SQLAlchemy.
   - **Endpoints**:
     - CRUD operations (e.g., `/items/{id}` for create, read, update, delete).
     - MCP-related endpoints (e.g., `/tools/list` to fetch tools from the server).
   - **Security**:
     - Require OAuth bearer tokens for all endpoints.
     - Validate tokens against an identity provider.
2. **MCP Client**:
   - **Setup**:
     - Use `requests` for HTTP POST/GET and `sseclient-py` for SSE.
     - Connect to the MCP server’s `/mcp` endpoint.
   - **Protocol Handling**:
     - Perform capability negotiation (e.g., `{"capabilities": {"resources": {}, "tools": {}}}`).
     - Send JSON-RPC requests (e.g., `{"method": "tools/list", "params": {}}`).
     - Listen for SSE events (responses and notifications).
   - **Integration**:
     - Expose methods for the FastAPI server to call (e.g., `client.list_tools()`).
3. **Host Logic**:
   - Manage the MCP client lifecycle (create, connect, disconnect).
   - Coordinate AI/LLM integration if using Claude (optional, not specified).
   - Handle notifications (e.g., update local state on `notifications/resources/updated`).

### Example FastAPI Endpoint
```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2PasswordBearer
import sqlite3

app = FastAPI()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@app.get("/items/{item_id}")
async def read_item(item_id: int, token: str = Depends(oauth2_scheme)):
    conn = sqlite3.connect("local.db")
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM items WHERE id = ?", (item_id,))
    item = cursor.fetchone()
    conn.close()
    if item:
        return {"id": item[0], "name": item[1]}
    raise HTTPException(status_code=404, detail="Item not found")
```

## Step 3: Integrating the Components
- **Data Flow**:
  - User sends a request to the FastAPI server (e.g., `/tools/call` with tool name and parameters).
  - FastAPI server uses the MCP client to send a `tools/call` request to the MCP server.
  - MCP server processes the request and streams the result via SSE.
  - MCP client receives the result and passes it to the FastAPI server, which returns it to the user.
- **Error Handling**:
  - Handle JSON-RPC errors (e.g., `-32602` for unknown tool).
  - Implement timeouts and retries for server requests.
- **Security**:
  - FastAPI server enforces OAuth authentication.
  - MCP client validates server responses to prevent injection attacks.
  - Host enforces MCP security boundaries, sharing only necessary context.

## Example Interaction Flow
| **Step**                     | **Component**       | **Action**                                                                 |
|------------------------------|---------------------|---------------------------------------------------------------------------|
| 1. User Request              | FastAPI Server      | Receives `/tools/list` request with OAuth token.                           |
| 2. MCP Client Request        | MCP Client          | Sends `tools/list` JSON-RPC request to MCP server.                        |
| 3. Server Response           | MCP Server          | Streams tool list via SSE.                                                |
| 4. Client Processing         | MCP Client          | Receives and parses tool list, passes to FastAPI server.                  |
| 5. User Response             | FastAPI Server      | Returns tool list to user.                                                |

## Security and Best Practices
- **FastAPI Server**:
  - Use HTTPS for all endpoints.
  - Validate OAuth tokens securely.
  - Sanitize database inputs to prevent SQL injection.
- **MCP Server**:
  - Validate tool inputs and rate-limit invocations.
  - Use secure HTTP headers (e.g., `Content-Security-Policy`).
- **MCP Client**:
  - Prompt for user confirmation on sensitive tool calls.
  - Log interactions for auditing.
- **Host**:
  - Enforce MCP’s server isolation principle, sharing minimal context.
  - Regularly update capabilities based on server negotiations.

## Notes on Prompts
Prompts are user-controlled templates, but the specification lacks detailed methods (e.g., `prompts/list`). Assume they are handled similarly to resources or tools, with methods for listing and executing. The [Completion API](https://modelcontextprotocol.io/specification/2025-03-26/server/utilities/completion) supports promptI supports prompt argument autocompletion, suggesting prompts are interactive inputs. Define custom prompt templates based on your use case (e.g., `/code_review` for code analysis).

## Potential Challenges
- **Prompt Implementation**: Lack of specific protocol methods may require custom logic.
- **SSE Reliability**: Ensure robust handling of connection drops and message redelivery.
- **Claude Integration**: If "Claude Desktop" involves the Claude AI model, integrate it with the host’s AI coordination logic (not specified).

## Connecting to MCP Servers with Claude Desktop
To connect your MCP server to [Claude Desktop](https://claude.ai/download), configure the `claude_desktop_config.json` file to register your server, enabling Claude to access its resources, tools, and prompts.

### Step 1: Locate the Configuration File
Find or create the `claude_desktop_config.json` file in the appropriate directory:
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `C:\Users\<username>\AppData\Roaming\Claude\claude_desktop_config.json` (replace `<username>` with your actual username)

### Step 2: Edit the Configuration File
Add your MCP server under the `mcpServers` key. For example:

```json
{
    "mcpServers": {
        "my-mcp-server": {
            "command": "python",
            "args": [
                "--directory",
                "/absolute/path/to/your/server",
                "run",
                "server.py"
            ]
        }
    }
}
```

- **Key Fields**:
  - `"my-mcp-server"`: A unique name for your server.
  - `"command"`: The executable to run your server (e.g., `python`, `uv`).
  - `"args"`: Arguments including the absolute path to your server directory and the entry file (e.g., `server.py`).
- **Best Practices**:
  - Use absolute paths (e.g., `/Users/john/server` on macOS, `C:\\Users\\John\\server` on Windows).
  - Ensure the command and arguments match your server’s execution requirements.

### Step 3: Save and Restart Claude Desktop
Save the configuration file and restart Claude Desktop to apply the changes.

### Step 4: Verify the Connection
In Claude Desktop, look for the hammer icon in the interface, indicating that your server’s tools are available. You can now interact with your server’s resources, tools, or prompts via Claude’s interface.

### What MCP Servers Provide
MCP servers enhance Claude’s capabilities by offering:
- **Resources**: File-like data (e.g., API responses, file contents) accessible via URIs.
- **Tools**: Functions callable by Claude (with user approval), such as database queries or API calls.
- **Prompts**: Pre-written templates for specific tasks (e.g., code generation, data analysis).

### Troubleshooting
If the server fails to connect:
- Verify that all paths in `claude_desktop_config.json` are absolute and correct.
- Check server logs in:
  - **macOS**: `~/Library/Application Support/Claude/logs/`
  - **Windows**: `C:\Users\<username>\AppData\Roaming\Claude\logs\`
- Run the server command manually to diagnose errors.
- Consult the [MCP Troubleshooting Guide](https://modelcontextprotocol.io/quickstart/user#troubleshooting) for detailed debugging steps.

### Example Configuration for Windows
For a Windows-based MCP server:

```json
{
    "mcpServers": {
        "my-mcp-server": {
            "command": "python",
            "args": [
                "--directory",
                "C:\\Users\\John\\mcp-server",
                "run",
                "server.py"
            ]
        }
    }
}
```

### Additional Resources
- [Building an MCP Client](https://modelcontextprotocol.io/quickstart/client)
- [List of MCP Clients](https://modelcontextprotocol.io/clients)
- [Debugging MCP](https://modelcontextprotocol.io/docs/tools/debugging)

## Conclusion
This setup creates a modular, secure system leveraging MCP’s architecture. The FastAPI server provides a user-friendly interface, the MCP client enables server communication, and the SSE-based MCP server delivers rich AI context. By connecting to Claude Desktop, you can extend Claude’s capabilities with custom resources, tools, and prompts. Follow MCP’s design principles for maintainability and extensibility.
