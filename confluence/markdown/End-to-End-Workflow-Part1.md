# DSP AI Platform: From Use Case to Production (Part 1)

> This guide walks you through taking an AI use case from concept to production. Part 1 covers defining use cases, testing prompts with promptfoo, and preparing for manifest generation with the Control Tower CLI.

---

## Overview

### Complete Workflow

```text
┌──────────────────────────────────────────────────────────────┐
│                    DSP AI Platform Workflow                  │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Define Use Case                                          │
│     └─> Identify requirements and objectives                 │
│                                                              │
│  2. Develop & Test Prompts                                   │
│     └─> Use promptfoo for comprehensive testing              │
│                                                              │
│  3. Generate Project Manifest                                │
│     └─> Use Control Tower Manifestor CLI                     │
│                                                              │
│  4. Configure Services                                       │
│     └─> Set up JWT, RAG, and other modules                   │
│                                                              │
│  5. Deploy with Front Door                                   │
│     └─> Route and manage API endpoints                       │
│                                                              │
│  6. Monitor & Optimize                                       │
│     └─> Track performance and iterate                        │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1: Define Your Use Case

### Example Use Cases

#### Use Case 1: Customer Support Chatbot

- **Objective:** Automate customer support responses using company documentation.
- **Requirements:**
  - Access to product documentation and FAQs
  - Natural language understanding
  - Contextual responses
  - Authentication for sensitive queries
- **Components Needed:**
  - RAG for document retrieval
  - LLM for response generation
  - JWT for authentication
  - API gateway for routing

#### Use Case 2: Code Translation Service

- **Objective:** Convert SAS code to Python automatically.
- **Requirements:**
  - Code parsing and understanding
  - Syntax translation
  - Library mapping
  - Validation and testing
- **Components Needed:**
  - Inference endpoint for code LLM
  - Custom prompts for translation
  - Validation service
  - Monitoring for accuracy

#### Use Case 3: Document Analysis Pipeline

- **Objective:** Extract insights from financial reports.
- **Requirements:**
  - PDF processing
  - Entity extraction
  - Sentiment analysis
  - Summary generation
- **Components Needed:**
  - Document processor
  - Multiple LLM endpoints
  - Data pipeline for ETL
  - Storage for results

---

## Step 2: Test Prompts with Promptfoo

### What is Promptfoo?

Promptfoo is a testing framework for LLM prompts that helps you evaluate prompt performance across models and test cases.

### Setting Up Promptfoo

```bash
# Install promptfoo globally
npm install -g promptfoo

# Or use npx (no installation needed)
npx promptfoo@latest init

# Verify installation
promptfoo --version
```

### Example 1: Customer Support Prompt Testing

```yaml
# promptfooconfig.yaml
# Customer Support Chatbot Configuration
description: "Customer Support Response Testing"

prompts:
  - id: support_v1
    label: "Basic Support Prompt"
    raw: |
      You are a helpful customer support agent for TechCorp.
      Use the following context to answer the customer's question:
      
      Context: {{context}}
      
      Customer Question: {{query}}
      
      Provide a helpful, accurate, and friendly response.

  - id: support_v2
    label: "Enhanced Support Prompt"
    raw: |
      You are an expert customer support specialist at TechCorp.
      
      CONTEXT INFORMATION:
      {{context}}
      
      CUSTOMER INQUIRY:
      {{query}}
      
      INSTRUCTIONS:
      1. Analyze the customer's question carefully
      2. Use only information from the provided context
      3. If the answer isn't in the context, politely say so
      4. Be concise but thorough
      5. Maintain a professional and friendly tone
      
      Response:

providers:
  - id: openai:gpt-4
    label: GPT-4
    config:
      temperature: 0.3
      max_tokens: 500

  - id: groq:llama3-70b
    label: Llama 3 70B
    config:
      apiBaseUrl: https://api.groq.com/openai/v1
      apiKey: ${GROQ_API_KEY}
      temperature: 0.3
      max_tokens: 500

tests:
  - description: "Product return policy"
    vars:
      context: |
        TechCorp Return Policy:
        - 30-day return window for unopened products
        - 14-day return for opened products with defects
        - Original receipt required
        - Shipping costs non-refundable
      query: "Can I return a laptop I bought 3 weeks ago?"
    assert:
      - type: contains
        value: "30-day"
      - type: contains
        value: "return"
      - type: not-contains
        value: "I don't know"

  - description: "Technical support"
    vars:
      context: |
        Troubleshooting WiFi Issues:
        1. Restart your router
        2. Check cable connections
        3. Update network drivers
        4. Run network diagnostics
      query: "My WiFi keeps disconnecting"
    assert:
      - type: contains-any
        value: ["restart", "router", "diagnostics"]
      - type: llm-rubric
        value: "Response provides clear troubleshooting steps"

  - description: "Out of context question"
    vars:
      context: |
        Product catalog includes laptops, desktops, and accessories.
      query: "What's your competitor's pricing?"
    assert:
      - type: llm-rubric
        value: "Response politely indicates information is not available"
      - type: not-contains
        value: "competitor pricing"
```

### Running Promptfoo Tests

```bash
# Run all tests
promptfoo eval

# Run with specific output format
promptfoo eval --output results.json

# View results in web UI
promptfoo view

# Generate detailed HTML report
promptfoo eval --output results.html --output-format html

# Run specific test cases
promptfoo eval --filter "return policy"

# Compare multiple prompts
promptfoo eval --prompts support_v1,support_v2
```

### Example 2: Code Translation Prompt Testing

```yaml
# sas2py-promptfoo.yaml
description: "SAS to Python Code Translation Testing"

prompts:
  - file: prompts/sas2py_basic.txt
  - file: prompts/sas2py_advanced.txt

providers:
  - id: openai:gpt-4
    label: GPT-4 Code
    config:
      temperature: 0.1
      max_tokens: 2000

  - id: anthropic:claude-3
    label: Claude 3
    config:
      temperature: 0.1
      max_tokens: 2000

tests:
  - description: "Simple DATA step translation"
    vars:
      sas_code: |
        DATA work.output;
          SET work.input;
          new_var = old_var * 2;
        RUN;
    assert:
      - type: contains
        value: "pandas"
      - type: contains
        value: "df"
      - type: python
        value: |
          # Check if output is valid Python
          import ast
          try:
            ast.parse(output)
            return True
          except SyntaxError:
            return False

  - description: "PROC SQL translation"
    vars:
      sas_code: |
        PROC SQL;
          CREATE TABLE result AS
          SELECT id, SUM(amount) as total
          FROM transactions
          GROUP BY id;
        QUIT;
    assert:
      - type: contains-any
        value: ["groupby", "agg", "sum"]
      - type: llm-rubric
        value: "The Python code correctly implements SQL group by logic"

  - description: "Macro variable translation"
    vars:
      sas_code: |
        %LET threshold = 100;
        DATA filtered;
          SET original;
          WHERE value > &threshold;
        RUN;
    assert:
      - type: contains
        value: "threshold = 100"
      - type: contains-any
        value: ["query", "loc", "filter"]
```

### Analyzing Test Results

Key promptfoo metrics to watch:

- **Pass rate** – percentage of assertions passed.
- **Latency** – response time per provider.
- **Cost** – token usage and estimated cost.
- **Quality score** – LLM-judged quality metrics.

---

## Related Documentation

- **End-to-End Workflow Part 1B** – Manifest generation and deployment.
- **End-to-End Workflow Part 2** – Production deployment and advanced topics.
- **Workflow Quick Reference** – Quick-start commands and checklists.

> Next, continue with **Part 1B** to learn about using the Control Tower Manifestor CLI, creating project manifests, deploying with Front Door and JWT, and running integration tests.
