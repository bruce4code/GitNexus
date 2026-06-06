# Deep Dive: Graph RAG Agent Architecture in GitNexus

> GitNexus implements a sophisticated LLM Agent that combines Graph RAG (Retrieval-Augmented Generation) with LangGraph's ReAct pattern. This article analyzes the architecture, streaming mechanism, tool design, and dynamic context injection that make this agent effective for code analysis.

## Overview: The Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React)                          │
│  useAppState.tsx → sendChatMessage() → streamAgentResponse() │
└────────────────────────────┬────────────────────────────────┘
                             │ HTTP/WebSocket
┌────────────────────────────▼────────────────────────────────┐
│                    Agent Core                                │
│  agent.ts → createGraphRAGAgent() → streamAgentResponse()    │
│  ├── LangGraph ReAct Agent                                   │
│  ├── 7 Graph RAG Tools                                       │
│  └── Dynamic Context Builder                                 │
└────────────────────────────┬────────────────────────────────┘
                             │ Backend API Calls
┌────────────────────────────▼────────────────────────────────┐
│                    Backend (gitnexus serve)                   │
│  LadybugDB → Cypher queries, FTS, Vector search              │
└─────────────────────────────────────────────────────────────┘
```

---

## Part 1: Agent Factory — `createGraphRAGAgent`

### 1.1 Multi-Provider Chat Model Factory

The agent supports **9 LLM providers** through LangChain's unified interface:

```typescript
export const createChatModel = (config: ProviderConfig): BaseChatModel => {
  switch (config.provider) {
    case 'openai':      return new ChatOpenAI({ ... });
    case 'azure-openai': return new AzureChatOpenAI({ ... });
    case 'gemini':      return new ChatGoogleGenerativeAI({ ... });
    case 'anthropic':   return new ChatAnthropic({ ... });
    case 'ollama':      return new ChatOllama({ ... });
    case 'openrouter':  return new ChatOpenAI({ baseURL: 'https://openrouter.ai/api/v1' });
    case 'minimax':     return new ChatAnthropic({ baseURL: 'https://api.minimax.io/anthropic' });
    case 'glm':         return new ChatOpenAI({ baseURL: 'https://api.z.ai/api/coding/paas/v4' });
    case 'deepseek':    return new DeepSeekChatOpenAI({ ... });  // Special handling!
  }
};
```

**Key insight**: Most providers use standard LangChain adapters, but **DeepSeek** requires a custom subclass because it needs to replay `reasoning_content` from prior turns—a provider-specific field that LangChain's OpenAI adapter drops.

### 1.2 ReAct Agent Creation

```typescript
export const createGraphRAGAgent = (
  config: ProviderConfig,
  backend: GraphRAGBackend,
  codebaseContext?: CodebaseContext,
) => {
  const model = createChatModel(config);
  const tools = createGraphRAGTools(backend);

  const systemPrompt = codebaseContext
    ? buildDynamicSystemPrompt(BASE_SYSTEM_PROMPT, codebaseContext)
    : BASE_SYSTEM_PROMPT;

  const agent = createReactAgent({
    llm: model,
    tools: tools,
    messageModifier: new SystemMessage(systemPrompt),
  });

  return agent;
};
```

The agent is created via `createReactAgent` from `@langchain/langgraph/prebuilt`. This implements the **ReAct (Reasoning + Acting)** pattern:

```
User Query → [Reasoning] → [Tool Call] → [Tool Result] → [Reasoning] → ... → [Final Answer]
```

---

## Part 2: System Prompt Engineering

### 2.1 Prompt Structure

The `BASE_SYSTEM_PROMPT` follows a **directive-first** design inspired by Aider/Cline research:

```typescript
export const BASE_SYSTEM_PROMPT = `You are Nexus, a Code Analysis Agent with access to a Knowledge Graph. Your responses MUST be grounded.

## ⚠️ MANDATORY: GROUNDING
Every factual claim MUST include a citation.
- File refs: [[src/auth.ts:45-60]] (line range with hyphen)
- NO citation = NO claim. Say "I didn't find evidence" instead of guessing.

## ⚠️ MANDATORY: VALIDATION
Every output MUST be validated.
- Use cypher to validate the results and confirm completeness of context before final output.
- NO validation = NO claim.

## 🧠 CORE PROTOCOL
You are an investigator. For each question:
1. **Search** → Use cypher, search or grep to find relevant code
2. **Read** → Use read to see the actual source
3. **Trace** → Use cypher to follow connections in the graph
4. **Cite** → Ground every finding with [[file:line]] or [[Type:Name]]
5. **Validate** → Use cypher to validate the results and confirm completeness of context before final output.

## 🛠️ TOOLS
- **search** — Hybrid search. Results grouped by process with cluster context.
- **cypher** — Cypher queries against the graph. Use {{QUERY_VECTOR}} for vector search.
- **grep** — Regex search. Best for exact strings, TODOs, error codes.
- **read** — Read file content. Always use after search/grep to see full code.
- **explore** — Deep dive on a symbol, cluster, or process.
- **overview** — Codebase map showing all clusters and processes.
- **impact** — Impact analysis. Shows affected processes, clusters, and risk level.

## 📝CRITICAL RULES
- **impact output is trusted.** Do NOT re-validate with cypher.
- **Cite or retract.** Never state something you can't ground.
- **Read before concluding.** Don't guess from names alone.
- **OUTPUT STYLE** Prefer using tables and mermaid diagrams instead of long explanations.
`;
```

**Design principles**:
1. **Grounding mandate at the top** — most critical instruction
2. **Short, punchy directives** — no long explanations
3. **No template-inducing examples** — let LLM figure out HOW
4. **Explicit validation requirement** — anti-laziness directive

### 2.2 Dynamic Context Injection

The system prompt is **augmented with live codebase context** at agent initialization:

```typescript
// context-builder.ts
export async function buildCodebaseContext(
  executeQuery: (cypher: string) => Promise<any[]>,
  projectName: string,
): Promise<CodebaseContext> {
  const [stats, hotspots, folderTree] = await Promise.all([
    getCodebaseStats(executeQuery, projectName),   // File/Function/Class counts
    getHotspots(executeQuery),                      // Most connected nodes
    getFolderTree(executeQuery),                    // ASCII folder tree
  ]);
  return { stats, hotspots, folderTree };
}
```

The injected context looks like:

```
---

## 📦 CURRENT CODEBASE
### 📊 CODEBASE: gitnexus-web
Files: 150 | Functions: 420 | Classes: 35 | Interfaces: 28

**Hotspots** (most connected):
- `useAppState` (Function) — 85 edges
- `GraphCanvas` (Class) — 72 edges
- `backend-client` (File) — 65 edges

### 📁 STRUCTURE
```
gitnexus-web/
  src/
    components/ (45 files)
    hooks/
      useAppState.tsx
      useSigma.ts
    core/
      llm/
        agent.ts
        tools.ts
    services/
      backend-client.ts
```
```

This gives the LLM a **mental model of the project** before it even starts exploring.

---

## Part 3: The 7 Graph RAG Tools

### Tool Architecture

```typescript
export interface GraphRAGBackend {
  executeQuery: (cypher: string) => Promise<Record<string, unknown>[]>;
  search: (query: string, opts?: { limit?: number; mode?: 'hybrid' | 'semantic' | 'bm25' }) => Promise<EnrichedSearchResult[]>;
  grep: (pattern: string, limit?: number) => Promise<GrepResult[]>;
  readFile: (filePath: string) => Promise<string>;
}

export const createGraphRAGTools = (backend: GraphRAGBackend) => {
  // Each tool is created with LangChain's tool() function
  // Schema defined with Zod for automatic validation
};
```

### Tool 1: `search` — Hybrid Search with Process Grouping

```typescript
const searchTool = tool(
  async ({ query, limit, groupByProcess }) => {
    const searchResults = await backendSearch(query, { limit: k, enrich: true });
    
    // Group by process for context
    if (groupByProcess) {
      // Results grouped as:
      // PROCESS: onCreate → showToast (5 matches, 8 steps)
      //   [1] Function: validateToken [score: 0.85] (step 3/8)
      //     ID: fn-123
      //     File: src/auth.ts (lines 45-60)
      //     Cluster: Authentication
      //     Connections: -[CALLS 100%]-> verifySignature
    }
  },
  {
    name: 'search',
    description: 'Search for code by keywords or concepts. Groups results by process with cluster context.',
    schema: z.object({
      query: z.string().describe('What you are looking for'),
      groupByProcess: z.boolean().optional().describe('Group results by process (default: true)'),
      limit: z.number().optional().describe('Max results (default: 10)'),
    }),
  },
);
```

**Key feature**: Results are **enriched** with:
- 1-hop graph connections (CALLS, IMPORTS)
- Cluster membership
- Process participation (step number)

### Tool 2: `cypher` — Raw Graph Queries with Vector Embedding

```typescript
const cypherTool = tool(
  async ({ query, cypher }) => {
    // Special handling for {{QUERY_VECTOR}} placeholder
    if (cypher.includes('{{QUERY_VECTOR}}')) {
      // Route to backend semantic search instead of local embedding
      const semanticResults = await backendSearch(query, { mode: 'semantic' });
      return formatSemanticResults(semanticResults);
    }
    
    const results = await executeQuery(cypher);
    // Format as markdown table (token-efficient)
    return formatAsMarkdownTable(results);
  },
  {
    name: 'cypher',
    description: `Execute a Cypher query against the code graph.
    
Example queries:
- MATCH (caller:Function)-[:CodeRelation {type: 'CALLS'}]->(fn:Function {name: 'validate'}) RETURN caller.name
- MATCH (child:Class)-[:CodeRelation {type: 'EXTENDS'}]->(parent:Class) RETURN child.name, parent.name`,
    schema: z.object({
      cypher: z.string().describe('The Cypher query'),
      query: z.string().optional().describe('Natural language query (required if cypher contains {{QUERY_VECTOR}})'),
    }),
  },
);
```

**Why `{{QUERY_VECTOR}}` placeholder?** Embedding generation happens on the backend (LadybugDB's vector index), not in the browser. The tool routes to semantic search API when this placeholder is detected.

### Tool 3: `grep` — Regex Pattern Search

```typescript
const grepTool = tool(
  async ({ pattern, fileFilter, caseSensitive, maxResults }) => {
    // Validate regex locally before sending
    new RegExp(pattern, caseSensitive ? 'g' : 'gi');
    
    const results = await backendGrep(fullPattern, limit);
    return `Found ${results.length} matches:\n\n${results.map(r => `${r.filePath}:${r.line}: ${r.text}`).join('\n')}`;
  },
  { name: 'grep', ... },
);
```

### Tool 4: `read` — File Content Reader

```typescript
const readTool = tool(
  async ({ filePath }) => {
    const content = await readFile(filePath);
    
    // Truncate large files (50KB limit)
    if (content.length > 50000) {
      return `File: ${filePath} (${lines} lines, truncated)\n\n${content.slice(0, 50000)}\n\n... [truncated]`;
    }
    return `File: ${filePath} (${lines} lines)\n\n${content}`;
  },
  { name: 'read', ... },
);
```

### Tool 5: `overview` — Codebase Map

```typescript
const overviewTool = tool(
  async () => {
    // Run 4 parallel queries
    const [clusters, processes, deps, critical] = await Promise.all([
      executeQuery(`MATCH (c:Community) RETURN c.label, c.symbolCount, c.cohesion ORDER BY c.symbolCount DESC`),
      executeQuery(`MATCH (p:Process) RETURN p.label, p.stepCount, p.processType`),
      executeQuery(`MATCH (a)-[:CodeRelation {type: 'CALLS'}]->(b) WHERE a.cluster <> b.cluster RETURN ...`),
      executeQuery(`MATCH (s)-[:CodeRelation {type: 'STEP_IN_PROCESS'}]->(p:Process) RETURN p.label, COUNT(*)`),
    ]);
    
    return formatOverviewTable(clusters, processes, deps, critical);
  },
  { name: 'overview', ... },
);
```

### Tool 6: `explore` — Deep Dive on Symbol/Cluster/Process

```typescript
const exploreTool = tool(
  async ({ target, type }) => {
    // Auto-detect type if not specified
    if (!type || type === 'process') {
      const processRes = await executeQuery(`MATCH (p:Process) WHERE p.label = '${target}' RETURN ...`);
      if (processRes.length > 0) resolvedType = 'process';
    }
    
    // Similar for cluster and symbol...
    
    // Return detailed info based on type
    if (resolvedType === 'symbol') {
      return [
        `SYMBOL: ${nodeType} ${name}`,
        `ID: ${nodeId}`,
        `File: ${filePath}`,
        `Cluster: ${clusterLabel}`,
        `PROCESSES:`,
        `- ${processLabel} (step 3/8)`,
        `CONNECTIONS:`,
        `-[CALLS 100%]-> verifySignature, <-[IMPORTS 95%]- auth.ts`,
      ].join('\n');
    }
  },
  { name: 'explore', ... },
);
```

### Tool 7: `impact` — Change Impact Analysis

```typescript
const impactTool = tool(
  async ({ target, direction, maxDepth, relationTypes, minConfidence }) => {
    // Find target node
    const targetResults = await executeQuery(`MATCH (n) WHERE n.name = '${target}' RETURN n.id, label(n)`);
    
    // Multi-depth traversal (separate queries for each depth)
    const depthQueries = [
      executeQuery(d1Query),  // Direct connections
      executeQuery(d2Query),  // 2 hops
      executeQuery(d3Query),  // 3 hops
    ];
    const depthResults = await Promise.all(depthQueries);
    
    // Aggregate affected processes and clusters
    const [affectedProcesses, affectedClusters] = await Promise.all([
      executeQuery(`MATCH (s)-[:STEP_IN_PROCESS]->(p) WHERE s.id IN [...] RETURN p.label, COUNT(*)`),
      executeQuery(`MATCH (s)-[:MEMBER_OF]->(c) WHERE s.id IN [...] RETURN c.label, COUNT(*)`),
    ]);
    
    // Calculate risk level
    let risk = 'LOW';
    if (directCount >= 30 || processCount >= 5) risk = 'CRITICAL';
    else if (directCount >= 15 || processCount >= 3) risk = 'HIGH';
    else if (directCount >= 5) risk = 'MEDIUM';
    
    return formatImpactReport(depthResults, affectedProcesses, affectedClusters, risk);
  },
  { name: 'impact', ... },
);
```

**Output format**:

```
🔴 IMPACT: validateToken | upstream | 45 affected
Confidence: High 38 | Medium 5 | Low 2

AFFECTED PROCESSES:
- onCreate → showToast - BROKEN at step 3 (12 symbols, 8 steps)
- login → redirect - BROKEN at step 2 (8 symbols, 5 steps)

AFFECTED CLUSTERS:
- Authentication (direct, 15 symbols)
- API Handlers (indirect, 8 symbols)

RISK: HIGH
- Direct callers: 12
- Processes affected: 2
- Clusters affected: 2

d=1 (Directly DEPEND ON validateToken):
  Function|handleAuth|auth.ts:45|CALLS|100%
    ↳ "const token = validateToken(req.headers.authorization)"
  Function|checkSession|session.ts:20|CALLS|95%
    ↳ "if (!validateSession(token)) return false"

✅ GRAPH ANALYSIS COMPLETE (trusted)
⚠️ Optional: grep("validateToken") for dynamic patterns
```

---

## Part 4: Streaming Architecture — `streamAgentResponse`

### 4.1 Dual Stream Mode

The agent uses **both** LangGraph stream modes simultaneously:

```typescript
const stream = await agent.stream({ messages: formattedMessages }, {
  streamMode: ['values', 'messages'],  // ← Both modes!
  recursionLimit: 50,                   // Allow long tool chains
  signal: options.signal,               // Abort support
});

for await (const event of stream) {
  // Events come as [mode, data] tuples
  let [mode, data] = event;
  
  if (mode === 'messages') {
    // Token-by-token streaming (AIMessageChunk)
  }
  if (mode === 'values') {
    // State snapshots (tool calls, results)
  }
}
```

**Why both modes?**

| Mode | What it provides | Use case |
|------|------------------|----------|
| `messages` | Token-by-token text chunks | Real-time text streaming |
| `values` | Full state snapshots | Tool call/result detection in order |

### 4.2 Reasoning vs Content Classification

A critical challenge: **distinguish "thinking" from "final answer"**.

```typescript
// Track pending tool calls
let pendingToolCalls = 0;
let hasSeenToolCallThisTurn = false;

// In AIMessageChunk handling:
if (content && content.length > 0) {
  // Determine if this is reasoning/narration vs final answer content
  const isReasoning = !hasSeenToolCallThisTurn || toolCalls.length > 0 || pendingToolCalls > 0;
  
  if (isReasoning) {
    yield { type: 'reasoning', reasoning: content };
  } else {
    yield { type: 'content', content };
  }
}
```

**Logic**:
- Before first tool call → reasoning (narration of plan)
- Between tool calls → reasoning (interpreting results)
- After all tools complete → final content (answer)

### 4.3 Deduplication Across Modes

Both modes can emit the same tool call, so we deduplicate:

```typescript
const yieldedToolCalls = new Set<string>();
const yieldedToolResults = new Set<string>();

// In messages mode:
if (!yieldedToolCalls.has(toolId)) {
  yieldedToolCalls.add(toolId);
  yield { type: 'tool_call', toolCall: { id: toolId, name: tc.name, status: 'running' } };
}

// In values mode (backup):
if (!yieldedToolCalls.has(toolId)) {
  yieldedToolCalls.add(toolId);
  yield { type: 'tool_call', ... };
}
```

### 4.4 History Message Capture

For providers like DeepSeek that require exact transcript replay:

```typescript
if (options.captureHistory) {
  lastStepMessages = stepMessages;  // Capture from 'values' mode
}

// At stream end:
yield {
  type: 'done',
  historyMessages: serializeAgentHistoryMessages(lastStepMessages, startIndex),
};
```

The `historyMessages` are stored in the frontend's `ChatMessage.historyMessages` and replayed on subsequent turns.

---

## Part 5: Frontend Integration

### 5.1 Agent Initialization

```typescript
// useAppState.tsx
const initializeAgent = async (overrideProjectName?: string) => {
  const config = getActiveProviderConfig();
  
  // Build backend interface
  const backend = {
    executeQuery: (cypher) => backendRunQuery(cypher, repo),
    search: (query, opts) => backendSearch(query, { ...opts, repo }),
    grep: (pattern, limit) => backendGrep(pattern, repo, limit),
    readFile: (filePath) => backendReadFile(filePath, { repo }).then(r => r.content),
  };
  
  // Build codebase context (parallel queries)
  const codebaseContext = await buildCodebaseContext(executeQuery, projectName);
  
  // Create agent
  agentRef.current = createGraphRAGAgent(config, backend, codebaseContext);
  setIsAgentReady(true);
};
```

### 5.2 Streaming Message Handler

```typescript
const sendChatMessage = async (message: string) => {
  // Add user message
  setChatMessages(prev => [...prev, userMessage]);
  
  // Create placeholder for assistant response
  const assistantMessageId = `assistant-${Date.now()}`;
  const stepsForMessage: MessageStep[] = [];
  
  // rAF-scheduled update
  const scheduleMessageUpdate = () => {
    if (pendingUpdate) return;
    pendingUpdate = true;
    rafHandle = requestAnimationFrame(() => {
      pendingUpdate = false;
      updateMessage();
    });
  };
  
  // Stream agent response
  for await (const chunk of streamAgentResponse(agent, history, { signal })) {
    switch (chunk.type) {
      case 'reasoning':
        stepsForMessage.push({ id: `step-${stepCounter++}`, type: 'reasoning', content: chunk.reasoning });
        scheduleMessageUpdate();
        break;
      case 'tool_call':
        stepsForMessage.push({ id: `step-${stepCounter++}`, type: 'tool_call', toolCall: chunk.toolCall });
        setCurrentToolCalls(prev => [...prev, chunk.toolCall]);
        scheduleMessageUpdate();
        break;
      case 'content':
        stepsForMessage.push({ id: `step-${stepCounter++}`, type: 'content', content: chunk.content });
        scheduleMessageUpdate();
        break;
      case 'done':
        // Store historyMessages for next turn
        assistantHistoryMessages = chunk.historyMessages;
        break;
    }
  }
};
```

### 5.3 UI Rendering of Steps

```tsx
// RightPanel.tsx
{message.steps && message.steps.length > 0 ? (
  <div className="space-y-4">
    {message.steps.map((step, index) => (
      <div key={step.id}>
        {step.type === 'reasoning' && (
          <div className="border-l-2 border-text-muted/30 pl-3 italic">
            <MarkdownRenderer content={step.content} />
          </div>
        )}
        {step.type === 'tool_call' && (
          <ToolCallCard toolCall={step.toolCall} defaultExpanded={false} />
        )}
        {step.type === 'content' && (
          <MarkdownRenderer content={step.content} showCopyButton={true} />
        )}
      </div>
    ))}
  </div>
) : (
  // Fallback: old format (content + toolCalls)
)}
```

---

## Part 6: Key Design Decisions

### 6.1 Why LangGraph ReAct over Custom Agent?

- **Built-in tool orchestration** — no need to implement tool selection logic
- **Streaming support** — both token and state streaming
- **Recursion limit** — prevents infinite tool loops
- **Abort signal** — clean cancellation

### 6.2 Why 7 Tools Instead of More?

The tool set is **consolidated** for efficiency:
- `search` combines BM25 + semantic + graph context
- `explore` handles symbol, cluster, and process deep-dives
- `impact` does multi-depth traversal in one call

Fewer tools = less decision overhead for the LLM.

### 6.3 Why Backend-Side Embedding?

Embedding models (ONNX/Transformers) are **heavy** (~100MB). Running them in the browser would:
- Increase bundle size dramatically
- Require WebGPU (not universally available)
- Duplicate computation across clients

The backend (LadybugDB) has:
- Pre-computed embeddings stored in vector index
- Efficient similarity search via `QUERY_VECTOR_INDEX`

### 6.4 Why Markdown Tables for Cypher Results?

```typescript
// Token-efficient format
const header = `| ${columnNames.join(' | ')} |`;
const separator = `|${columnNames.map(() => '---').join('|')}|`;
const rows = results.slice(0, 50).map(row => `| ${values.join(' | ')} |`);
```

JSON per-row would be verbose. Markdown tables are:
- Human-readable
- Compact
- Easy for LLM to parse

---

## Summary

The GitNexus Graph RAG Agent demonstrates several advanced patterns:

1. **Multi-provider abstraction** with provider-specific handling (DeepSeek reasoning)
2. **Dynamic context injection** — codebase stats, hotspots, folder tree appended to system prompt
3. **Dual-stream architecture** — `messages` for tokens, `values` for state
4. **Reasoning vs content classification** — based on tool call timing
5. **Consolidated tool set** — 7 tools covering search, graph, file, and impact analysis
6. **Backend-side embedding** — vector search via LadybugDB
7. **rAF-scheduled updates** — smooth UI streaming at 60fps

This architecture enables effective code analysis by combining the LLM's reasoning capability with structured graph data — the essence of Graph RAG.