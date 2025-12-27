### Folder Structure

If I were to build a real MCP server in Python, here is how I would organize it:

```
mcp-server/
├── main.py                     # Entry point, initializes the MCP server
├── config/
│   ├── __init__.py
│   └── settings.py             # Environment variables, API keys, rate limit configs
├── tools/
│   ├── __init__.py
│   ├── registry.py             # Central registry for tool discovery
│   ├── search_tool.py          # Example: FAQ or document search
│   ├── database_tool.py        # Example: CRUD operations
│   └── notification_tool.py    # Example: Send email or Slack message
├── schemas/
│   ├── __init__.py
│   └── tool_schemas.py         # Pydantic models for inputs/outputs
├── middleware/
│   ├── __init__.py
│   ├── auth.py                 # Authentication logic (API keys, JWT)
│   └── rate_limiter.py         # Rate limiting per client/tool
├── utils/
│   ├── __init__.py
│   └── helpers.py              # Shared utility functions
├── tests/
│   └── test_tools.py           # Unit tests for each tool
├── requirements.txt
└── README.md
```

### Defining Tools, Inputs, and Outputs

Each tool would be defined as a Python class or function with:

- **Name**: A clear, descriptive identifier (e.g., `search_faq`, `create_order`)
- **Description**: A plain-language explanation of what the tool does, so the LLM knows when to use it
- **Input Schema**: Defined using Pydantic models with required and optional fields, types, and validation rules
- **Output Schema**: Structured response format so the LLM can reliably parse results

**Example structure for a tool:**

```
Tool: search_faq
Description: "Search the FAQ database for answers related to a user question."
Inputs:
  - query (string, required): The user's question
  - limit (integer, optional, default=3): Number of results to return
Outputs:
  - results (array): List of matching FAQ entries with question, answer, and relevance score
  - success (boolean): Whether the search completed successfully
```

All tools would be registered in a central `registry.py` file, which maintains a dictionary of available tools with their metadata. This registry serves as the single source of truth for tool discovery.

### Handling Auth, Rate Limits, and Tool Discovery

**Authentication:**
- Implement API key validation in middleware that runs before any tool execution
- For more sensitive deployments, use JWT tokens with expiration
- Store valid keys in environment variables or a secure secrets manager
- Return clear error responses (401 Unauthorized) when auth fails

**Rate Limiting:**
- Use an in-memory store (like Redis) to track requests per client per time window
- Apply limits at two levels: global (per client) and per-tool (for expensive operations)
- Return 429 Too Many Requests with a Retry-After header when limits are exceeded
- Configure different limits for different tools based on their resource cost

**Tool Discovery:**
- Expose a `/tools` or `/discover` endpoint that returns the full list of available tools
- Each tool entry includes: name, description, input schema, output schema
- The LLM or orchestrator queries this endpoint to understand what capabilities are available
- Tools can be dynamically enabled/disabled based on user permissions or context

---

## 2. Logic Distribution in n8n-Based Tool Approach

Here is how I separate concerns across the three layers:

### Inside the LLM Prompt

- **Role definition**: Who the agent is and how it should behave
- **Tool descriptions**: Clear explanations of what each tool does and when to use it
- **Decision logic**: Instructions on when to answer directly vs. when to call a tool
- **Output formatting rules**: How to structure responses for the user
- **Guardrails**: What the agent should not do, topics to avoid, escalation triggers

**Example prompt logic:**
> "If the user asks about order status, call the `check_order` tool with the order ID. If the user asks a general question about return policy, answer directly from your knowledge."

### Inside n8n

- **Orchestration**: Deciding which workflow or sub-workflow to execute based on tool selection
- **Data transformation**: Shaping inputs before sending to databases or APIs
- **API integrations**: Making HTTP calls to external services
- **Error handling**: Try/catch logic, retries, fallback paths
- **Conditional routing**: If/else branches based on tool outputs or validation results
- **Batching and looping**: Processing multiple records efficiently
- **Response formatting**: Preparing structured data to return to the LLM

### Inside the Database

- **Persistent storage**: User records, orders, FAQs, conversation history
- **Business data**: Product catalogs, pricing, inventory, customer information
- **Memory storage**: Long-term agent memories, session states, user preferences
- **Idempotency tracking**: Logs of processed webhook IDs or request identifiers
- **Vector embeddings**: Stored in vector databases like Pinecone for semantic search
- **Audit logs**: Records of what actions were taken and when

**Summary Table:**

| Layer    | Responsibility                                      |
|----------|-----------------------------------------------------|
| Prompt   | Decision-making, tool selection, response style     |
| n8n      | Execution, orchestration, data shaping, integrations|
| Database | Persistent data, memory, embeddings, audit trails   |

---

## 3. Debugging a RAG System That Returns Wrong Answers

If my RAG system returns a wrong or hallucinated answer even though relevant data exists in Pinecone, here is how I would debug it step by step:

### Step 1: Check Embeddings

**What I check:**
- Is the embedding model consistent between indexing and querying? (Same model must be used for both)
- Are the embeddings being generated correctly for the user query?
- Is there a mismatch in embedding dimensions?

**How I debug:**
- Log the raw embedding vector for a test query
- Manually verify the embedding model version in both indexing and retrieval code
- Test with a simple, known query that should have an obvious match

**Common issues:**
- Using a different embedding model at query time than at index time
- Preprocessing differences (lowercasing, cleaning) between indexing and querying

### Step 2: Check Chunking

**What I check:**
- Is the relevant information split across multiple chunks?
- Are chunks too large (diluting relevance) or too small (losing context)?
- Is there enough overlap between chunks to preserve meaning?

**How I debug:**
- Find the original source document containing the correct answer
- Examine how it was chunked—did the chunking split the answer awkwardly?
- Check chunk boundaries and overlap settings

**Common issues:**
- Critical information split between two chunks, so neither scores high enough
- Chunk size too large, causing important details to be buried
- No overlap, so context is lost at chunk boundaries

### Step 3: Check Retrieval

**What I check:**
- What chunks are actually being returned by Pinecone?
- What are their similarity scores?
- Is the correct chunk present but ranked too low?

**How I debug:**
- Log the full retrieval results, not just top-k
- Check the similarity scores of returned chunks
- Manually inspect whether the correct chunk exists in Pinecone at all
- Try increasing top-k to see if the correct chunk appears lower in the ranking

**Common issues:**
- Top-k is too small, and the correct chunk is at position 6 when only fetching 3
- Metadata filters are accidentally excluding relevant chunks
- The query embedding does not semantically match the chunk embedding (vocabulary mismatch)

### Step 4: Check Prompt Construction

**What I check:**
- Are the retrieved chunks actually being passed into the prompt?
- Is the prompt structured so the LLM prioritizes the retrieved context?
- Is there conflicting or confusing information in the context?

**How I debug:**
- Log the full prompt being sent to the LLM, including all retrieved chunks
- Verify the retrieved context appears before the question
- Check if the prompt explicitly instructs the LLM to answer only based on provided context
- Look for contradictory information across chunks that might confuse the model

**Common issues:**
- Retrieved chunks are not clearly separated or labeled in the prompt
- Prompt does not instruct the LLM to prioritize context over its own knowledge
- Too many chunks causing the LLM to lose focus or hit context limits
- Chunks are included but placed after the question, reducing their influence

### Debugging Checklist Summary

| Step               | Key Question                                         | Action                                      |
|--------------------|------------------------------------------------------|---------------------------------------------|
| Embeddings         | Same model for index and query?                      | Log and compare embedding vectors           |
| Chunking           | Is the answer split or buried?                       | Inspect source document chunks              |
| Retrieval          | Is the right chunk being retrieved?                  | Log full results with scores                |
| Prompt Construction| Is context being used correctly by the LLM?          | Log full prompt, check instructions         |

---

Thank you for taking the time to review my responses. I genuinely enjoyed thinking through these scenarios, and I'm excited about the opportunity to bring this kind of problem-solving to your team. Looking forward to the next steps!

— Junaid Asif