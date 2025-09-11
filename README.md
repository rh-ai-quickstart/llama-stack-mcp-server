# Llama Stack with MCP Server

Welcome to the Llama Stack with MCP Server Quickstart!

Use this to quickly deploy Llama 3.2-3B on vLLM with Llama Stack and MCP servers in your OpenShift AI environment.

To see how it's done, jump straight to [installation](#install).

## Table of Contents

1. [Description](#description)
2. [Custom MCP Server](#custom-mcp-server)
3. [How MCP Servers Work with Llama Stack](#how-mcp-servers-work-with-llama-stack)
   - [MCP Server Registration](#mcp-server-registration)
   - [Tool Execution Flow](#tool-execution-flow)
4. [Architecture diagrams](#architecture-diagrams)
5. [References](#references)
6. [Prerequisites](#prerequisites)
   - [Minimum hardware requirements](#minimum-hardware-requirements)
   - [Required software](#required-software)
   - [Required permissions](#required-permissions)
7. [Install](#install)
   - [Clone the repository](#clone-the-repository)
   - [Create the project](#create-the-project)
   - [Single Command Installation (Recommended)](#single-command-installation-recommended)
8. [Test](#test)
9. [Cleanup](#cleanup)

## Description

This quickstart provides a complete setup for deploying:
- Llama 3.2-3B model using vLLM on OpenShift AI
- Llama Stack for agent-based interactions
- Sample HR application providing restful services to HR data e.g. vacation booking
- MCP Weather Server for real-time weather data access
- Custom MCP server providing access to the sample HR application

## Custom MCP Server

The custom MCP server (`custom-mcp-server/`) demonstrates how to build a Model Context Protocol server that integrates with enterprise APIs. This server provides the following tools to the LLM:

- **Vacation Management**: Check vacation balances and create vacation requests

The MCP server acts as a bridge between the Llama Stack and the HR Enterprise API, translating LLM tool calls into REST API requests.

**Source Code & Build Instructions**: If you want to modify the custom MCP server, see the complete source code and build instructions in the `custom-mcp-server/` directory. The server is built using Python and can be customized to integrate with your own enterprise APIs.

## How MCP Servers Work with Llama Stack

### MCP Server Registration

MCP servers are registered with Llama Stack through configuration. The Llama Stack server automatically discovers and connects to configured MCP servers at startup. Here's an example of how MCP servers are configured:

```yaml
# Llama Stack MCP server configuration
mcpServers:
  - name: "mcp-weather"
    uri: "http://mcp-weather:3001"
    description: "Weather data MCP server"
  - name: "hr-api-tools"
    uri: "http://custom-mcp-server:8000/sse"
    description: "HR API MCP server with employee, vacation, job, and performance tools"
```

In this example, this configuration is maintained in the `llama-stack-config` config map, part of the llama-stack helm chart.

When Llama Stack starts, it:
1. **Connects to each MCP server** via Server-Sent Events (SSE) or WebSocket
2. **Discovers available tools** by querying each server's capabilities
3. **Registers tool schemas** that describe what each tool does and its parameters
4. **Makes tools available** to the LLM for use in conversations

### Tool Execution Flow

When a user requests to use a tool, here's the complete flow:

1. **User Request**: User asks a question in the Llama Stack Playground (e.g., "What's the weather in New York?")

2. **LLM Context**: Llama Stack includes the available tool definitions in the system message sent to the LLM

3. **LLM Response**: The LLM decides to use a tool and responds with a structured tool call (e.g., `getforecast` with location parameter)

4. **Tool Execution**: Llama Stack intercepts the tool call and routes it to the appropriate MCP server

5. **MCP Processing**: The MCP server executes the tool (e.g., calls the weather API or HR database)

6. **Result Return**: The MCP server returns structured results back to Llama Stack

7. **LLM Integration**: Llama Stack provides the tool results to the LLM as context

8. **Final Response**: The LLM incorporates the tool results into a natural language response for the user

This seamless integration allows the LLM to access real-time data and perform actions while maintaining a natural conversational interface.

## Architecture diagrams

![Llama Stack with MCP Servers Architecture](assets/images/architecture-diagram.png)

## References

- [Llama Stack Documentation](https://rh-aiservices-bu.github.io/llama-stack-tutorial/)
- [Model Context Protocol (MCP) Quick Start](https://modelcontextprotocol.io/quickstart/server)
- [vLLM Documentation](https://github.com/vllm-project/vllm)
- [Red Hat OpenShift AI Documentation](https://access.redhat.com/documentation/en-us/red_hat_openshift_ai)


## Prerequisites

### Minimum hardware requirements

- 1 GPU required (NVIDIA L40, A10, or similar)
- 8+ vCPUs
- 24+ GiB RAM

### Required software

- Red Hat OpenShift
- Red Hat OpenShift AI 2.16+
- OpenShift CLI (`oc`) - [Download here](https://docs.openshift.com/container-platform/latest/cli_reference/openshift_cli/getting-started-cli.html)
- Helm CLI (`helm`) - [Download here](https://helm.sh/docs/intro/install/)


### Required permissions

- Standard user. No elevated cluster permissions required

## Install

**Please note before you start**

This example was tested on Red Hat OpenShift 4.17.30 & Red Hat OpenShift AI v2.19.0.

All components are deployed using Helm charts located in the `helm/` directory:
- `helm/llama3.2-3b/` - Llama 3.2-3B model on vLLM
- `helm/llama-stack/` - Llama Stack server
- `helm/mcp-weather/` - Weather MCP server
- `helm/llama-stack-playground/` - Playground UI
- `helm/custom-mcp-server/` - Custom HR API MCP server
- `helm/hr-api/` - HR Enterprise API
- `helm/llama-stack-mcp/` - Umbrella chart for single-command deployment

### Clone the repository

```bash
git clone https://github.com/rh-ai-quickstart/llama-stack-mcp-server.git && \
    cd llama-stack-mcp-server/
```

### Create the project

```bash
oc new-project llama-stack-mcp-demo
```

### Build and deploy the helm chart

By default the `device` variable is set to `gpu`. Change the value of `device` in these files to the desired hardware to run on [gpu, cpu, hpu]:
- helm/llama-stack/values.yaml
- helm/llama3.2-3b/values.yaml

Deploy the complete Llama Stack with MCP servers using the umbrella chart:

```bash

# Build dependencies (downloads and packages all required charts)
helm dependency build ./helm/llama-stack-mcp

# Deploy everything with a single command
helm install llama-stack-mcp ./helm/llama-stack-mcp 
```

**Note:** The `llama-stack` pod will be in `CrashLoopBackOff` status until the Llama model is fully loaded and being served. This is normal behavior as the Llama Stack server requires the model endpoint to be available before it can start successfully.

This will deploy all components including:
- Llama 3.2-3B model on vLLM
- Llama Stack server with automatic configuration
- MCP Weather Server
- HR Enterprise API  
- HR MCP Server
- Llama Stack Playground

Once the deployment is complete, you should see:

To get the playground URL:
```bash
  export PLAYGROUND_URL=$(oc get route llama-stack-playground -o jsonpath='{.spec.host}' 2>/dev/null || echo "Route not found")
  echo "Playground: https://$PLAYGROUND_URL"
```

To check the status of all components:
```bash
  helm status llama-stack-mcp
  oc get pods 
```

For troubleshooting:
```bash
  oc get pods
  oc logs -l app.kubernetes.io/name=llama-stack
  oc logs -l app.kubernetes.io/name=llama3-2-3b
  oc logs -l app.kubernetes.io/name=custom-mcp-server
  oc logs -l app.kubernetes.io/name=hr-enterprise-api
  oc logs -l app.kubernetes.io/name=mcp-weather
```

When the deployment is complete, you should see all pods running in your OpenShift console:

![OpenShift Deployment](assets/images/deployment.png)

## Test

1. Get the Llama Stack playground route:
```bash
oc get route llama-stack-playground -n llama-stack-mcp-demo
```

2. Open the playground URL in your browser (it will look something like `https://llama-stack-playground-llama-stack-mcp-demo.apps.openshift-cluster.company.com`)


3. In the playground:
   - Click on the "Tools" tab
   - Select "Weather" MCP Server from the available tools
   - In the chat interface, type: "What's the weather in New York?"

4. You should receive a response similar to:
```
🛠 Using "getforecast" tool:

The current weather in New York is mostly sunny with a temperature of 75°F and a gentle breeze coming from the southwest at 7 mph. There is a chance of showers and thunderstorms this afternoon. Tonight, the temperature will drop to 66°F with a wind coming from the west at 9 mph. The forecast for the rest of the week is mostly sunny with temperatures ranging from 69°F to 85°F. There is a slight chance of showers and thunderstorms on Thursday and Friday nights.
```

This confirms that the Llama Stack is successfully communicating with the MCP Weather Server and can process weather-related queries.

2. **Test HR API MCP server tools**:

In the playground interface:

- **Navigate to Tools**: Click on the "Tools" tab
- **Verify availability**: Look for your Internal HR tools:
  - `get_vacation_balance` - Check employee vacation balances
  - `create_vacation_request` - Submit new vacation requests

Select the "internal-hr" MCP Server

3. **Test with sample queries**:

**Test vacation balance:**
```
What is the vacation balance for employee EMP001?
```

**Test vacation request:**
```
book some annual vacation time off for EMP001 for June 8th and 9th
```

![Llama Stack Playground](assets/images/playground.png)

### Verification

Verify that your custom MCP server is working correctly:

```bash
# Check all pods are running
oc get pods

# Check the custom MCP server logs
oc logs -l app.kubernetes.io/name=custom-mcp-server

# Test the service connectivity
oc exec -it deployment/llama-stack -- curl http://custom-mcp-server/health

## Cleanup

To remove all components from OpenShift:

### Option 1: Remove umbrella chart (if using single command installation)
```bash
# Remove the complete deployment
helm uninstall llama-stack-mcp
```


### Option 2: Delete the entire project
```bash
# Delete the project and all its resources
oc delete project llama-stack-mcp-demo
```

This will remove:
- Llama 3.2-3B vLLM deployment
- Llama Stack services and playground
- MCP Weather Server
- Custom MCP Server (if deployed)
- HR Enterprise API (if deployed)
- All associated ConfigMaps, Services, Routes, and Secrets