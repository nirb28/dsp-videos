# Control Tower – LangGraph Workflow Integration

> Integrate LangGraph workflows with Control Tower to build complex, stateful AI agent workflows with conditional logic, parallel branches, and human-in-the-loop steps.

---

## Overview

### What is LangGraph?

LangGraph is a framework for building stateful, multi-actor applications with LLMs. It supports:

- Nodes and edges defining workflow graphs
- Shared state passed between steps
- Conditional routing and loops
- Checkpointing and recovery
- Human-in-the-loop decisions

### Key Concepts

- **Nodes** – Individual processing steps or agents
- **Edges** – Connections and conditional transitions
- **State** – Shared context for the workflow
- **Checkpoints** – Persistent snapshots of state
- **Channels** – Communication paths between nodes
- **Graphs** – Complete workflow definitions

---

## LangGraph Module Configuration

### Module Definition in Manifest

```json
{
  "name": "document-processing-workflow",
  "type": "langgraph_workflow",
  "config": {
    "workflow_name": "document_processor",
    "graph_definition": {
      "nodes": [
        {
          "id": "extractor",
          "type": "llm",
          "model": "gpt-4",
          "prompt": "Extract key information from: {document}"
        },
        {
          "id": "classifier",
          "type": "llm",
          "model": "gpt-4",
          "prompt": "Classify document type: {summary}"
        }
      ],
      "edges": [
        {"source": "extractor", "target": "classifier"}
      ]
    },
    "checkpointing": {
      "enabled": true,
      "backend": "filesystem",
      "path": "./checkpoints/document_workflows"
    }
  }
}
```

### Cross-References in Manifest

The LangGraph module can reference other modules via Control Tower cross-references:

```json
"cross_references": {
  "rag_backend": {
    "module_name": "rag-config-main",
    "module_type": "rag_config",
    "purpose": "Document retrieval and embeddings",
    "required": true
  }
}
```

---

## Execution Model

### Lifecycle

1. **Workflow Invocation** – Front Door or another service calls the LangGraph module endpoint.  
2. **State Initialization** – Initial state created from request parameters.  
3. **Graph Execution** – Nodes run sequentially or in parallel according to the graph.  
4. **Checkpointing** – State persisted at key points.  
5. **Completion / Pause** – Workflow returns a result or waits for human input.  

### Example Invocation

```bash
curl -X POST http://localhost:8080/projects/ai-docs/modules/document-processing-workflow/execute \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "document_id": "doc-123",
    "metadata": {"source": "knowledge_base"}
  }'
```

---

## Human-in-the-Loop

Workflows can pause and wait for human approval or additional input.

Typical pattern:

- Node emits an `approval_required` state
- Workflow checkpoint is stored
- External UI or system calls a **resume** endpoint with the decision

```bash
curl -X POST http://localhost:8080/projects/ai-docs/modules/document-processing-workflow/resume \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "checkpoint_id": "chkpt-456",
    "decision": "approve",
    "comments": "Looks good"
  }'
```

---

## Monitoring & Logging

- **Execution Logs** – Per-node logs with inputs/outputs (sanitized)
- **State Snapshots** – Checkpoint metadata (IDs, timestamps, status)
- **Metrics** – Workflow latency, success/failure counts, retries

Integrate with:

- Control Tower manifest-based configuration
- Platform-wide logging/metrics stack (e.g., Prometheus, Grafana)

---

## Example Use Cases

### Document Processing Pipeline

- Extract entities and key information
- Classify document type
- Route to specialized processors (e.g., finance, legal)
- Generate summaries and recommendations

### Customer Support Escalation

- Initial AI response generation
- Sentiment analysis node
- Conditional escalation to human agent
- Feedback loop to improve prompts and workflows

### Code Review Automation

- Security scan node
- Style and formatting checks
- Performance analysis
- Aggregated review summary

---

## Best Practices

- **Single Responsibility Nodes** – Keep each node focused on one task
- **Use Checkpoints** – For long-running or critical workflows
- **Error Handling** – Implement robust error handling per node
- **Human Oversight** – Add human approvals for high-risk operations
- **Parallelism** – Use parallel branches for independent tasks
- **Monitoring** – Track workflow metrics and errors
- **Version Control** – Store workflow definitions in Git
- **Testing** – Simulate different input scenarios before production

---

## Related Documentation

- Control Tower Documentation
- CT-Vault-Integration
- CT-Policy-Management
