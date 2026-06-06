# Low-Level Design (LLD) - AI Engine
## AI-Powered Workflow Creation System

**Tech Stack:** Python, FastAPI, LangGraph, LangChain, Azure OpenAI, pgvector, Langfuse, Redis, LiteLLM

**Deployment:** Separate Docker container on Kubernetes cluster

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Service Architecture](#service-architecture)
3. [LLM Integration](#llm-integration)
4. [LangGraph Workflow Agent](#langgraph-workflow-agent)
5. [RAG System](#rag-system)
6. [Async Job Queue](#async-job-queue)
7. [LLM Gateway](#llm-gateway)
8. [Observability & Tracing](#observability--tracing)
9. [API Endpoints](#api-endpoints)
10. [Error Handling](#error-handling)
11. [Performance Optimization](#performance-optimization)
12. [Security Considerations](#security-considerations)
13. [Testing Strategy](#testing-strategy)
14. [Deployment](#deployment)

---

## 1. Architecture Overview

### 1.1 AI Engine High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    .NET Backend API                          │
├──────────────────────────────────────────────────────────────┤
│  (Authentication, Authorization, Business Logic, Data Access)│
└──────────────────┬───────────────────────────────────────────┘
                   │
                   │ HTTP/REST Calls
                   │
┌──────────────────▼───────────────────────────────────────────┐
│                  Python FastAPI AI Engine                    │
├──────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐  ┌──────────────────────────────┐  │
│  │  API Layer           │  │  LangGraph Agent             │  │
│  │  (FastAPI Routes)    │  │  (Workflow Orchestrator)     │  │
│  └──────────────────────┘  └──────────────────────────────┘  │
│  ┌──────────────────────┐  ┌──────────────────────────────┐  │
│  │  Service Layer       │  │  RAG System                  │  │
│  │  (Business Logic)    │  │  (pgvector + Retrieval)      │  │
│  └──────────────────────┘  └──────────────────────────────┘  │
│  ┌──────────────────────┐  ┌──────────────────────────────┐  │
│  │  LLM Integration     │  │  Async Queue Integration     │  │
│  │  (LiteLLM Proxy)     │  │  (Hangfire Job Handler)      │  │
│  └──────────────────────┘  └──────────────────────────────┘  │
│  ┌──────────────────────┐  ┌──────────────────────────────┐  │
│  │  Observability       │  │  Cache Layer                 │  │
│  │  (Langfuse Tracing)  │  │  (Redis)                     │  │
│  └──────────────────────┘  └──────────────────────────────┘  │
└──────────────┬──────────────────────────┬────────────────────┘
               │                          │
        ┌──────▼─────────┐        ┌──────▼──────────────┐
        │ Azure OpenAI   │        │ PostgreSQL          │
        │ (GPT-4o)       │        │ (pgvector + RAG)    │
        └────────────────┘        └─────────────────────┘
                │
                │
        ┌───────▼────────────┐
        │ LiteLLM Proxy      │
        │ (Fallback, Cache)  │
        └────────────────────┘
```

### 1.2 Why a Separate Python Service?

| Aspect | Reason |
|--------|--------|
| **AI Libraries** | LangChain, LangGraph, semantic-kernel are Python-native |
| **Orchestration** | Complex multi-step agent workflows easier in Python |
| **Development** | Faster iteration on AI features |
| **Dependencies** | AI/ML libraries add significant overhead to .NET |
| **Scalability** | Separate container allows independent scaling |
| **Deployment** | Python ecosystem better for AI model serving |
| **Monitoring** | Dedicated observability for AI operations |

---

## 2. Service Architecture

### 2.1 FastAPI Application Structure

```
ai-engine/
├── main.py                          # FastAPI app entry point
├── config.py                        # Configuration management
├── requirements.txt                 # Python dependencies
├── Dockerfile                       # Container definition
├── .env.example                     # Environment variables template
│
├── app/
│   ├── __init__.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── routes.py            # Route definitions
│   │   │   ├── endpoints/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── workflow_generation.py
│   │   │   │   ├── workflow_optimization.py
│   │   │   │   ├── validation.py
│   │   │   │   ├── chat.py
│   │   │   │   └── health.py
│   │   │   └── dependencies.py      # Dependency injection
│   │
│   ├── core/
│   │   ├── __init__.py
│   │   ├── settings.py              # Settings & config
│   │   ├── security.py              # Auth & security
│   │   └── exceptions.py            # Custom exceptions
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── llm_service.py           # LLM interactions
│   │   ├── langgraph_agent.py       # LangGraph orchestrator
│   │   ├── rag_service.py           # RAG retrieval
│   │   ├── validation_service.py    # Workflow validation
│   │   └── cache_service.py         # Caching layer
│   │
│   ├── agents/
│   │   ├── __init__.py
│   │   ├── workflow_agent.py        # Main workflow generation agent
│   │   ├── optimizer_agent.py       # Optimization agent
│   │   └── validator_agent.py       # Validation agent
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   ├── schemas.py               # Pydantic models
│   │   ├── workflow.py              # Workflow data models
│   │   └── execution.py             # Execution models
│   │
│   ├── integrations/
│   │   ├── __init__.py
│   │   ├── azure_openai.py          # Azure OpenAI client
│   │   ├── pgvector_rag.py          # pgvector integration
│   │   ├── redis_cache.py           # Redis integration
│   │   ├── litellm_gateway.py       # LiteLLM proxy
│   │   ├── langfuse_tracing.py      # Langfuse integration
│   │   └── hangfire_client.py       # Hangfire job queue
│   │
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── logger.py                # Logging setup
│   │   ├── validators.py            # Validation utilities
│   │   ├── parsers.py               # JSON/output parsing
│   │   └── helpers.py               # Helper functions
│   │
│   └── middleware/
│       ├── __init__.py
│       ├── error_handler.py         # Error handling
│       ├── auth.py                  # Auth middleware
│       └── observability.py         # Tracing middleware
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                  # Pytest fixtures
│   ├── test_api.py
│   ├── test_agents.py
│   ├── test_rag.py
│   └── test_llm_integration.py
│
└── docker-compose.yml               # Local development stack
```

### 2.2 Main FastAPI Application

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZIPMiddleware
from contextlib import asynccontextmanager

from app.api.v1 import routes
from app.core import settings
from app.middleware import error_handler, observability
from app.integrations import azure_openai, redis_cache, langfuse_tracing

# Startup/shutdown events
@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await redis_cache.connect()
    await azure_openai.initialize()
    langfuse_tracing.initialize()
    
    print("✓ AI Engine started successfully")
    
    yield
    
    # Shutdown
    await redis_cache.disconnect()
    print("✓ AI Engine shutdown complete")

# Create FastAPI app
app = FastAPI(
    title="AI Workflow Engine",
    description="AI-powered workflow generation and optimization",
    version="1.0.0",
    lifespan=lifespan
)

# Middleware
app.add_middleware(GZIPMiddleware, minimum_size=1000)
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom middleware
app.middleware("http")(error_handler.error_middleware)
app.middleware("http")(observability.tracing_middleware)

# Routes
app.include_router(routes.router, prefix="/api/v1")

# Health check
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "AI Workflow Engine",
        "version": "1.0.0"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=8000,
        reload=settings.DEBUG
    )
```

---

## 3. LLM Integration

### 3.1 Azure OpenAI Client Setup

```python
# integrations/azure_openai.py
import os
from langchain_openai import AzureChatOpenAI
from litellm import Router
import asyncio

class AzureOpenAIClient:
    """Wrapper for Azure OpenAI with LiteLLM fallback"""
    
    def __init__(self):
        self.primary_llm = None
        self.router = None
        self.initialized = False
    
    async def initialize(self):
        """Initialize Azure OpenAI clients"""
        # Primary model - GPT-4o for complex workflows
        self.primary_llm = AzureChatOpenAI(
            model="gpt-4o",
            azure_endpoint=os.getenv("AZURE_OAI_ENDPOINT"),
            api_key=os.getenv("AZURE_OAI_KEY"),
            api_version="2024-05-01-preview",
            temperature=0.7,
            max_tokens=4096
        )
        
        # Fast model - GPT-4o-mini for classification
        self.fast_llm = AzureChatOpenAI(
            model="gpt-4o-mini",
            azure_endpoint=os.getenv("AZURE_OAI_ENDPOINT"),
            api_key=os.getenv("AZURE_OAI_KEY"),
            api_version="2024-05-01-preview",
            temperature=0.5,
            max_tokens=1024
        )
        
        # LiteLLM Router for fallback
        self.router = Router(
            model_list=[
                {
                    "model_name": "primary",
                    "litellm_params": {
                        "model": "azure/gpt-4o",
                        "api_key": os.getenv("AZURE_OAI_KEY"),
                        "api_base": os.getenv("AZURE_OAI_ENDPOINT"),
                        "api_version": "2024-05-01-preview"
                    }
                },
                {
                    "model_name": "fallback",
                    "litellm_params": {
                        "model": "claude-sonnet-4-6-20250514",
                        "api_key": os.getenv("ANTHROPIC_API_KEY")
                    }
                }
            ],
            timeout=60,
            num_retries=3
        )
        
        self.initialized = True
    
    async def generate_workflow(self, prompt: str, context: dict = None) -> str:
        """Generate workflow using primary LLM"""
        try:
            response = await self.primary_llm.agenerate_prompt([prompt])
            return response.generations[0][0].text
        except Exception as e:
            # Fallback to alternate model
            return await self.router.acompletion(
                model="fallback",
                messages=[{"role": "user", "content": prompt}]
            )
    
    async def classify_intent(self, text: str) -> dict:
        """Fast classification using mini model"""
        prompt = f"""Classify the workflow intent. Return JSON:
{{"intent": "create|optimize|validate", "confidence": 0.0-1.0}}

Text: {text}"""
        
        response = await self.fast_llm.agenerate_prompt([prompt])
        return json.loads(response.generations[0][0].text)

# Singleton instance
azure_openai = AzureOpenAIClient()
```

### 3.2 LLM Configuration

```python
# core/settings.py
from pydantic_settings import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    # Azure OpenAI
    AZURE_OAI_ENDPOINT: str
    AZURE_OAI_KEY: str
    AZURE_OAI_MODEL: str = "gpt-4o"
    AZURE_OAI_DEPLOYMENT: str = "workflow-agent"
    AZURE_OAI_API_VERSION: str = "2024-05-01-preview"
    
    # LiteLLM
    LITELLM_ENABLED: bool = True
    LITELLM_API_KEY: Optional[str] = None
    LITELLM_PROXY_URL: Optional[str] = None
    FALLBACK_MODEL: str = "claude-sonnet-4-6-20250514"
    ANTHROPIC_API_KEY: Optional[str] = None
    
    # LangChain
    LANGCHAIN_TRACING_V2: bool = True
    LANGCHAIN_ENDPOINT: Optional[str] = None
    LANGCHAIN_API_KEY: Optional[str] = None
    
    # Embeddings
    EMBEDDING_MODEL: str = "text-embedding-3-large"
    EMBEDDING_DIMENSION: int = 3072
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    REDIS_CACHE_TTL: int = 3600  # 1 hour
    
    # PostgreSQL (pgvector)
    POSTGRES_URL: str
    PG_VECTOR_COLLECTION: str = "industry_knowledge"
    
    # Langfuse
    LANGFUSE_PUBLIC_KEY: Optional[str] = None
    LANGFUSE_SECRET_KEY: Optional[str] = None
    LANGFUSE_HOST: Optional[str] = None
    
    # API
    DEBUG: bool = False
    LOG_LEVEL: str = "INFO"
    CORS_ORIGINS: list = ["https://app.example.com", "http://localhost:3000"]
    
    class Config:
        env_file = ".env"

settings = Settings()
```

---

## 4. LangGraph Workflow Agent

### 4.1 LangGraph State Machine

```python
# agents/workflow_agent.py
from typing import Dict, Any, List, Optional
from langgraph.graph import StateGraph, START, END
from langgraph.types import StateSnapshot
from pydantic import BaseModel
import json

# State Definition
class WorkflowState(BaseModel):
    """State object that flows through the LangGraph"""
    
    # Input
    user_description: str
    available_integrations: List[str] = []
    user_preferences: Dict[str, Any] = {}
    
    # Processing
    intent: Optional[str] = None
    expanded_description: Optional[str] = None
    nodes_draft: Optional[List[Dict]] = None
    edges_draft: Optional[List[Dict]] = None
    
    # Validation
    validation_errors: List[str] = []
    validation_warnings: List[str] = []
    is_valid: bool = False
    
    # Final output
    workflow_json: Optional[Dict] = None
    explanation: Optional[str] = None
    
    # Metadata
    message_history: List[Dict] = []
    execution_logs: List[str] = []
    step_count: int = 0

class WorkflowAgent:
    """LangGraph-based workflow generation agent"""
    
    def __init__(self, llm_service, rag_service, validation_service):
        self.llm_service = llm_service
        self.rag_service = rag_service
        self.validation_service = validation_service
        self.graph = self._build_graph()
    
    def _build_graph(self) -> StateGraph:
        """Build the LangGraph state machine"""
        graph = StateGraph(WorkflowState)
        
        # Add nodes
        graph.add_node("classify_intent", self.classify_intent_node)
        graph.add_node("retrieve_context", self.retrieve_context_node)
        graph.add_node("expand_description", self.expand_description_node)
        graph.add_node("draft_workflow", self.draft_workflow_node)
        graph.add_node("validate", self.validate_node)
        graph.add_node("optimize", self.optimize_node)
        graph.add_node("export", self.export_node)
        graph.add_node("error_handler", self.error_handler_node)
        
        # Add edges
        graph.add_edge(START, "classify_intent")
        graph.add_edge("classify_intent", "retrieve_context")
        graph.add_edge("retrieve_context", "expand_description")
        graph.add_edge("expand_description", "draft_workflow")
        graph.add_edge("draft_workflow", "validate")
        
        # Conditional edge: if validation fails, expand more
        graph.add_conditional_edges(
            "validate",
            self._should_optimize,
            {
                "optimize": "optimize",
                "error": "error_handler",
                "export": "export"
            }
        )
        
        graph.add_edge("optimize", "export")
        graph.add_edge("export", END)
        graph.add_edge("error_handler", END)
        
        return graph.compile()
    
    # Node implementations
    async def classify_intent_node(self, state: WorkflowState) -> WorkflowState:
        """Classify user intent"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Classifying intent")
        
        intent_result = await self.llm_service.classify_intent(state.user_description)
        state.intent = intent_result.get("intent", "create")
        
        return state
    
    async def retrieve_context_node(self, state: WorkflowState) -> WorkflowState:
        """Retrieve relevant industry knowledge from RAG"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Retrieving context from RAG")
        
        # Hybrid retrieval: dense + BM25
        context_docs = await self.rag_service.retrieve(
            query=state.user_description,
            top_k=8,
            include_metadata=True
        )
        
        state.message_history.append({
            "role": "system",
            "content": f"Retrieved {len(context_docs)} relevant documents from knowledge base"
        })
        
        return state
    
    async def expand_description_node(self, state: WorkflowState) -> WorkflowState:
        """Use LLM to expand user description with details"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Expanding description")
        
        expansion_prompt = f"""
Expand and clarify this workflow description:
"{state.user_description}"

Consider:
1. Missing details or edge cases
2. Integration points needed
3. Data transformations required
4. Error handling scenarios
5. Compliance requirements

Provide expanded description in 3-5 sentences.
"""
        
        expanded = await self.llm_service.generate_workflow(expansion_prompt)
        state.expanded_description = expanded
        
        return state
    
    async def draft_workflow_node(self, state: WorkflowState) -> WorkflowState:
        """Generate initial workflow structure"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Drafting workflow structure")
        
        draft_prompt = f"""
Generate a workflow based on this description:
{state.expanded_description}

Available integrations: {', '.join(state.available_integrations)}

Return valid JSON with this exact structure:
{{
  "nodes": [
    {{
      "id": "unique_id",
      "type": "Task|Decision|Loop|Trigger|End|Webhook",
      "label": "Node name",
      "description": "What it does",
      "configuration": {{}}
    }}
  ],
  "edges": [
    {{
      "source": "node_id",
      "target": "node_id",
      "type": "success|failure|conditional"
    }}
  ]
}}

Generate 5-10 nodes minimum. Include error handling paths.
"""
        
        response = await self.llm_service.generate_workflow(draft_prompt)
        state.workflow_json = json.loads(response)
        
        # Extract nodes/edges for validation
        state.nodes_draft = state.workflow_json.get("nodes", [])
        state.edges_draft = state.workflow_json.get("edges", [])
        
        return state
    
    async def validate_node(self, state: WorkflowState) -> WorkflowState:
        """Validate workflow structure"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Validating workflow")
        
        validation_result = await self.validation_service.validate(
            nodes=state.nodes_draft,
            edges=state.edges_draft,
            integrations=state.available_integrations
        )
        
        state.is_valid = validation_result.is_valid
        state.validation_errors = validation_result.errors
        state.validation_warnings = validation_result.warnings
        
        return state
    
    async def optimize_node(self, state: WorkflowState) -> WorkflowState:
        """Optimize workflow for performance"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Optimizing workflow")
        
        optimization_prompt = f"""
Optimize this workflow for performance and maintainability:
{json.dumps(state.workflow_json, indent=2)}

Provide optimization suggestions as JSON:
{{
  "suggestions": [
    {{"nodeId": "id", "improvement": "description"}}
  ],
  "optimizedWorkflow": {{ ... }}
}}
"""
        
        optimization = await self.llm_service.generate_workflow(optimization_prompt)
        optimized = json.loads(optimization)
        state.workflow_json = optimized.get("optimizedWorkflow", state.workflow_json)
        
        return state
    
    async def export_node(self, state: WorkflowState) -> WorkflowState:
        """Export final workflow"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Exporting workflow")
        
        explanation_prompt = f"""
Summarize this workflow for the user in 2-3 sentences:
{json.dumps(state.workflow_json, indent=2)}

Explain the key steps and any special handling.
"""
        
        state.explanation = await self.llm_service.generate_workflow(explanation_prompt)
        
        return state
    
    async def error_handler_node(self, state: WorkflowState) -> WorkflowState:
        """Handle validation errors"""
        state.step_count += 1
        state.execution_logs.append(f"Step {state.step_count}: Error handling")
        
        # Log errors but don't fail - return partial result
        state.execution_logs.append(f"Validation errors: {state.validation_errors}")
        
        return state
    
    def _should_optimize(self, state: WorkflowState) -> str:
        """Decide next step based on validation"""
        if state.validation_errors:
            return "error"
        elif state.user_preferences.get("optimize", True):
            return "optimize"
        else:
            return "export"
    
    async def invoke(self, user_description: str, **kwargs) -> Dict[str, Any]:
        """Run the agent"""
        initial_state = WorkflowState(
            user_description=user_description,
            available_integrations=kwargs.get("integrations", []),
            user_preferences=kwargs.get("preferences", {})
        )
        
        # Stream results as they happen
        result = None
        async for snapshot in self.graph.astream(initial_state):
            result = snapshot
            yield snapshot  # Stream to client
        
        return result
```

### 4.2 LangGraph Integration

```python
# services/langgraph_agent.py
from typing import AsyncGenerator, Dict, Any
from app.agents.workflow_agent import WorkflowAgent

class LangGraphService:
    """Service wrapper for LangGraph agent"""
    
    def __init__(self, llm_service, rag_service, validation_service):
        self.agent = WorkflowAgent(llm_service, rag_service, validation_service)
    
    async def generate_workflow_stream(
        self,
        description: str,
        **kwargs
    ) -> AsyncGenerator[Dict[str, Any], None]:
        """Generate workflow with streaming"""
        async for chunk in self.agent.invoke(description, **kwargs):
            yield {
                "step": chunk.step_count,
                "status": self._extract_status(chunk),
                "data": chunk.model_dump() if hasattr(chunk, 'model_dump') else chunk
            }
    
    def _extract_status(self, state) -> str:
        """Extract current step name"""
        if state.intent:
            return "classify"
        if state.expanded_description:
            return "expand"
        if state.nodes_draft:
            return "draft"
        if state.validation_errors:
            return "validate_error"
        if state.workflow_json:
            return "complete"
        return "processing"
```

---

## 5. RAG System

### 5.1 pgvector RAG Setup

```python
# integrations/pgvector_rag.py
import asyncpg
from langchain_postgres import PGVector
from langchain_openai import AzureOpenAIEmbeddings
from rank_bm25 import BM25Okapi
import numpy as np

class RAGService:
    """Retrieval-Augmented Generation using pgvector + BM25"""
    
    def __init__(self, postgres_url: str):
        self.postgres_url = postgres_url
        self.vectorstore = None
        self.embedding_model = None
        self.collection_name = "industry_knowledge"
        self.bm25_index = None
        self.corpus = []
    
    async def initialize(self):
        """Initialize RAG components"""
        # Initialize embeddings
        self.embedding_model = AzureOpenAIEmbeddings(
            model="text-embedding-3-large",
            deployment_id="embedding",
            api_key=os.getenv("AZURE_OAI_KEY"),
            api_base=os.getenv("AZURE_OAI_ENDPOINT")
        )
        
        # Initialize pgvector store
        self.vectorstore = PGVector(
            connection_string=self.postgres_url,
            embedding_function=self.embedding_model,
            collection_name=self.collection_name
        )
        
        # Initialize BM25 index
        await self._build_bm25_index()
    
    async def _build_bm25_index(self):
        """Build BM25 index from corpus"""
        # Fetch all documents from database
        conn = await asyncpg.connect(self.postgres_url)
        
        docs = await conn.fetch(f"""
            SELECT id, content, metadata FROM langchain_pg_collection
            WHERE collection_id = (
                SELECT uuid FROM langchain_pg_collection_name 
                WHERE name = $1
            )
        """, self.collection_name)
        
        # Build corpus and BM25 index
        self.corpus = [doc['content'] for doc in docs]
        tokenized_corpus = [doc.split() for doc in self.corpus]
        self.bm25_index = BM25Okapi(tokenized_corpus)
        
        await conn.close()
    
    async def retrieve(
        self,
        query: str,
        top_k: int = 8,
        include_metadata: bool = True
    ) -> List[Dict[str, Any]]:
        """Hybrid retrieval: dense (pgvector) + sparse (BM25)"""
        
        # 1. Dense retrieval using pgvector
        dense_results = await self._dense_retrieve(query, top_k=5)
        
        # 2. Sparse retrieval using BM25
        sparse_results = await self._sparse_retrieve(query, top_k=5)
        
        # 3. Combine and rerank
        combined = self._combine_results(dense_results, sparse_results)
        reranked = await self._rerank_with_cross_encoder(query, combined, top_k)
        
        return reranked
    
    async def _dense_retrieve(self, query: str, top_k: int = 5) -> List[Dict]:
        """Dense vector retrieval using pgvector"""
        results = self.vectorstore.similarity_search_with_score(
            query,
            k=top_k
        )
        
        return [
            {
                "content": doc.page_content,
                "score": score,
                "type": "dense",
                "metadata": doc.metadata
            }
            for doc, score in results
        ]
    
    async def _sparse_retrieve(self, query: str, top_k: int = 5) -> List[Dict]:
        """Sparse retrieval using BM25"""
        query_tokens = query.split()
        scores = self.bm25_index.get_scores(query_tokens)
        
        top_indices = np.argsort(scores)[-top_k:][::-1]
        
        return [
            {
                "content": self.corpus[idx],
                "score": float(scores[idx]),
                "type": "sparse",
                "metadata": {}
            }
            for idx in top_indices if scores[idx] > 0
        ]
    
    def _combine_results(
        self,
        dense: List[Dict],
        sparse: List[Dict]
    ) -> List[Dict]:
        """Combine dense and sparse results"""
        # Reciprocal rank fusion
        rrf_scores = {}
        
        for i, result in enumerate(dense):
            key = result["content"]
            rrf_scores[key] = rrf_scores.get(key, 0) + 1 / (i + 60)
        
        for i, result in enumerate(sparse):
            key = result["content"]
            rrf_scores[key] = rrf_scores.get(key, 0) + 1 / (i + 60)
        
        # Sort by combined score
        combined = [
            {
                "content": content,
                "score": score,
                "type": "hybrid"
            }
            for content, score in sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)
        ]
        
        return combined
    
    async def _rerank_with_cross_encoder(
        self,
        query: str,
        candidates: List[Dict],
        top_k: int = 8
    ) -> List[Dict]:
        """Rerank using cross-encoder model"""
        from sentence_transformers import CrossEncoder
        
        reranker = CrossEncoder(
            'cross-encoder/ms-marco-MiniLM-L-12-v2'
        )
        
        # Score all candidates
        pairs = [[query, doc["content"]] for doc in candidates]
        scores = reranker.predict(pairs)
        
        # Sort by reranked scores
        scored_candidates = [
            {**doc, "rerank_score": float(score)}
            for doc, score in zip(candidates, scores)
        ]
        
        return sorted(
            scored_candidates,
            key=lambda x: x["rerank_score"],
            reverse=True
        )[:top_k]
    
    async def add_document(
        self,
        content: str,
        metadata: Dict[str, Any] = None
    ) -> str:
        """Add document to RAG system"""
        doc_id = await self.vectorstore.add_texts(
            [content],
            metadatas=[metadata or {}]
        )
        
        # Update BM25 index
        self.corpus.append(content)
        await self._build_bm25_index()
        
        return doc_id[0]
    
    async def search_by_metadata(
        self,
        metadata_filter: Dict[str, Any],
        top_k: int = 8
    ) -> List[Dict]:
        """Search documents by metadata"""
        conn = await asyncpg.connect(self.postgres_url)
        
        # Build WHERE clause
        conditions = []
        for key, value in metadata_filter.items():
            conditions.append(f"metadata ->> '{key}' = '{value}'")
        
        where_clause = " AND ".join(conditions) if conditions else "1=1"
        
        docs = await conn.fetch(f"""
            SELECT content, metadata FROM langchain_pg_collection
            WHERE collection_id = (
                SELECT uuid FROM langchain_pg_collection_name 
                WHERE name = $1
            )
            AND {where_clause}
            LIMIT $2
        """, self.collection_name, top_k)
        
        await conn.close()
        
        return [
            {"content": doc["content"], "metadata": doc["metadata"]}
            for doc in docs
        ]
```

### 5.2 Knowledge Base Population

```python
# services/knowledge_base_service.py
class KnowledgeBaseService:
    """Manage industry knowledge base"""
    
    def __init__(self, rag_service):
        self.rag_service = rag_service
    
    async def populate_templates(self):
        """Populate with workflow templates"""
        templates = [
            {
                "content": "E-commerce order processing: Receive order → Validate → Process payment → Update inventory → Send confirmation",
                "metadata": {"type": "template", "domain": "ecommerce", "complexity": "medium"}
            },
            {
                "content": "Data pipeline: Extract from API → Transform → Load to warehouse → Validate quality → Notify",
                "metadata": {"type": "template", "domain": "data", "complexity": "high"}
            },
            # ... more templates
        ]
        
        for template in templates:
            await self.rag_service.add_document(
                template["content"],
                template["metadata"]
            )
    
    async def populate_compliance_rules(self):
        """Populate compliance and regulation rules"""
        rules = [
            {
                "content": "GDPR: Ensure data encryption at rest and in transit, implement data retention policies, provide data export functionality",
                "metadata": {"type": "compliance", "regulation": "GDPR", "region": "EU"}
            },
            # ... more rules
        ]
        
        for rule in rules:
            await self.rag_service.add_document(
                rule["content"],
                rule["metadata"]
            )
    
    async def populate_role_taxonomy(self):
        """Populate role and responsibility taxonomy"""
        roles = [
            {
                "content": "Data Analyst: Validates data quality, creates reports, identifies trends, collaborates with teams",
                "metadata": {"type": "role", "domain": "analytics", "level": "mid"}
            },
            # ... more roles
        ]
        
        for role in roles:
            await self.rag_service.add_document(
                role["content"],
                role["metadata"]
            )
```

---

## 6. Async Job Queue

### 6.1 Hangfire Integration

```csharp
// .NET Backend - WorkflowAIService.cs
using Hangfire;
using System.Threading.Tasks;

public class WorkflowAIService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<WorkflowAIService> _logger;
    
    public WorkflowAIService(HttpClient httpClient, ILogger<WorkflowAIService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }
    
    [Queue("ai_generation")]
    public async Task GenerateAsync(WorkflowGenerationRequest request, CancellationToken cancellationToken)
    {
        try
        {
            var jobId = JobStorage.Current.GetConnection().GetJobParameter(
                BackgroundJob.Current.Id,
                "JobId"
            );
            
            _logger.LogInformation("Starting AI workflow generation for job {jobId}", jobId);
            
            // Call Python AI Engine
            var content = new StringContent(
                JsonSerializer.Serialize(request),
                Encoding.UTF8,
                "application/json"
            );
            
            var response = await _httpClient.PostAsync(
                "http://ai-engine:8000/api/v1/workflows/generate",
                content,
                cancellationToken
            );
            
            response.EnsureSuccessStatusCode();
            
            var result = await response.Content.ReadAsStringAsync(cancellationToken);
            var workflow = JsonSerializer.Deserialize<WorkflowDto>(result);
            
            // Save to database
            await _workflowRepository.CreateAsync(workflow);
            
            _logger.LogInformation("Workflow generation completed for job {jobId}", jobId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Workflow generation failed");
            throw;
        }
    }
}

// Controller
[ApiController]
[Route("api/v1/workflows")]
public class WorkflowController
{
    private readonly IBackgroundJobClient _backgroundJobClient;
    
    [HttpPost("generate-async")]
    public IActionResult GenerateAsync([FromBody] WorkflowGenerationRequest request)
    {
        // Enqueue background job
        var jobId = _backgroundJobClient.Enqueue<WorkflowAIService>(
            service => service.GenerateAsync(request, CancellationToken.None)
        );
        
        return Accepted(new { jobId });
    }
    
    [HttpGet("jobs/{jobId}/status")]
    public IActionResult GetJobStatus(string jobId)
    {
        var job = JobStorage.Current.GetConnection().GetJobData(jobId);
        
        if (job == null)
            return NotFound();
        
        return Ok(new
        {
            jobId,
            state = job.State,
            createdAt = job.CreatedAt,
            expireAt = job.ExpireAt
        });
    }
}
```

### 6.2 Job Status Tracking

```python
# services/job_service.py
import asyncio
from typing import Optional, Dict, Any
import aioredis

class JobService:
    """Manage long-running job tracking"""
    
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.redis = None
    
    async def initialize(self):
        """Initialize Redis connection"""
        self.redis = await aioredis.from_url(self.redis_url)
    
    async def create_job(
        self,
        job_id: str,
        description: str
    ) -> Dict[str, Any]:
        """Create new job tracking entry"""
        job_data = {
            "id": job_id,
            "status": "pending",
            "description": description,
            "progress": 0,
            "result": None,
            "error": None
        }
        
        await self.redis.setex(
            f"job:{job_id}",
            86400,  # 24 hours
            json.dumps(job_data)
        )
        
        return job_data
    
    async def update_job_progress(
        self,
        job_id: str,
        status: str,
        progress: int = None,
        message: str = None
    ):
        """Update job progress"""
        job_data = await self.get_job(job_id)
        
        job_data["status"] = status
        if progress is not None:
            job_data["progress"] = progress
        if message:
            job_data["message"] = message
        
        await self.redis.setex(
            f"job:{job_id}",
            86400,
            json.dumps(job_data)
        )
    
    async def complete_job(
        self,
        job_id: str,
        result: Dict[str, Any]
    ):
        """Mark job as complete"""
        job_data = await self.get_job(job_id)
        
        job_data["status"] = "completed"
        job_data["result"] = result
        
        await self.redis.setex(
            f"job:{job_id}",
            86400,
            json.dumps(job_data)
        )
    
    async def fail_job(
        self,
        job_id: str,
        error: str
    ):
        """Mark job as failed"""
        job_data = await self.get_job(job_id)
        
        job_data["status"] = "failed"
        job_data["error"] = error
        
        await self.redis.setex(
            f"job:{job_id}",
            86400,
            json.dumps(job_data)
        )
    
    async def get_job(self, job_id: str) -> Optional[Dict[str, Any]]:
        """Get job status"""
        data = await self.redis.get(f"job:{job_id}")
        return json.loads(data) if data else None
```

---

## 7. LLM Gateway

### 7.1 LiteLLM Proxy Setup

```python
# integrations/litellm_gateway.py
import litellm
from litellm import Router
from typing import Dict, Any, List

class LiteLLMGateway:
    """LLM proxy gateway with fallback and caching"""
    
    def __init__(self, config_path: str = None):
        self.router = None
        self.config_path = config_path
        self.cache = {}
    
    async def initialize(self):
        """Initialize LiteLLM router"""
        litellm.enable_logging = True
        
        # Model configuration
        model_list = [
            {
                "model_name": "gpt-4o",
                "litellm_params": {
                    "model": "azure/gpt-4o",
                    "api_key": os.getenv("AZURE_OAI_KEY"),
                    "api_base": os.getenv("AZURE_OAI_ENDPOINT"),
                    "api_version": "2024-05-01-preview"
                }
            },
            {
                "model_name": "gpt-4o-mini",
                "litellm_params": {
                    "model": "azure/gpt-4o-mini",
                    "api_key": os.getenv("AZURE_OAI_KEY"),
                    "api_base": os.getenv("AZURE_OAI_ENDPOINT"),
                    "api_version": "2024-05-01-preview"
                }
            },
            {
                "model_name": "claude-sonnet",
                "litellm_params": {
                    "model": "claude-sonnet-4-6-20250514",
                    "api_key": os.getenv("ANTHROPIC_API_KEY")
                }
            }
        ]
        
        self.router = Router(
            model_list=model_list,
            timeout=60,
            num_retries=3,
            fallbacks=[
                {"gpt-4o": ["gpt-4o-mini", "claude-sonnet"]},
                {"gpt-4o-mini": ["claude-sonnet"]},
            ]
        )
    
    async def completion(
        self,
        model: str,
        messages: List[Dict],
        temperature: float = 0.7,
        max_tokens: int = 2048,
        **kwargs
    ) -> str:
        """LLM completion with fallback"""
        try:
            response = await self.router.acompletion(
                model=model,
                messages=messages,
                temperature=temperature,
                max_tokens=max_tokens,
                **kwargs
            )
            
            return response.choices[0].message.content
        except Exception as e:
            # Fallback handled by router
            raise
    
    async def structured_completion(
        self,
        model: str,
        messages: List[Dict],
        response_format: Dict[str, Any],
        **kwargs
    ) -> Dict[str, Any]:
        """Structured output completion"""
        response = await self.completion(
            model=model,
            messages=messages,
            response_format=response_format,
            **kwargs
        )
        
        return json.loads(response)
    
    def get_model_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        """Calculate cost for a completion"""
        return litellm.completion_cost(
            model=model,
            prompt_tokens=input_tokens,
            completion_tokens=output_tokens
        )
```

### 7.2 LiteLLM Configuration File

```yaml
# litellm_config.yaml
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: azure/gpt-4o
      api_key: ${AZURE_OAI_KEY}
      api_base: ${AZURE_OAI_ENDPOINT}
      api_version: 2024-05-01-preview
    
  - model_name: gpt-4o-mini
    litellm_params:
      model: azure/gpt-4o-mini
      api_key: ${AZURE_OAI_KEY}
      api_base: ${AZURE_OAI_ENDPOINT}
      api_version: 2024-05-01-preview
    
  - model_name: claude-sonnet
    litellm_params:
      model: claude-sonnet-4-6-20250514
      api_key: ${ANTHROPIC_API_KEY}

fallback_options:
  gpt-4o:
    - gpt-4o-mini
    - claude-sonnet
  gpt-4o-mini:
    - claude-sonnet

router_settings:
  timeout: 60
  num_retries: 3
  enable_logging: true
  log_level: "DEBUG"

cache_config:
  type: "redis"
  redis_url: ${REDIS_URL}
  ttl: 3600
```

---

## 8. Observability & Tracing

### 8.1 Langfuse Integration

```python
# integrations/langfuse_tracing.py
from langfuse import Langfuse
from langfuse.callback import CallbackHandler
import os

class LangfuseObservability:
    """Langfuse integration for AI tracing"""
    
    def __init__(self):
        self.langfuse = None
        self.callback_handler = None
    
    def initialize(self):
        """Initialize Langfuse client"""
        self.langfuse = Langfuse(
            public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
            secret_key=os.getenv("LANGFUSE_SECRET_KEY"),
            host=os.getenv("LANGFUSE_HOST", "http://localhost:3000")
        )
        
        self.callback_handler = CallbackHandler(
            public_key=os.getenv("LANGFUSE_PUBLIC_KEY"),
            secret_key=os.getenv("LANGFUSE_SECRET_KEY"),
            host=os.getenv("LANGFUSE_HOST", "http://localhost:3000")
        )
    
    def get_callback_handler(self) -> CallbackHandler:
        """Get callback handler for LangGraph"""
        return self.callback_handler
    
    def trace_workflow_generation(
        self,
        user_id: str,
        workflow_description: str,
        workflow_result: Dict[str, Any]
    ):
        """Trace workflow generation"""
        trace = self.langfuse.trace(
            name="workflow_generation",
            user_id=user_id,
            metadata={
                "workflow_type": "generation",
                "description_length": len(workflow_description),
                "node_count": len(workflow_result.get("nodes", []))
            }
        )
        
        # Log input
        trace.span(
            name="input",
            input={"description": workflow_description}
        )
        
        # Log output
        trace.span(
            name="output",
            output=workflow_result
        )
        
        trace.save()
    
    def trace_llm_call(
        self,
        model: str,
        prompt: str,
        response: str,
        tokens: Dict[str, int]
    ):
        """Trace LLM call"""
        trace = self.langfuse.trace(
            name="llm_call",
            metadata={
                "model": model,
                "input_tokens": tokens.get("prompt_tokens", 0),
                "output_tokens": tokens.get("completion_tokens", 0)
            }
        )
        
        trace.span(
            name="generation",
            input={"prompt": prompt},
            output={"response": response},
            metadata={
                "tokens": tokens,
                "cost": tokens.get("cost", 0)
            }
        )
        
        trace.save()
    
    def get_analytics_dashboard(self) -> str:
        """Get Langfuse dashboard URL"""
        return f"{os.getenv('LANGFUSE_HOST', 'http://localhost:3000')}/dashboard"
```

### 8.2 Structured Logging

```python
# utils/logger.py
import logging
import json
from datetime import datetime

class StructuredLogger:
    """Structured logging for observability"""
    
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
    
    def log_event(
        self,
        event: str,
        level: str = "INFO",
        **kwargs
    ):
        """Log structured event"""
        log_data = {
            "timestamp": datetime.utcnow().isoformat(),
            "event": event,
            "level": level,
            **kwargs
        }
        
        if level == "INFO":
            self.logger.info(json.dumps(log_data))
        elif level == "ERROR":
            self.logger.error(json.dumps(log_data))
        elif level == "WARNING":
            self.logger.warning(json.dumps(log_data))
        elif level == "DEBUG":
            self.logger.debug(json.dumps(log_data))
    
    def log_workflow_event(
        self,
        workflow_id: str,
        event: str,
        status: str,
        metadata: Dict[str, Any] = None
    ):
        """Log workflow event"""
        self.log_event(
            event=event,
            level="INFO",
            workflow_id=workflow_id,
            status=status,
            metadata=metadata or {}
        )
    
    def log_llm_event(
        self,
        model: str,
        event: str,
        tokens: Dict[str, int],
        latency_ms: int
    ):
        """Log LLM event"""
        self.log_event(
            event=event,
            level="INFO",
            model=model,
            tokens=tokens,
            latency_ms=latency_ms
        )

# Usage
logger = StructuredLogger("ai_engine")
logger.log_workflow_event(
    workflow_id="wf-123",
    event="generation_started",
    status="running"
)
```

---

## 9. API Endpoints

### 9.1 Workflow Generation Endpoint

```python
# api/v1/endpoints/workflow_generation.py
from fastapi import APIRouter, HTTPException, BackgroundTasks
from pydantic import BaseModel
import uuid

router = APIRouter()

class WorkflowGenerationRequest(BaseModel):
    description: str
    integrations: List[str] = []
    preferences: Dict[str, Any] = {}

class WorkflowGenerationResponse(BaseModel):
    workflow_id: str
    nodes: List[Dict]
    edges: List[Dict]
    explanation: str
    metadata: Dict[str, Any]

@router.post("/workflows/generate")
async def generate_workflow(
    request: WorkflowGenerationRequest,
    background_tasks: BackgroundTasks
) -> WorkflowGenerationResponse:
    """Generate workflow from description with streaming"""
    
    workflow_id = str(uuid.uuid4())
    
    # Get services from dependency injection
    llm_service = get_llm_service()
    rag_service = get_rag_service()
    langgraph_service = get_langgraph_service()
    job_service = get_job_service()
    
    try:
        # Create job entry
        await job_service.create_job(
            workflow_id,
            f"Generate workflow: {request.description}"
        )
        
        # Stream results from LangGraph agent
        final_state = None
        async for state_update in langgraph_service.generate_workflow_stream(
            description=request.description,
            integrations=request.integrations,
            preferences=request.preferences
        ):
            await job_service.update_job_progress(
                workflow_id,
                status=state_update["status"],
                progress=state_update.get("progress", 0)
            )
            final_state = state_update["data"]
        
        # Complete job
        result = {
            "nodes": final_state.nodes_draft,
            "edges": final_state.edges_draft,
            "explanation": final_state.explanation
        }
        
        await job_service.complete_job(workflow_id, result)
        
        return WorkflowGenerationResponse(
            workflow_id=workflow_id,
            nodes=final_state.nodes_draft,
            edges=final_state.edges_draft,
            explanation=final_state.explanation,
            metadata={
                "steps": final_state.step_count,
                "warnings": final_state.validation_warnings,
                "execution_logs": final_state.execution_logs
            }
        )
    
    except Exception as e:
        await job_service.fail_job(workflow_id, str(e))
        raise HTTPException(status_code=500, detail=str(e))

@router.get("/workflows/generate/{workflow_id}/stream")
async def stream_workflow_generation(workflow_id: str):
    """Stream workflow generation progress"""
    job_service = get_job_service()
    
    async def event_generator():
        while True:
            job_data = await job_service.get_job(workflow_id)
            if not job_data:
                break
            
            yield f"data: {json.dumps(job_data)}\n\n"
            
            if job_data["status"] in ["completed", "failed"]:
                break
            
            await asyncio.sleep(1)
    
    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream"
    )

@router.post("/workflows/optimize")
async def optimize_workflow(request: WorkflowOptimizeRequest):
    """Optimize existing workflow"""
    # Implementation
    pass

@router.post("/workflows/validate")
async def validate_workflow(request: WorkflowValidateRequest):
    """Validate workflow structure"""
    # Implementation
    pass

@router.post("/chat")
async def chat_with_ai(request: ChatRequest):
    """Interactive chat with AI assistant"""
    # Implementation
    pass
```

---

## 10. Error Handling

### 10.1 Custom Exceptions

```python
# core/exceptions.py
class AIEngineException(Exception):
    """Base exception for AI Engine"""
    def __init__(self, message: str, code: str = None, details: Dict = None):
        self.message = message
        self.code = code or "AI_ENGINE_ERROR"
        self.details = details or {}
        super().__init__(message)

class LLMException(AIEngineException):
    """LLM related errors"""
    pass

class ValidationException(AIEngineException):
    """Workflow validation errors"""
    pass

class RAGException(AIEngineException):
    """RAG system errors"""
    pass

class JobException(AIEngineException):
    """Job queue errors"""
    pass

# middleware/error_handler.py
from fastapi import Request, status
from fastapi.responses import JSONResponse

async def error_middleware(request: Request, call_next):
    try:
        response = await call_next(request)
        return response
    except AIEngineException as e:
        return JSONResponse(
            status_code=status.HTTP_400_BAD_REQUEST,
            content={
                "error": {
                    "code": e.code,
                    "message": e.message,
                    "details": e.details
                }
            }
        )
    except Exception as e:
        return JSONResponse(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            content={
                "error": {
                    "code": "INTERNAL_SERVER_ERROR",
                    "message": str(e)
                }
            }
        )
```

---

## 11. Performance Optimization

### 11.1 Caching Strategy

```python
# integrations/redis_cache.py
import aioredis
import json

class RedisCache:
    """Redis caching layer"""
    
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self.redis = None
    
    async def connect(self):
        """Connect to Redis"""
        self.redis = await aioredis.from_url(self.redis_url)
    
    async def get(self, key: str, deserialize: bool = True):
        """Get from cache"""
        value = await self.redis.get(key)
        if value and deserialize:
            return json.loads(value)
        return value
    
    async def set(self, key: str, value: Any, ttl: int = 3600):
        """Set cache with TTL"""
        serialized = json.dumps(value) if not isinstance(value, str) else value
        await self.redis.setex(key, ttl, serialized)
    
    async def semantic_cache(self, query: str, embedding: List[float]) -> Optional[Dict]:
        """Semantic caching for LLM results"""
        # Check if similar query exists in cache
        cache_key = f"semantic:{hash(query)}"
        return await self.get(cache_key)
```

### 11.2 Batch Processing

```python
# services/batch_service.py
class BatchService:
    """Handle batch workflow generation"""
    
    async def generate_batch(
        self,
        descriptions: List[str],
        batch_size: int = 5
    ) -> List[Dict]:
        """Generate multiple workflows efficiently"""
        results = []
        
        for i in range(0, len(descriptions), batch_size):
            batch = descriptions[i:i+batch_size]
            
            # Process batch concurrently
            batch_results = await asyncio.gather(*[
                self.langgraph_service.invoke(desc)
                for desc in batch
            ])
            
            results.extend(batch_results)
        
        return results
```

---

## 12. Security Considerations

### 12.1 Input Validation

```python
# utils/validators.py
from pydantic import BaseModel, validator

class WorkflowInput(BaseModel):
    description: str
    integrations: List[str] = []
    
    @validator('description')
    def validate_description(cls, v):
        if not v or len(v) < 10:
            raise ValueError("Description must be at least 10 characters")
        if len(v) > 5000:
            raise ValueError("Description must not exceed 5000 characters")
        return v
    
    @validator('integrations')
    def validate_integrations(cls, v):
        allowed = {"http", "database", "api", "webhook"}
        for integration in v:
            if integration not in allowed:
                raise ValueError(f"Unknown integration: {integration}")
        return v
```

### 12.2 Rate Limiting

```python
# middleware/rate_limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.get("/api/v1/workflows/generate")
@limiter.limit("10/minute")
async def generate_workflow(request: Request):
    # Implementation
    pass
```

### 12.3 API Key Management

```python
# core/security.py
from fastapi import HTTPException, Header

async def verify_api_key(x_api_key: str = Header(...)):
    """Verify API key"""
    valid_keys = os.getenv("VALID_API_KEYS", "").split(",")
    if x_api_key not in valid_keys:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return x_api_key
```

---

## 13. Testing Strategy

### 13.1 Unit Tests

```python
# tests/test_agents.py
import pytest
from app.agents.workflow_agent import WorkflowAgent

@pytest.fixture
def workflow_agent(llm_service, rag_service, validation_service):
    return WorkflowAgent(llm_service, rag_service, validation_service)

@pytest.mark.asyncio
async def test_workflow_generation(workflow_agent):
    """Test workflow generation"""
    state = await workflow_agent.invoke(
        "Create a simple order processing workflow"
    )
    
    assert state.workflow_json is not None
    assert len(state.nodes_draft) > 0
    assert len(state.edges_draft) > 0

@pytest.mark.asyncio
async def test_validation_integration(workflow_agent):
    """Test validation during generation"""
    state = await workflow_agent.invoke(
        "Create workflow with missing integrations"
    )
    
    # Should handle gracefully
    assert state is not None
```

### 13.2 Integration Tests

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_health_check():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

@pytest.mark.asyncio
async def test_workflow_generation_api():
    response = await client.post(
        "/api/v1/workflows/generate",
        json={
            "description": "Create an e-commerce order workflow",
            "integrations": ["http", "database"]
        }
    )
    
    assert response.status_code == 200
    data = response.json()
    assert "workflow_id" in data
    assert "nodes" in data
```

---

## 14. Deployment

### 14.1 Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

# Run application
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 14.2 docker-compose.yml

```yaml
version: '3.8'

services:
  ai-engine:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - AZURE_OAI_ENDPOINT=${AZURE_OAI_ENDPOINT}
      - AZURE_OAI_KEY=${AZURE_OAI_KEY}
      - POSTGRES_URL=postgresql://user:pass@postgres:5432/workflow_db
      - REDIS_URL=redis://redis:6379
      - LANGFUSE_PUBLIC_KEY=${LANGFUSE_PUBLIC_KEY}
      - LANGFUSE_SECRET_KEY=${LANGFUSE_SECRET_KEY}
    depends_on:
      - postgres
      - redis
    networks:
      - workflow-network

  postgres:
    image: pgvector/pgvector:pg16
    environment:
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
      - POSTGRES_DB=workflow_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - workflow-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - workflow-network

  langfuse:
    image: ghcr.io/langfuse/langfuse:latest
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/langfuse
    depends_on:
      - postgres
    networks:
      - workflow-network

volumes:
  postgres_data:
  redis_data:

networks:
  workflow-network:
    driver: bridge
```

### 14.3 Kubernetes Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-engine
  namespace: workflow-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ai-engine
  template:
    metadata:
      labels:
        app: ai-engine
    spec:
      containers:
      - name: ai-engine
        image: acr.azurecr.io/ai-engine:latest
        ports:
        - containerPort: 8000
        env:
        - name: AZURE_OAI_ENDPOINT
          valueFrom:
            secretKeyRef:
              name: ai-secrets
              key: azure-oai-endpoint
        - name: REDIS_URL
          value: "redis://redis-service:6379"
        - name: POSTGRES_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: postgres-url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: ai-engine-service
  namespace: workflow-system
spec:
  type: ClusterIP
  selector:
    app: ai-engine
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ai-engine-hpa
  namespace: workflow-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ai-engine
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## Requirements.txt

```
# Core Framework
fastapi==0.104.1
uvicorn==0.24.0
pydantic==2.5.0
pydantic-settings==2.1.0

# AI/LLM
langchain==0.1.0
langchain-openai==0.0.7
langgraph==0.0.1
langfuse==2.0.0

# Embeddings & RAG
langchain-postgres==0.0.1
pgvector==0.2.0
sentence-transformers==2.2.2
rank-bm25==0.2.2

# LLM Gateway
litellm==1.20.0

# Database
asyncpg==0.29.0
psycopg2-binary==2.9.9

# Cache
redis==5.0.1
aioredis==2.0.1

# Utilities
python-dotenv==1.0.0
httpx==0.25.1
slowapi==0.1.9

# Logging & Observability
python-json-logger==2.0.7

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0

# Development
black==23.12.1
flake8==6.1.0
mypy==1.7.1
```

---

## Environment Variables Template

```bash
# .env.example

# Azure OpenAI
AZURE_OAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OAI_KEY=your-api-key
AZURE_OAI_MODEL=gpt-4o
AZURE_OAI_DEPLOYMENT=workflow-agent
AZURE_OAI_API_VERSION=2024-05-01-preview

# Alternative: Anthropic
ANTHROPIC_API_KEY=sk-ant-xxxxx

# PostgreSQL
POSTGRES_URL=postgresql://user:password@localhost:5432/workflow_db

# Redis
REDIS_URL=redis://localhost:6379

# Langfuse
LANGFUSE_PUBLIC_KEY=pk_xxxxx
LANGFUSE_SECRET_KEY=sk_xxxxx
LANGFUSE_HOST=http://localhost:3000

# API
DEBUG=False
LOG_LEVEL=INFO
CORS_ORIGINS=["https://app.example.com","http://localhost:3000"]

# LiteLLM
LITELLM_ENABLED=True
```

---

## Summary

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Framework** | FastAPI + Uvicorn | High-performance async API |
| **Orchestration** | LangGraph | Multi-step workflow agent |
| **LLM** | Azure OpenAI (GPT-4o) | Primary model for generation |
| **Fallback** | Claude Sonnet (via LiteLLM) | Fallback during outages |
| **RAG** | pgvector + BM25 | Knowledge retrieval |
| **Reranking** | Cross-encoder | Result reranking |
| **Cache** | Redis | Response caching |
| **Observability** | Langfuse | AI tracing & analytics |
| **Async Jobs** | Hangfire (.NET) + Redis | Long-running tasks |
| **Deployment** | Docker + Kubernetes | Container orchestration |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-06 | Initial LLD for AI Engine Architecture |
