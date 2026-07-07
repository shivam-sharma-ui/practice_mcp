env.example 

GROQ_API_KEY=
GROQ_MODEL=llama-3.3-70b-versatile
src/schemas.py


from pydantic import BaseModel, Field
from typing import Optional 

class ShipmentMetadata(BaseModel):
    shipment_id: str = Field(..., description="The unique identifier for the shipment container (e.g., SH-4002).")
    cargo_weight_tons: float = Field(..., description="Weight of the cargo in tons as a floating point number.")
    cargo_type: str = Field(..., description="Type of cargo being transported (e.g., industrial electronics).")
    target_warehouse_id: str = Field(..., description="The originally planned target destination warehouse ID.")
    has_perishables: bool = Field(..., description="True if the cargo mentions perishable components or cooling needs, else False.")
    maximum_tolerable_delay_hours: Optional[float] = Field(None, description="Maximum transit delay the shipment can sustain in hours, or null if unmentioned.") 

Ssrc/state
from typing import TypedDict, Optional, List, Any 

class LogisticsIncidentState(TypedDict):
    incident_id: str
    manifest_text: str
    disrupted_port_id: str
    extracted_metadata: Optional[dict]
    routing_rag_context: str
    available_routes: List[dict]
    current_route_index: int
    selected_route: Optional[dict]
    warehouse_db_context: Optional[dict]
    reroute_impact_score: int
    routing_decision: str  # OPTIMAL_PATH_FOUND, ROUTE_CLARIFICATION, CRITICAL_DELAY
    clarification_attempts: int
    max_clarification_attempts: int
    logs: List[str]
    final_report: Optional[dict] 

Tools
import json
import os
from typing import Dict, List, Any, Optional 

DATA_DIR = os.path.join(os.path.dirname(os.path.dirname(__file__)), "data") 

def query_warehouse_inventory_tool(warehouse_id: str) -> Dict[str, Any]:
    """Queries warehouse metrics like utilization, status, and risk tier."""
    file_path = os.path.join(DATA_DIR, "inventory_status.json")
    if not os.path.exists(file_path):
        return {"error": f"Inventory database not found at {file_path}"}
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        if warehouse_id in data:
            return data[warehouse_id]
        return {"error": f"Warehouse ID '{warehouse_id}' not found in system data."}
    except Exception as e:
        return {"error": f"Tool execution failed: {str(e)}"} 

def get_alternative_routes_tool(disrupted_port_id: str) -> List[Dict[str, Any]]:
    """Loads all alternative path choices matching a blocked port ID."""
    file_path = os.path.join(DATA_DIR, "route_options.json")
    if not os.path.exists(file_path):
        return []
    
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            data = json.load(f)
        return data.get(disrupted_port_id, [])
    except Exception:
        return []
Rag
import os
from langchain_community.vectorstores import FAISS
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter 

DATA_DIR = os.path.join(os.path.dirname(os.path.dirname(__file__)), "data") 

class LogisticsRAG:
    def __init__(self):
        self.embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")
        self.db = None
        self._init_vector_store() 

    def _init_vector_store(self):
        kb_path = os.path.join(DATA_DIR, "logistics_knowledge_base.txt")
        if not os.path.exists(kb_path):
            raise FileNotFoundError(f"Missing rulebook content file at {kb_path}")
            
        with open(kb_path, 'r', encoding='utf-8') as f:
            text = f.read()
            
        splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
        chunks = splitter.split_text(text)
        self.db = FAISS.from_texts(chunks, self.embeddings) 

    def retrieve_rules(self, query: str) -> str:
        """Pulls operational logic metrics context relevant to the incoming prompt."""
        if not self.db:
            return "No vector space initialized."
        docs = self.db.similarity_search(query, k=3)
        return "\n".join([doc.page_content for doc in docs]) 

rag_instance = LogisticsRAG() 

Nodes 
import os
from langchain_groq import ChatGroq
from langchain_core.prompts import ChatPromptTemplate
from src.state import LogisticsIncidentState
from src.schemas import ShipmentMetadata
from src.tools import get_alternative_routes_tool, query_warehouse_inventory_tool
from src.rag import rag_instance 

def get_llm():
    return ChatGroq(
        temperature=0.0,
        model_name=os.getenv("GROQ_MODEL", "llama-3.3-70b-versatile"),
        groq_api_key=os.getenv("GROQ_API_KEY")
    ) 

def parse_incident(state: LogisticsIncidentState) -> dict:
    llm = get_llm()
    structured_llm = llm.with_structured_output(ShipmentMetadata)
    
    prompt = ChatPromptTemplate.from_template(
        "Analyze the following manifest text and extract metadata:\n\n{text}"
    )
    chain = prompt | structured_llm
    metadata = chain.invoke({"text": state["manifest_text"]})
    
    logs = state.get("logs", [])
    logs.append(f"Successfully extracted manifest fields for {metadata.shipment_id}")
    return {"extracted_metadata": metadata.model_dump(), "logs": logs} 

def policy_rag_lookup(state: LogisticsIncidentState) -> dict:
    context = rag_instance.retrieve_rules(state["manifest_text"])
    logs = state.get("logs", [])
    logs.append("Executed RAG contextual rulebook lookup")
    return {"routing_rag_context": context, "logs": logs} 

def load_alternative_routes(state: LogisticsIncidentState) -> dict:
    routes = get_alternative_routes_tool(state["disrupted_port_id"])
    logs = state.get("logs", [])
    logs.append(f"Loaded {len(routes)} potential rerouting choices from local catalog")
    return {
        "available_routes": routes,
        "current_route_index": 0,
        "clarification_attempts": 0,
        "max_clarification_attempts": 2,
        "logs": logs
    } 

def select_route(state: LogisticsIncidentState) -> dict:
    idx = state["current_route_index"]
    routes = state["available_routes"]
    logs = state.get("logs", [])
    
    if idx < len(routes):
        selected = routes[idx]
        logs.append(f"Assigned verification targets to path index {idx}: {selected['route_id']}")
        return {"selected_route": selected, "logs": logs}
    
    logs.append("No active path remaining at current pointer position")
    return {"selected_route": None, "logs": logs} 

def check_warehouse(state: LogisticsIncidentState) -> dict:
    selected = state["selected_route"]
    if not selected:
        return {}
    
    wh_data = query_warehouse_inventory_tool(selected["warehouse_id"])
    logs = state.get("logs", [])
    logs.append(f"Retrieved active status metrics for target storage unit {selected['warehouse_id']}")
    return {"warehouse_db_context": wh_data, "logs": logs} 

def analyze_route(state: LogisticsIncidentState) -> dict:
    selected = state["selected_route"]
    metadata = state["extracted_metadata"]
    wh_ctx = state["warehouse_db_context"]
    logs = state.get("logs", [])
    
    if not selected:
        logs.append("Aborting logic checklist pass due to missing candidate routing reference")
        return {"routing_decision": "CRITICAL_DELAY", "logs": logs}
    
    if "error" in wh_ctx:
        logs.append(f"Rejecting candidate route {selected['route_id']} due to missing warehouse footprint records")
        return {"routing_decision": "ROUTE_CLARIFICATION", "logs": logs}
        
    util = wh_ctx.get("current_utilization_pct", 0)
    status = wh_ctx.get("operational_status", "INACTIVE")
    risk = wh_ctx.get("risk_tier", "NORMAL")
    delay = selected.get("added_delay_hours", 0)
    max_delay = metadata.get("maximum_tolerable_delay_hours")
    
    # 1. Evaluate Score Matrix strictly using Python logic[span_2](start_span)[span_2](end_span)
    score = 0
    if util > 85: score += 30
    if risk == "ELEVATED": score += 25
    if status != "ACTIVE": score += 30
    if max_delay is not None and delay > max_delay: score += 25
    if delay > 120: score += 50
    score = min(score, 100)
    
    # 2. Evaluate Exact Branching States based on the PRD Rules[span_3](start_span)[span_3](end_span)
    if delay > 120:
        decision = "CRITICAL_DELAY"
    elif util > 85 or status != "ACTIVE" or risk == "ELEVATED":
        decision = "ROUTE_CLARIFICATION"
    elif max_delay is not None and delay > max_delay:
        decision = "ROUTE_CLARIFICATION"
    else:
        decision = "OPTIMAL_PATH_FOUND"
        
    logs.append(f"Evaluated routing {selected['route_id']} -> Score: {score}, Decision: {decision}")
    return {"reroute_impact_score": score, "routing_decision": decision, "logs": logs} 

def route_clarification(state: LogisticsIncidentState) -> dict:
    attempts = state["clarification_attempts"] + 1
    next_idx = state["current_route_index"] + 1
    logs = state.get("logs", [])
    
    routes_logged = state.get("final_report", {}).get("routes_evaluated", [])
    current_route = state["selected_route"]
    
    reason = "Warehouse capacity or operational limitations reached"
    if current_route:
        delay = current_route.get("added_delay_hours", 0)
        max_delay = state["extracted_metadata"].get("maximum_tolerable_delay_hours")
        if max_delay is not None and delay > max_delay:
            reason = f"Transit delay ({delay} hrs) violates threshold caps ({max_delay} hrs)"
            
        routes_logged.append({
            "route_id": current_route["route_id"],
            "decision": "ROUTE_CLARIFICATION",
            "reason": reason
        })
    
    if next_idx >= len(state["available_routes"]) or attempts >= state["max_clarification_attempts"]:
        logs.append(f"Loop boundaries reached. Transitioning case tracker to escalation pathways.")
        return {
            "routing_decision": "CRITICAL_DELAY",
            "clarification_attempts": attempts,
            "current_route_index": next_idx,
            "logs": logs,
            "final_report": {"routes_evaluated": routes_logged}
        }
        
    logs.append(f"Advancing iterator indexes to candidate index value: {next_idx}")
    return {
        "clarification_attempts": attempts,
        "current_route_index": next_idx,
        "routing_decision": "ROUTE_CLARIFICATION",
        "logs": logs,
        "final_report": {"routes_evaluated": routes_logged}
    } 

def finalize_route(state: LogisticsIncidentState) -> dict:
    logs = state.get("logs", [])
    logs.append(f"Formally logging optimization target: {state['selected_route']['route_id']}")
    
    routes_logged = state.get("final_report", {}).get("routes_evaluated", [])
    current_route = state["selected_route"]
    if current_route:
        routes_logged.append({
            "route_id": current_route["route_id"],
            "decision": "OPTIMAL_PATH_FOUND",
            "reason": "Warehouse and route conditions are acceptable"
        })
        
    return {"logs": logs, "final_report": {"routes_evaluated": routes_logged}} 

def escalate_incident(state: LogisticsIncidentState) -> dict:
    logs = state.get("logs", [])
    logs.append("CRITICAL: Rerouting paths exhausted or delay threshold violated. Escalating to global logistics desk.")
    return {"logs": logs, "routing_decision": "CRITICAL_DELAY"}
Repoet writer 
import os
from src.state import LogisticsIncidentState
from src.nodes import get_llm 

def generate_report(state: LogisticsIncidentState) -> dict:
    llm = get_llm()
    
    selected_route = state.get("selected_route") if state["routing_decision"] == "OPTIMAL_PATH_FOUND" else {}
    wh_ctx = state.get("warehouse_db_context") if state["warehouse_db_context"] else {}
    
    routes_evaluated = state.get("final_report", {}).get("routes_evaluated", [])
    if not routes_evaluated and state["selected_route"]:
        routes_evaluated.append({
            "route_id": state["selected_route"]["route_id"],
            "decision": state["routing_decision"],
            "reason": "Evaluated based on automated dispatch parameters"
        }) 

    # Short summary LLM generator
    summary_prompt = (
        f"Generate a very short, professional final operations brief explaining a shipping incident disruption. "
        f"Context details: Port ID {state['disrupted_port_id']}, Incident Manifest Summary: {state['manifest_text']}. "
        f"Final operational decision made: {state['routing_decision']}. "
        f"Be direct, action-oriented, and extremely concise."
    )
    brief_response = llm.invoke(summary_prompt)
    
    report_data = {
        "incident_id": state["incident_id"],
        "original_incident_summary": state["manifest_text"],
        "parsed_metadata": state["extracted_metadata"],
        "rag_validation_rules_applied": [state["routing_rag_context"]],
        "routes_evaluated": routes_evaluated,
        "final_selected_route": selected_route,
        "queried_warehouse_metrics": wh_ctx,
        "graph_routing_metadata": {
            "loops_executed": state["clarification_attempts"],
            "final_decision_state": state["routing_decision"],
            "reroute_impact_score": state["reroute_impact_score"]
        },
        "final_operations_brief": brief_response.content.strip()
    }
    
    outputs_dir = os.path.join(os.path.dirname(os.path.dirname(__file__)), "outputs")
    os.makedirs(outputs_dir, exist_ok=True)
    
    import json
    target_path = os.path.join(outputs_dir, "reroute_advisory_report.json")
    with open(target_path, 'w', encoding='utf-8') as f:
        json.dump(report_data, f, indent=2)
        
    # Also drop specific run target paths to match candidate expectations
    incident_specific_path = os.path.join(outputs_dir, f"{state['incident_id']}_reroute_advisory_report.json")
    with open(incident_specific_path, 'w', encoding='utf-8') as f:
        json.dump(report_data, f, indent=2)
        
    return {"final_report": report_data} 

Grqph
from langgraph.graph import StateGraph, START, END
from src.state import LogisticsIncidentState
import src.nodes as nodes
from src.report_writer import generate_report 

def decide_routing_logic(state: LogisticsIncidentState) -> str:
    decision = state["routing_decision"]
    if decision == "OPTIMAL_PATH_FOUND":
        return "finalize_route"
    elif decision == "ROUTE_CLARIFICATION":
        return "route_clarification"
    return "escalate_incident" 

def decide_loop_next(state: LogisticsIncidentState) -> str:
    if state["routing_decision"] == "CRITICAL_DELAY":
        return "escalate_incident"
    return "select_route" 

def build_logistics_graph():
    builder = StateGraph(LogisticsIncidentState)
    
    # Register core operational steps nodes
    builder.add_node("parse_incident", nodes.parse_incident)
    builder.add_node("policy_rag_lookup", nodes.policy_rag_lookup)
    builder.add_node("load_alternative_routes", nodes.load_alternative_routes)
    builder.add_node("select_route", nodes.select_route)
    builder.add_node("check_warehouse", nodes.check_warehouse)
    builder.add_node("analyze_route", nodes.analyze_route)
    builder.add_node("route_clarification", nodes.route_clarification)
    builder.add_node("finalize_route", nodes.finalize_route)
    builder.add_node("escalate_incident", nodes.escalate_incident)
    builder.add_node("generate_report", generate_report)
    
    # Establish direct workflows bindings
    builder.add_edge(START, "parse_incident")
    builder.add_edge("parse_incident", "policy_rag_lookup")
    builder.add_edge("policy_rag_lookup", "load_alternative_routes")
    builder.add_edge("load_alternative_routes", "select_route")
    builder.add_edge("select_route", "check_warehouse")
    builder.add_edge("check_warehouse", "analyze_route")
    
    # Inject routing logic checks[span_4](start_span)[span_4](end_span)
    builder.add_conditional_edges(
        "analyze_route",
        decide_routing_logic,
        {
            "finalize_route": "finalize_route",
            "route_clarification": "route_clarification",
            "escalate_incident": "escalate_incident"
        }
    )
    
    builder.add_conditional_edges(
        "route_clarification",
        decide_loop_next,
        {
            "select_route": "select_route",
            "escalate_incident": "escalate_incident"
        }
    )
    
    builder.add_edge("finalize_route", "generate_report")
    builder.add_edge("escalate_incident", "generate_report")
    builder.add_edge("generate_report", END)
    
    return builder.compile() 

logistics_app = build_logistics_graph()
Main
import os
import json
from dotenv import load_dotenv
load_dotenv() 

from src.graph import logistics_app 

def main():
    data_path = os.path.join(os.path.dirname(os.path.dirname(__file__)), "data", "sample_incidents.json")
    if not os.path.exists(data_path):
        print(f"CRITICAL: Testing files array database matrix target missing at {data_path}")
        return
        
    with open(data_path, 'r', encoding='utf-8') as f:
        incidents = json.load(f)
        
    print("\n==============================================")
    print(" Supply Chain Logistics Crisis Re-Router CLI ")
    print("==============================================\n")
    print("Available Incidents:")
    for idx, inc in enumerate(incidents):
        print(f"{idx + 1}. {inc['incident_id']} - {inc['manifest_text'][:60]}...")
        
    try:
        selection = int(input("\nSelect incident number to execute: ")) - 1
        if selection < 0 or selection >= len(incidents):
            print("Invalid index choice selected. Shutting down pipeline processing run execution context.")
            return
    except ValueError:
        print("Invalid input sequence parameters.")
        return
        
    target_incident = incidents[selection]
    print(f"\nKicking off LangGraph engine workflows for target run: {target_incident['incident_id']}...")
    
    initial_input = {
        "incident_id": target_incident["incident_id"],
        "manifest_text": target_incident["manifest_text"],
        "disrupted_port_id": target_incident["disrupted_port_id"],
        "current_route_index": 0,
        "clarification_attempts": 0,
        "logs": [],
        "routing_rag_context": "",
        "available_routes": [],
        "selected_route": None,
        "warehouse_db_context": None,
        "reroute_impact_score": 0,
        "routing_decision": "PENDING"
    }
    
    final_state = logistics_app.invoke(initial_input)
    print(f"\nExecution finalized successfully! Result Decision Status Matrix state caught: {final_state['routing_decision']}")
    print(f"GeneratedOperationsBrief Summary Note: {final_state['final_report']['final_operations_brief']}") 

if __name__ == "__main__":
    main()
