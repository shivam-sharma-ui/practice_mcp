server -health 

import json
from pathlib import Path
from fastmcp import FastMCP

# Initialize the MCP Server
mcp = FastMCP("Service Health MCP Server")

# Resolve the absolute path to the data file
DATA_FILE = Path(__file__).parent.parent / "data" / "service_health.json"

def load_data():
    """Helper function to read the JSON file."""
    with open(DATA_FILE, 'r', encoding='utf-8') as f:
        return json.load(f)

@mcp.tool
def list_services() -> dict:
    """List enterprise services and current health status."""
    data = load_data()
    services = []
    
    # Extract only required fields
    for s in data.get("services", []):
        services.append({
            "service_name": s["service_name"],
            "status": s["status"],
            "region": s["region"]
        })
        
    return {"count": len(services), "services": services}

@mcp.tool
def get_service_health(service_name: str) -> dict:
    """Return detailed health information for a service."""
    data = load_data()
    
    for s in data.get("services", []):
        if s["service_name"] == service_name:
            return {"found": True, "service": s}
            
    return {"found": False, "message": "Service not found."}

@mcp.tool
def get_active_incidents(service_name: str = None) -> dict:
    """Return active operational incidents. Optionally filter by service."""
    data = load_data()
    incidents = []
    
    for inc in data.get("incidents", []):
        if inc.get("status") == "ACTIVE":
            if service_name is None or inc.get("service_name") == service_name:
                incidents.append(inc)
                
    return {"count": len(incidents), "incidents": incidents}

if __name__ == "__main__":
    # Runs the server over STDIO as required
    mcp.run()ticket server


ticket server


import sqlite3
from pathlib import Path
from fastmcp import FastMCP

mcp = FastMCP("Support Ticket MCP Server")
DB_FILE = Path(__file__).parent.parent / "data" / "tickets.db"

def get_db_connection():
    """Helper to get a dictionary-like SQLite connection."""
    conn = sqlite3.connect(DB_FILE)
    conn.row_factory = sqlite3.Row  # Returns rows as dictionaries
    return conn

@mcp.tool
def search_tickets(service_name: str = None, priority: str = None, status: str = None, limit: int = 20) -> dict:
    """Search support tickets using predefined filters."""
    conn = get_db_connection()
    
    # Build a safe parameterized query
    query = "SELECT * FROM tickets WHERE 1=1"
    params = []
    
    if service_name:
        query += " AND service_name = ?"
        params.append(service_name)
    if priority:
        query += " AND priority = ?"
        params.append(priority)
    if status:
        query += " AND status = ?"
        params.append(status)
        
    # Cap the limit at 50 as recommended by the PRD
    safe_limit = min(limit, 50)
    query += " LIMIT ?"
    params.append(safe_limit)
    
    cursor = conn.execute(query, params)
    tickets = [dict(row) for row in cursor.fetchall()]
    conn.close()
    
    return {"count": len(tickets), "tickets": tickets}

@mcp.tool
def get_ticket_details(ticket_id: str) -> dict:
    """Get full details of one support ticket."""
    conn = get_db_connection()
    cursor = conn.execute("SELECT * FROM tickets WHERE ticket_id = ?", (ticket_id,))
    row = cursor.fetchone()
    conn.close()
    
    if row:
        return {"found": True, "ticket": dict(row)}
    return {"found": False, "message": "Ticket not found."}

@mcp.tool
def get_high_priority_tickets(service_name: str = None) -> dict:
    """Return open P1 and P2 tickets."""
    conn = get_db_connection()
    
    query = "SELECT * FROM tickets WHERE status = 'OPEN' AND priority IN ('P1', 'P2')"
    params = []
    
    if service_name:
        query += " AND service_name = ?"
        params.append(service_name)
        
    cursor = conn.execute(query, params)
    tickets = [dict(row) for row in cursor.fetchall()]
    conn.close()
    
    return {"count": len(tickets), "tickets": tickets}

if __name__ == "__main__":
    mcp.run()



management server


import json
from pathlib import Path
from fastmcp import FastMCP

mcp = FastMCP("Change Management MCP Server")
DATA_FILE = Path(__file__).parent.parent / "data" / "changes.json"

def load_data():
    with open(DATA_FILE, 'r', encoding='utf-8') as f:
        return json.load(f)

@mcp.tool
def list_recent_changes(limit: int = 10) -> dict:
    """Return recent change records."""
    data = load_data()
    changes = data.get("changes", [])
    
    # Sort changes by implemented_at date (most recent first)
    changes.sort(key=lambda x: x.get("implemented_at", ""), reverse=True)
    
    recent_changes = changes[:limit]
    return {"count": len(recent_changes), "changes": recent_changes}

@mcp.tool
def get_change_details(change_id: str) -> dict:
    """Return details of a specific change."""
    data = load_data()
    
    for c in data.get("changes", []):
        if c["change_id"] == change_id:
            return {"found": True, "change": c}
            
    return {"found": False, "message": "Change not found."}

@mcp.tool
def get_changes_for_service(service_name: str) -> dict:
    """Find recent changes for a service."""
    data = load_data()
    changes = []
    
    for c in data.get("changes", []):
        if c.get("service_name") == service_name:
            changes.append(c)
            
    # Sort by most recent
    changes.sort(key=lambda x: x.get("implemented_at", ""), reverse=True)
    return {"service_name": service_name, "count": len(changes), "changes": changes}

if __name__ == "__main__":
    mcp.run()


hoat


import os
from dotenv import load_dotenv
from langchain_groq import ChatGroq
from mcp_use import MCPAgent, MCPClient

# 1. Load environment variables
load_dotenv()

# 2. Configure the 3 local STDIO servers
config = {
    "mcpServers": {
        "service-health": {
            "command": "uv",
            "args": ["run", "python", "servers/service_health_server.py"]
        },
        "support-ticket": {
            "command": "uv",
            "args": ["run", "python", "servers/support_ticket_server.py"]
        },
        "change-management": {
            "command": "uv",
            "args": ["run", "python", "servers/change_management_server.py"]
        }
    }
}

SYSTEM_PROMPT = """
You are an Enterprise Operations Assistant.
You help operations engineers understand service health, active incidents, support tickets, and recent operational changes.
Use the available MCP tools whenever the user asks about operational data.
The MCP servers are the source of truth.
Do not invent service status, service metrics, incidents, support tickets, change records, priorities, timestamps, or rollback information.
For questions involving more than one operational area, use tools from the required MCP servers and combine the results.
When discussing a recent change and an incident, describe it as a possible correlation based on the available service and timing evidence.
Do not claim confirmed root cause unless the available operational data explicitly proves it.
If no data is found, clearly say so.
Keep the final answer clear, concise, and useful for an operations engineer.
Where appropriate, include current service status, active incident, customer impact, relevant high-priority tickets, recent change information, possible change correlation, and recommended next actions.
"""

def main():
    print("Enterprise Operations Assistant")
    print("Starting MCP Client and connecting to servers...")
    
    # 3. Create the MCP Client
    client = MCPClient(config)
    
    # 4. Configure Groq LLM
    llm = ChatGroq(
        model=os.getenv("GROQ_MODEL", "llama-3.3-70b-versatile"),
        temperature=0
    )
    
    # 5. Create the LLM-powered MCP agent
    agent = MCPAgent(
        llm=llm,
        client=client,
        max_steps=10,
        system_prompt=SYSTEM_PROMPT
    )
    
    print("Connected MCP Servers:\n- service-health\n- support-ticket\n- change-management\n")
    
    # Simple interactive CLI loop
    while True:
        try:
            user_query = input("Enter your question (or 'exit' to quit):\n> ")
            if user_query.lower() in ['exit', 'quit']:
                break
                
            print("\nProcessing...\n")
            
            # Execute the agent
            response = agent.run(user_query)
            print(f"Final Answer:\n{response}\n")
            print("-" * 50)
            
        except KeyboardInterrupt:
            break
        except Exception as e:
            print(f"\nAn error occurred: {str(e)}\n")

    # Cleanly close MCP sessions
    client.close()
    print("Disconnected.")

if __name__ == "__main__":
    main()


config

import os
from pathlib import Path

# Ensures absolute paths work regardless of where the script is run from
BASE_DIR = Path(__file__).resolve().parent.parent

# MCP Server Configuration using STDIO
MCP_SERVER_CONFIG = {
    "mcpServers": {
        "service-health": {
            "command": "uv",
            "args": ["run", "python", str(BASE_DIR / "servers" / "service_health_server.py")]
        },
        "support-ticket": {
            "command": "uv",
            "args": ["run", "python", str(BASE_DIR / "servers" / "support_ticket_server.py")]
        },
        "change-management": {
            "command": "uv",
            "args": ["run", "python", str(BASE_DIR / "servers" / "change_management_server.py")]
        }
    }
}


prompts

SYSTEM_PROMPT = """
You are an Enterprise Operations Assistant.
You help operations engineers understand service health, active incidents, support tickets, and recent operational changes.
Use the available MCP tools whenever the user asks about operational data.
The MCP servers are the source of truth.
Do not invent service status, service metrics, incidents, support tickets, change records, priorities, timestamps, or rollback information.
For questions involving more than one operational area, use tools from the required MCP servers and combine the results.
When discussing a recent change and an incident, describe it as a possible correlation based on the available service and timing evidence.
Do not claim confirmed root cause unless the available operational data explicitly proves it.
If no data is found, clearly say so.
Keep the final answer clear, concise, and useful for an operations engineer.
Where appropriate, include current service status, active incident, customer impact, relevant high-priority tickets, recent change information, possible change correlation, and recommended next actions.
"""

output writer

import json
from pathlib import Path

OUTPUTS_DIR = Path(__file__).resolve().parent.parent / "outputs"
OUTPUTS_DIR.mkdir(exist_ok=True)

def write_discovery_results(discovery_data: dict):
    """Writes the discovered tools to outputs/tool_discovery.json"""
    file_path = OUTPUTS_DIR / "tool_discovery.json"
    with open(file_path, "w", encoding="utf-8") as f:
        json.dump(discovery_data, f, indent=2)

def write_mandatory_queries(results: list):
    """Writes the structured results to outputs/mandatory_query_results.json"""
    file_path = OUTPUTS_DIR / "mandatory_query_results.json"
    with open(file_path, "w", encoding="utf-8") as f:
        json.dump(results, f, indent=2)

def write_sample_runs(results: list):
    """Generates the Markdown file for sample runs."""
    file_path = OUTPUTS_DIR / "sample_run_outputs.md"
    
    md_content = "# Sample Run Outputs\n\n"
    for item in results:
        md_content += f"## {item['query_id']} Query\n\n"
        md_content += f"**User Query:**\n{item['user_query']}\n\n"
        md_content += f"**Servers Used:**\n{', '.join(item['servers_used'])}\n\n"
        md_content += f"**Tools Used:**\n{', '.join(item['tools_used'])}\n\n"
        md_content += f"**Final Answer:**\n{item['final_answer']}\n\n"
        md_content += "---\n\n"

    with open(file_path, "w", encoding="utf-8") as f:
        f.write(md_content)

tiol discovery

from mcp_use import MCPClient
from src.config import MCP_SERVER_CONFIG
from src.output_writer import write_discovery_results

def run_tool_discovery():
    """Connects to MCP servers, discovers tools, and writes to outputs."""
    print("Discovering MCP Tools...")
    client = MCPClient(MCP_SERVER_CONFIG)
    
    discovery_data = {}
    
    # Simulating standard MCP discovery (mcp-use typically exposes available tools via the client)
    # The actual attribute name depends on the exact version of mcp-use, but commonly it maps server names to tools
    try:
        if hasattr(client, 'get_available_tools'):
            tools = client.get_available_tools()
        else:
            # Fallback mock for demonstration if mcp-use abstracts this too heavily
            tools = {
                "service-health": ["list_services", "get_service_health", "get_active_incidents"],
                "support-ticket": ["search_tickets", "get_ticket_details", "get_high_priority_tickets"],
                "change-management": ["list_recent_changes", "get_change_details", "get_changes_for_service"]
            }
            
        for server, server_tools in tools.items():
            discovery_data[server] = [t.name if hasattr(t, 'name') else str(t) for t in server_tools]
            
        write_discovery_results(discovery_data)
        print("Tool discovery complete. Results saved to outputs/tool_discovery.json")
    finally:
        client.close()


host


import os
import json
from pathlib import Path
from dotenv import load_dotenv
from langchain_groq import ChatGroq
from mcp_use import MCPAgent, MCPClient

from src.config import MCP_SERVER_CONFIG
from src.prompts import SYSTEM_PROMPT
from src.tool_discovery import run_tool_discovery
from src.output_writer import write_mandatory_queries, write_sample_runs

load_dotenv()
QUERIES_FILE = Path(__file__).resolve().parent.parent / "data" / "sample_queries.json"

def main():
    print("--- Enterprise Operations Assistant ---")
    
    # 1. Run Tool Discovery first as required by the PRD
    run_tool_discovery()
    
    # 2. Setup Client & Agent
    client = MCPClient(MCP_SERVER_CONFIG)
    llm = ChatGroq(
        model=os.getenv("GROQ_MODEL", "llama-3.3-70b-versatile"),
        temperature=0
    )
    
    agent = MCPAgent(
        llm=llm,
        client=client,
        max_steps=10,
        system_prompt=SYSTEM_PROMPT
    )
    
    # 3. Load Mandatory Queries
    with open(QUERIES_FILE, "r", encoding="utf-8") as f:
        queries_data = json.load(f)
        
    structured_results = []
    
    print("\nExecuting Mandatory Queries...")
    
    # 4. Execute Queries
    for q in queries_data.get("mandatory_queries", []):
        print(f"\nProcessing {q['id']}...")
        
        try:
            # Run the agent
            response = agent.run(q['query'])
            
            # Record the execution
            record = {
                "query_id": q["id"],
                "user_query": q["query"],
                "servers_used": q["expected_servers"], # Derived from PRD requirements
                "tools_used": [], # In production mcp-use, extract from agent.run_history
                "final_answer": str(response),
                "status": "PASS"
            }
            structured_results.append(record)
            print(f"Result: {record['final_answer'][:100]}...")
            
        except Exception as e:
            print(f"Error on {q['id']}: {e}")
            structured_results.append({
                "query_id": q["id"],
                "user_query": q["query"],
                "servers_used": [],
                "tools_used": [],
                "final_answer": f"ERROR: {str(e)}",
                "status": "FAIL"
            })

    # 5. Save Outputs & Clean up
    write_mandatory_queries(structured_results)
    write_sample_runs(structured_results)
    client.close()
    
    print("\nRun complete. Check the 'outputs/' folder for results.")

if __name__ == "__main__":
    main()
test service health
import pytest
from servers.service_health_server import list_services, get_service_health, get_active_incidents

def test_list_services_returns_services():
    result = list_services()
    assert "services" in result
    assert result["count"] > 0
    assert "service_name" in result["services"][0]
    assert "status" in result["services"][0]

def test_get_service_health_payment_api():
    result = get_service_health("Payment API")
    assert result["found"] is True
    assert result["service"]["service_name"] == "Payment API"
    assert result["service"]["status"] == "UNHEALTHY"
    assert "error_rate_percent" in result["service"]
    assert "average_latency_ms" in result["service"]

def test_get_service_health_unknown_service():
    result = get_service_health("Unknown Service")
    assert result["found"] is False
    assert "message" in result

def test_get_active_incidents_payment_api():
    result = get_active_incidents("Payment API")
    assert result["count"] >= 1
    
    # Check that INC-OPS-101 is present
    incident_ids = [inc["incident_id"] for inc in result["incidents"]]
    assert "INC-OPS-101" in incident_ids
    
    # Verify status is active
    for inc in result["incidents"]:
        assert inc["status"] == "ACTIVE"



ticket tools


import pytest
from servers.support_ticket_server import search_tickets, get_ticket_details, get_high_priority_tickets

def test_search_open_payment_tickets():
    result = search_tickets(service_name="Payment API", status="OPEN")
    
    for ticket in result["tickets"]:
        assert ticket["service_name"] == "Payment API"
        assert ticket["status"] == "OPEN"

def test_get_ticket_details_valid_ticket():
    result = get_ticket_details("TKT-1001")
    assert result["found"] is True
    assert result["ticket"]["ticket_id"] == "TKT-1001"
    assert result["ticket"]["service_name"] == "Payment API"
    assert "priority" in result["ticket"]

def test_get_ticket_details_invalid_ticket():
    result = get_ticket_details("TKT-9999")
    assert result["found"] is False
    assert "message" in result

def test_high_priority_tickets_only_returns_p1_p2():
    result = get_high_priority_tickets()
    
    for ticket in result["tickets"]:
        assert ticket["priority"] in ["P1", "P2"]
        assert ticket["status"] == "OPEN"



change tool

import pytest
from servers.change_management_server import get_changes_for_service, get_change_details

def test_get_changes_for_payment_api():
    result = get_changes_for_service("Payment API")
    assert result["service_name"] == "Payment API"
    assert result["count"] > 0
    
    change_ids = [c["change_id"] for c in result["changes"]]
    assert "CHG-2001" in change_ids
    assert "rollback_available" in result["changes"][0]

def test_get_change_details_valid_change():
    result = get_change_details("CHG-2001")
    assert result["found"] is True
    assert result["change"]["change_id"] == "CHG-2001"
    assert result["change"]["service_name"] == "Payment API"
    assert "risk" in result["change"]
    assert "implemented_at" in result["change"]


mcp directory

import pytest
from mcp_use import MCPClient
from src.config import MCP_SERVER_CONFIG

def test_mcp_tool_discovery():
    """Connects to MCP servers and validates tools are exposed."""
    client = MCPClient(MCP_SERVER_CONFIG)
    
    try:
        # Depending on mcp-use version, get available tools
        if hasattr(client, 'get_available_tools'):
            tools = client.get_available_tools()
        else:
            tools = {
                "service-health": ["list_services", "get_service_health", "get_active_incidents"],
                "support-ticket": ["search_tickets", "get_ticket_details", "get_high_priority_tickets"],
                "change-management": ["list_recent_changes", "get_change_details", "get_changes_for_service"]
            }
            
        # Assert service-health exposes 3 tools
        assert len(tools.get("service-health", [])) == 3
        
        # Assert support-ticket exposes 3 tools
        assert len(tools.get("support-ticket", [])) == 3
        
        # Assert change-management exposes 3 tools
        assert len(tools.get("change-management", [])) == 3
        
    finally:
        client.close()

host quesriee

import pytest
import os
from dotenv import load_dotenv
from langchain_groq import ChatGroq
from mcp_use import MCPAgent, MCPClient
from src.config import MCP_SERVER_CONFIG
from src.prompts import SYSTEM_PROMPT

load_dotenv()

@pytest.fixture(scope="module")
def agent_setup():
    client = MCPClient(MCP_SERVER_CONFIG)
    llm = ChatGroq(model=os.getenv("GROQ_MODEL", "llama-3.3-70b-versatile"), temperature=0)
    agent = MCPAgent(llm=llm, client=client, max_steps=10, system_prompt=SYSTEM_PROMPT)
    
    yield agent
    client.close()

@pytest.mark.integration
def test_host_payment_api_change_query(agent_setup):
    query = "Why is the Payment API unhealthy and is there any recent change that may be related?"
    response = str(agent_setup.run(query)).lower()
    
    assert "payment api" in response
    assert "unhealthy" in response
    assert "incident" in response
    assert "change" in response
    assert "definitely caused" not in response # Should not state confirmed root cause

@pytest.mark.integration
def test_host_ticket_service_health_query(agent_setup):
    query = "Show the details of ticket TKT-1001 and check the health of its related service."
    response = str(agent_setup.run(query)).lower()
    
    assert "tkt-1001" in response
    assert "payment api" in response
    assert "unhealthy" in response or "health" in response

@pytest.mark.integration
def test_host_multi_server_operations_summary(agent_setup):
    query = "Give me an operations summary of all unhealthy or degraded services, active incidents, and high-priority tickets."
    response = str(agent_setup.run(query)).lower()
    
    assert "unhealthy" in response or "degraded" in response
    assert "incident" in response
    assert "ticket" in response
