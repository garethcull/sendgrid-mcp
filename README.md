# SendGrid MCP Server

A Model Context Protocol (MCP) server that exposes SendGrid Email Management and Reporting tools as MCP-compatible endpoints. This MCP Server provides endpoints for automated interaction with SendGrid — allowing AI Assistants, MCP Clients such as OpenAI’s Responses API, Cursor, or the prototypr.ai MCP Client to read, create, analyze, and manage email templates or metrics through natural language.

This server is built using Flask and can be deployed to Google Cloud Run or any environment that supports Python-based web services.

It was originally designed as part of prototypr.ai's Studio product, to help streamline email marketing creation. When you connect this MCP in prototypr.ai Studio you can import any dynamic email template to the Studio canvas, make changes with leading models such as Gemini 3 Pro and save these changes back into your Sendgrid account.

You can also use prototypr.ai Chat to pull email related stats from your Sendgrid account.

Even though this MCP Tool was built for prototypr.ai, it is compatable with other MCP remote clients. Feel free to try them out as well!

# MCP Features

This SendGrid MCP Server exposes multiple tools through the MCP protocol:

- list_email_templates – Fetches a list of available SendGrid dynamic email templates.
- get_html_template_by_id – Retrieves the HTML body for a specific template ID.
- save_email_html_template – Saves a new or updated HTML email template to SendGrid.
- get_aggregate_email_stats – Aggregates email performance metrics (delivered, opens, clicks, etc.) by day, week, or month and makes available through chat.

All endpoints are protected. A valid MCP_TOKEN must be provided as a shared secret for authorization.


# MCP Architecture

This MCP server contains two files:
1. app.py - main python file which authenticates and delegates requests to mcp_helper.py
2. mcp_helper.py - Contains all SendGrid-specific logic, including email template management and statistical report generation.

### app.py
Flask app with POST /mcp
Handles JSON-RPC notifications by returning 204 No Content
Delegates to mcp_helper for MCP method logic

### mcp_helper.py
handle_request routes initialize, tools/list, tools/call
Contains business logic for interacting with the SendGrid REST API and formatting results for LLM or MCP consumption.


# Endpoints and Protocol
JSON-RPC MCP (preferred by this server)
POST /mcp
Content-Type: application/json
Auth: Authorization: Bearer MCP_TOKEN
Methods
initialize → returns protocolVersion, serverInfo, capabilities
tools/list → returns tools with inputSchema (camelCase)
tools/call → returns result with content array
notifications/initialized → must NOT return a JSON-RPC body; respond 204

# Environment Variables

This MCP server has environment variables that need to be set in order for it to work. They are:

MCP_TOKEN – Authorization token for all incoming requests.
SENDGRID_API_KEY – The SendGrid API key with sufficient permissions to read and write templates, as well as fetch analytics.


# Local Setup
Python environment

## Create and activate a virtual environment
python3 -m venv .venv
source .venv/bin/activate

## Install dependencies
pip install -r requirements.txt

Environment variables (example)
export MCP_TOKEN="your-shared-token"
export SENDGRID_API_KEY="your-sendgrid-api-key"


## Run locally
export FLASK_APP=app.py
flask run --host 0.0.0.0 --port 8080

## Quick JSON-RPC Tests
Use Python requests to verify initialize, notifications/initialized, tools/list, and tools/call. Replace the base URL and token.

```python
import requests, json

BASE = "https://<your-cloud-run-host>/mcp"
AUTH = "Bearer <your-mcp-token>"

def rpc(method, params, id_):
    payload = {"jsonrpc":"2.0","id":id_, "method":method, "params":params}
    r = requests.post(BASE, headers={"Authorization": AUTH, "Content-Type":"application/json"}, data=json.dumps(payload))
    print(method, r.status_code)
    print(r.text[:600])
    return r

# 1) initialize
rpc("initialize", {}, "1")

# 2) tools/list
rpc("tools/list", {}, "2")

# 3) tools/call
rpc("tools/call", {
    "name":"get_aggregate_email_stats",
  "arguments":{"start_date":"2025-01-01","end_date":"2025-01-31","aggregated_by":"day"}
}, "3")
```

# OpenAI Responses API Tool Configuration

This MCP tool was initially designed to use the OpenAI Responses API. For more details about OpenAI's Responses API and MCP, please check out this cookbook: 
https://cookbook.openai.com/examples/mcp/mcp_tool_guide

Configure an MCP tool in your Responses API request. Point server_url to your /mcp endpoint and include the Authorization header.

```python
tools = [
  {
    "type": "mcp",
    "server_label": "sendgrid-mcp",
    "server_url": "https://<your-cloud-run-host>/mcp",
    "headers": { "Authorization": "Bearer <your-mcp-token>" },
    "require_approval": "never"
  }
]
```



# Adding New Tools
New MCP tools can be registered by extending handle_tools_list() in mcp_helper.py, and defining logic in handle_tool_call().

Example:



```python
{
  "name": "list_email_templates",
  "description": "Fetches available SendGrid templates",
  "inputSchema": {
    "type": "object",
    "properties": {
      "list_type": {"type":"string","description":"dynamic or legacy"}
    },
    "required": ["list_type"]
  }
}
```

Then handle it inside handle_tool_call() as:

```python
if tool_name == "list_email_templates":
    data = list_email_templates(arguments)
    return {"content":[{"type":"text","text":str(data)}]}
```

# Security Considerations

All /mcp routes require a valid MCP_TOKEN for authorization.
Ensure the SENDGRID_API_KEY used has the correct API permissions (Mail Send, Marketing, and Stats API scopes).
Avoid logging or exposing email data publicly.

# Deploying to Google Cloud Run

This MCP server is designed for easy deployment to Google Cloud Run.

1. Containerize your code using a lightweight Python base image.
2. Set environment variables within the Cloud Run service.
3. Ensure your service is configured with an Authorization header for secure access.

Here is a link to an article that helped me deploy this MCP Server to Google Cloud Run: 
https://docs.cloud.google.com/run/docs/quickstarts/build-and-deploy/deploy-python-service


# License
MIT

# Contributions & Support
Feedback, issues and PRs welcome. Due to bandwidth constraints, I can't offer any timelines for free updates to this codebase. 

If you need help customizing this MCP server, I'm available for paid consulting and freelance projects. Feel free to reach out and connect w/ me on LinkedIn:
https://www.linkedin.com/in/garethcull/

Thanks for checking out the SendGrid MCP Server! This tool helps developers and AI agents automate SendGrid analytics, reporting, and template management in minutes.

Happy Emailing!
