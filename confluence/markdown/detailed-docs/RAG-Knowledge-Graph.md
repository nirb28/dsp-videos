# RAG Service - Knowledge Graph Integration

> This guide describes the Neo4j knowledge graph integration with the RAG service, enabling semantic relationship extraction, graph-based retrieval, and entity-relationship mapping for enhanced document understanding.

---

## Overview

### Knowledge Graph Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                 Knowledge Graph Pipeline                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Document Input                                             │
│       ↓                                                     │
│  Text Extraction                                            │
│       ↓                                                     │
│  Entity Recognition (NER)                                   │
│       ↓                                                     │
│  Relationship Extraction                                    │
│       ↓                                                     │
│  Graph Construction                                         │
│       ↓                                                     │
│  ┌─────────────────────────────────────┐                   │
│  │         Neo4j Graph Database        │                   │
│  │                                     │                   │
│  │  (Person)--[WORKS_FOR]-->(Company)  │                   │
│  │        ↓                            │                   │
│  │     [LOCATED_IN]                    │                   │
│  │        ↓                            │                   │
│  │     (Location)                      │                   │
│  └─────────────────────────────────────┘                   │
│       ↓                                                     │
│  Graph Queries & Traversal                                  │
│       ↓                                                     │
│  Semantic Search Results                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Configuration

### Neo4j Knowledge Graph Setup

```json
// knowledge-graph-config.json
{
  "configuration_name": "knowledge-graph-rag",
  "vector_store": {
    "type": "neo4j_knowledge_graph",
    "neo4j_uri": "neo4j://localhost:7687",
    "neo4j_user": "neo4j",
    "neo4j_password": "${NEO4J_PASSWORD}",
    "neo4j_database": "rag",
    "kg_llm_config_name": "entity-extraction-llm",
    "extraction_prompt": "Extract entities and relationships from the following text. Identify people, organizations, locations, events, and their relationships.",
    "node_properties": ["name", "type", "description", "source"],
    "relationship_properties": ["type", "weight", "confidence"],
    "embedding_dimension": 384
  },
  "chunking": {
    "strategy": "semantic",
    "chunk_size": 500,
    "chunk_overlap": 50
  },
  "embedding": {
    "model": "sentence-transformers/all-MiniLM-L6-v2"
  }
}
```

---

## Entity and Relationship Extraction

### LangGraph Transformer Implementation

```python
# knowledge_graph_extractor.py
from langchain_experimental.graph_transformers import LLMGraphTransformer
from langchain_core.documents import Document
from neo4j import GraphDatabase
from typing import List, Dict, Any

class KnowledgeGraphExtractor:
    def __init__(self, neo4j_config: Dict[str, Any], llm):
        self.driver = GraphDatabase.driver(
            neo4j_config["neo4j_uri"],
            auth=(neo4j_config["neo4j_user"], neo4j_config["neo4j_password"]),
        )
        self.llm = llm
        self.transformer = LLMGraphTransformer(
            llm=llm,
            allowed_nodes=[
                "Person", "Organization", "Location",
                "Event", "Concept", "Technology",
                "Product", "Date", "Metric",
            ],
            allowed_relationships=[
                "WORKS_FOR", "LOCATED_IN", "PART_OF",
                "RELATED_TO", "CREATED", "MANAGES",
                "OWNS", "PARTNERS_WITH", "COMPETES_WITH",
                "HAPPENED_ON", "PARTICIPATED_IN", "USES",
            ],
            node_properties=["description", "category", "importance"],
            relationship_properties=["since", "until", "confidence", "source"],
        )

    async def extract_and_store(self, documents: List[Document]) -> Dict[str, Any]:
        """Extract entities and relationships from documents and store them in Neo4j."""
        graph_documents = await self.transformer.aconvert_to_graph_documents(documents)

        stats = {
            "documents_processed": len(documents),
            "nodes_created": 0,
            "relationships_created": 0,
            "entities": {},
        }

        with self.driver.session() as session:
            for graph_doc in graph_documents:
                for node in graph_doc.nodes:
                    self._create_node(session, node)
                    stats["nodes_created"] += 1
                    t = node.type
                    stats["entities"][t] = stats["entities"].get(t, 0) + 1

                for relationship in graph_doc.relationships:
                    self._create_relationship(session, relationship)
                    stats["relationships_created"] += 1

        return stats

    def _create_node(self, session, node):
        query = """
        MERGE (n:{type} {{id: $id}})
        SET n.name = $name,
            n.description = $description,
            n.properties = $properties,
            n.created_at = timestamp()
        RETURN n
        """.format(type=node.type)

        session.run(
            query,
            id=node.id,
            name=node.properties.get("name", node.id),
            description=node.properties.get("description", ""),
            properties=node.properties,
        )

    def _create_relationship(self, session, relationship):
        query = """
        MATCH (start {{id: $start_id}})
        MATCH (end {{id: $end_id}})
        MERGE (start)-[r:{type}]->(end)
        SET r.properties = $properties,
            r.created_at = timestamp()
        RETURN r
        """.format(type=relationship.type)

        session.run(
            query,
            start_id=relationship.start_id,
            end_id=relationship.end_id,
            properties=relationship.properties,
        )
```

---

## Graph-Aware Retrieval

### Combining Vector Search with Graph Context

```cypher
// Example graph query snippet used for context expansion
MATCH (n {id: $node_id})
OPTIONAL MATCH path = (n)-[*1..$max_hops]-(connected)
WITH n, connected, path
RETURN n,
       collect(DISTINCT connected) as connected_nodes,
       collect(DISTINCT path) as paths
```

A typical retrieval flow:

1. Embed query text and perform vector search.
2. For each top-k result, gather graph context (neighbors, paths, entities).
3. Compute a **combined score** from similarity and graph importance.
4. Return re-ranked results plus rich graph context to the RAG pipeline.

---

## Advanced Graph Analytics

### Graph Algorithms (Neo4j GDS)

```cypher
// PageRank - find influential entities
CALL gds.pageRank.stream('knowledge-graph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC
LIMIT 10

// Community detection
CALL gds.louvain.stream('knowledge-graph')
YIELD nodeId, communityId
RETURN communityId, collect(gds.util.asNode(nodeId).name) AS members
ORDER BY size(members) DESC

// Centrality analysis
CALL gds.betweenness.stream('knowledge-graph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name,
       score AS betweenness_centrality
ORDER BY score DESC
LIMIT 10

// Node similarity
CALL gds.nodeSimilarity.stream('knowledge-graph')
YIELD node1, node2, similarity
RETURN gds.util.asNode(node1).name AS entity1,
       gds.util.asNode(node2).name AS entity2,
       similarity
ORDER BY similarity DESC
LIMIT 20
```

---

## Use Cases

### Expert Finding

- Identify subject matter experts based on graph connectivity.
- Combine roles, projects, publications, and relationships.

### Competitive Intelligence

- Map companies, products, and partnerships.
- Track technology adoption and collaborations.

### Research Discovery

- Connect papers, authors, and key concepts.
- Explore citation networks and topic clusters.

---

## Performance Optimization

### Graph Optimization Tips

- Create indexes on frequently queried properties.
- Use node labels to narrow match patterns.
- Limit traversal depth (`max_hops`).
- Batch node and relationship creation.
- Use graph projections for analytics (GDS).
- Cache common queries.
- Monitor query plans and execution times.

### Index Creation

```cypher
// Basic indexes
CREATE INDEX person_name IF NOT EXISTS FOR (p:Person) ON (p.name);
CREATE INDEX org_name IF NOT EXISTS FOR (o:Organization) ON (o.name);
CREATE INDEX location_name IF NOT EXISTS FOR (l:Location) ON (l.name);

// Composite index
CREATE INDEX person_org IF NOT EXISTS FOR (p:Person) ON (p.name, p.organization);

// Full-text search index
CALL db.index.fulltext.createNodeIndex(
  "searchIndex",
  ["Person", "Organization", "Technology"],
  ["name", "description"]
);

// Vector index for embeddings
CALL db.index.vector.createNodeIndex(
  'document-embeddings',
  'Document',
  'embedding',
  384,
  'cosine'
);
```

---

## Related Documentation

- RAG Service Documentation Part 1.
- RAG-MCP-Server.
- RAG Query Expansion (if present in your documentation set).
