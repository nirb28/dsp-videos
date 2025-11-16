# DSP AI Platform - Diagram Collection

This directory contains all diagrams for the DSP AI Platform documentation in formats compatible with Confluence.

## Diagram Formats

- **PlantUML (.puml)** - Can be directly imported into Confluence using the PlantUML macro
- **Mermaid (.mmd)** - Can be imported using the Mermaid macro or converted to images

## Available Diagrams

### Architecture Diagrams

| Diagram | Format | Description | Confluence Macro |
|---------|--------|-------------|------------------|
| platform-architecture.puml | PlantUML | Complete platform architecture with all components | `{plantuml}` |
| deployment-architecture.mmd | Mermaid | Docker Compose deployment layout | `{mermaid}` |
| security-layers.mmd | Mermaid | Security architecture layers | `{mermaid}` |

### Workflow Diagrams

| Diagram | Format | Description | Confluence Macro |
|---------|--------|-------------|------------------|
| end-to-end-workflow.puml | PlantUML | Complete workflow from use case to production | `{plantuml}` |
| rag-query-flow.puml | PlantUML | RAG query processing sequence | `{plantuml}` |
| jwt-authentication-flow.puml | PlantUML | JWT authentication and authorization flow | `{plantuml}` |
| langgraph-workflow.puml | PlantUML | LangGraph document processing workflow | `{plantuml}` |
| promptfoo-testing.mmd | Mermaid | Promptfoo testing workflow | `{mermaid}` |

### Component Diagrams

| Diagram | Format | Description | Confluence Macro |
|---------|--------|-------------|------------------|
| control-tower-manifest.puml | PlantUML | Manifest system class diagram | `{plantuml}` |
| vault-integration.puml | PlantUML | Vault integration architecture | `{plantuml}` |
| knowledge-graph-structure.mmd | Mermaid | Neo4j knowledge graph structure | `{mermaid}` |
| mcp-protocol-flow.mmd | Mermaid | MCP protocol communication sequence | `{mermaid}` |
| rate-limiting-strategy.mmd | Mermaid | Rate limiting implementation | `{mermaid}` |

## How to Import into Confluence

### PlantUML Diagrams

1. **Method 1: Direct Import**
   ```
   {plantuml}
   [Paste the content of the .puml file here]
   {plantuml}
   ```

2. **Method 2: Using PlantUML Server**
   - Upload the .puml file to your PlantUML server
   - Reference it in Confluence using the PlantUML macro

### Mermaid Diagrams

1. **Method 1: Mermaid Macro (if installed)**
   ```
   {mermaid}
   [Paste the content of the .mmd file here]
   {mermaid}
   ```

2. **Method 2: Convert to Image**
   - Use [Mermaid Live Editor](https://mermaid.live/)
   - Paste the .mmd content
   - Export as PNG/SVG
   - Insert image into Confluence

3. **Method 3: Draw.io Integration**
   - Many Mermaid diagrams can be imported into draw.io
   - Save as draw.io diagram in Confluence

## Diagram Themes and Styling

All diagrams use consistent color schemes:

- **Green (#E8F5E9)** - Core services, success states
- **Blue (#E3F2FD)** - Gateway/routing components
- **Orange (#FFE0B2)** - External services, entry points
- **Purple (#F3E5F5)** - Control/management layers
- **Yellow (#FFF3E0)** - Authentication/security
- **Red (#FFEBEE)** - Errors, warnings, critical paths

## Editing Diagrams

### PlantUML
- Use any text editor
- Preview with PlantUML plugins for VS Code, IntelliJ
- Online editor: http://www.plantuml.com/plantuml

### Mermaid
- Use any text editor
- Preview with Mermaid plugins for VS Code
- Online editor: https://mermaid.live/

## Best Practices

1. **Keep diagrams focused** - One concept per diagram
2. **Use consistent styling** - Follow the color scheme
3. **Add notes and legends** - Explain complex parts
4. **Version control** - Track changes in Git
5. **Update documentation** - Keep diagrams in sync with code

## Generating Images from Source

### PlantUML to PNG/SVG
```bash
# Install PlantUML
java -jar plantuml.jar diagram.puml

# Generate SVG
java -jar plantuml.jar -tsvg diagram.puml
```

### Mermaid to PNG/SVG
```bash
# Install Mermaid CLI
npm install -g @mermaid-js/mermaid-cli

# Generate PNG
mmdc -i diagram.mmd -o diagram.png

# Generate SVG
mmdc -i diagram.mmd -o diagram.svg
```

## Support

For questions or updates to diagrams:
- Contact the DSP AI Platform team
- Submit a pull request with diagram updates
- Use the diagram source files for modifications
