# Low-Level Design (LLD) - Data Layer
## AI-Powered Workflow Creation System

**Tech Stack:** PostgreSQL + pgvector, Redis, Azure Blob Storage, EF Core, SQLAlchemy, asyncpg

**Architecture:** Unified multi-purpose data layer extending existing infrastructure

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [PostgreSQL + pgvector](#postgresql--pgvector)
3. [Database Schema](#database-schema)
4. [EF Core Integration](#ef-core-integration)
5. [Redis Strategy](#redis-strategy)
6. [Azure Blob Storage](#azure-blob-storage)
7. [Data Access Patterns](#data-access-patterns)
8. [Caching Strategies](#caching-strategies)
9. [Performance Optimization](#performance-optimization)
10. [Security & Encryption](#security--encryption)
11. [Backup & Disaster Recovery](#backup--disaster-recovery)
12. [Monitoring & Maintenance](#monitoring--maintenance)
13. [Data Migration](#data-migration)
14. [Compliance & Retention](#compliance--retention)

---

## 1. Architecture Overview

### 1.1 Data Layer Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    Application Layer                             │
├──────────────────────────────────────────────────────────────────┤
│  .NET Backend              │          Python AI Engine            │
│  (EF Core ORM)            │         (SQLAlchemy/asyncpg)         │
└──────────────┬─────────────┼──────────────────┬──────────────────┘
               │             │                  │
        ┌──────▼─────────────▼──────────────────▼─────────┐
        │         Data Access & Caching Layer             │
        ├────────────────────────────────────────────────┤
        │  ┌─────────────────┐  ┌──────────────────────┐ │
        │  │  Repository     │  │  Cache Layer         │ │
        │  │  Pattern        │  │  (Redis)             │ │
        │  └─────────────────┘  └──────────────────────┘ │
        └──────────────────────┬──────────────────────────┘
                               │
        ┌──────────────────────┴──────────────────────┐
        │                                             │
        │                                             │
   ┌────▼──────────────┐  ┌───────────────────┐  ┌──▼───────────────┐
   │   PostgreSQL      │  │  Redis            │  │ Azure Blob       │
   │  + pgvector       │  │  (Cache/Queue)    │  │ Storage          │
   │                  │  │                   │  │ (Artifacts)      │
   │ Relational Data  │  │ Session Memory    │  │                  │
   │ Vector Embeddings │  │ Semantic Cache    │  │ Exports (PDF,    │
   │ AI Trace Logs    │  │ Hangfire Jobs     │  │ JSON, YAML)      │
   └────────────────────┘  └───────────────────┘  └──────────────────┘
```

### 1.2 Why This Architecture?

| Layer | Technology | Benefit |
|-------|-----------|---------|
| **Relational** | PostgreSQL | ACID compliance, complex queries, existing investment |
| **Vector Search** | pgvector | Semantic search for RAG, no separate vector DB |
| **In-Memory Cache** | Redis | Fast session access, conversation memory, Hangfire queue |
| **Blob Storage** | Azure Blob | Immutable artifact storage, scalability, cost-effective |
| **ORM** | EF Core + SQLAlchemy | Type-safe queries, migration management, multi-language |

### 1.3 Data Flow

```
User Creates Workflow
        │
        ▼
.NET API (EF Core)
        │
        ├─────────────────────────────────────┐
        │                                     │
        ▼                                     ▼
PostgreSQL                              Redis Cache
(Persistent)                       (Session + Semantic)
        │                                     │
        ▼                                     ▼
Python AI Engine                      LLM Response
(SQLAlchemy/asyncpg)                  (Cached/Fresh)
        │
        ▼
Generate Workflow
        │
        ├─────────────────────────────────────┐
        │                                     │
        ▼                                     ▼
PostgreSQL                          Azure Blob Storage
(Save Result)                       (Export Artifact)
        │                                     │
        └─────────────────────────────────────┘
                     │
                     ▼
              React Frontend
            (Download/Display)
```

---

## 2. PostgreSQL + pgvector

### 2.1 PostgreSQL Setup

```sql
-- Step 1: Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Step 2: Create schema
CREATE SCHEMA IF NOT EXISTS workflow;

-- Step 3: Set search_path for convenience
ALTER USER current_user SET search_path = workflow, public;
```

### 2.2 pgvector Configuration

```sql
-- Configure pgvector for optimal performance
ALTER SYSTEM SET shared_preload_libraries = 'vector';
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;

-- Restart PostgreSQL for shared_preload_libraries to take effect
-- SELECT pg_reload_conf();

-- Create vector index for fast similarity search
CREATE INDEX ON workflow.industry_knowledge USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Or use HNSW index for better quality (PostgreSQL 14+)
CREATE INDEX ON workflow.industry_knowledge USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

### 2.3 Vector Embedding Configuration

```sql
-- Configure embedding dimensions based on your model
-- text-embedding-3-large: 3072 dimensions
-- text-embedding-3-small: 1536 dimensions
-- text-davinci-002: 1536 dimensions

-- Example: Industry Knowledge with 3072-dim embeddings
CREATE TABLE workflow.industry_knowledge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    industry VARCHAR(100) NOT NULL,
    domain VARCHAR(100),
    content TEXT NOT NULL,
    embedding vector(3072),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    
    -- Constraints
    CONSTRAINT check_content_not_empty CHECK (length(content) > 0),
    
    -- Indexes
    INDEX idx_industry ON industry_knowledge(industry),
    INDEX idx_domain ON industry_knowledge(domain),
    INDEX idx_created_at ON industry_knowledge(created_at DESC)
);

-- GiST index for vector similarity search
CREATE INDEX idx_embedding_cosine ON workflow.industry_knowledge
USING gist (embedding vector_cosine_ops);
```

### 2.4 pgvector Usage Example

```python
# Python - asyncpg with pgvector
import asyncpg
import numpy as np
from typing import List, Tuple

class PgVectorClient:
    def __init__(self, dsn: str):
        self.dsn = dsn
        self.pool = None
    
    async def initialize(self):
        """Initialize connection pool"""
        self.pool = await asyncpg.create_pool(self.dsn)
    
    async def search_similar(
        self,
        query_embedding: List[float],
        industry: str = None,
        top_k: int = 8,
        threshold: float = 0.7
    ) -> List[dict]:
        """Search for similar documents using vector similarity"""
        async with self.pool.acquire() as conn:
            # Convert to PostgreSQL vector format
            vector_str = f"[{','.join(map(str, query_embedding))}]"
            
            query = """
            SELECT 
                id,
                industry,
                domain,
                content,
                metadata,
                (embedding <-> $1::vector) AS distance,
                (1 - (embedding <-> $1::vector)) AS similarity
            FROM workflow.industry_knowledge
            WHERE 1 - (embedding <-> $1::vector) >= $2
            """
            
            params = [vector_str, threshold]
            
            # Add industry filter if specified
            if industry:
                query += " AND industry = $3"
                params.append(industry)
            
            query += " ORDER BY distance ASC LIMIT $" + str(len(params) + 1)
            params.append(top_k)
            
            results = await conn.fetch(query, *params)
            
            return [
                {
                    "id": str(row["id"]),
                    "industry": row["industry"],
                    "domain": row["domain"],
                    "content": row["content"],
                    "metadata": row["metadata"],
                    "similarity": float(row["similarity"])
                }
                for row in results
            ]
    
    async def bulk_insert_embeddings(
        self,
        documents: List[dict]
    ) -> int:
        """Bulk insert documents with embeddings"""
        async with self.pool.acquire() as conn:
            count = 0
            for doc in documents:
                vector_str = f"[{','.join(map(str, doc['embedding']))}]"
                
                await conn.execute("""
                    INSERT INTO workflow.industry_knowledge
                    (industry, domain, content, embedding, metadata)
                    VALUES ($1, $2, $3, $4::vector, $5)
                    ON CONFLICT (id) DO UPDATE
                    SET updated_at = CURRENT_TIMESTAMP
                """,
                    doc["industry"],
                    doc.get("domain"),
                    doc["content"],
                    vector_str,
                    doc.get("metadata", {})
                )
                count += 1
            
            return count
```

---

## 3. Database Schema

### 3.1 Complete Schema with Migrations

```sql
-- ========================================
-- CORE WORKFLOW TABLES
-- ========================================

CREATE TABLE workflow.workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    created_by UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    category VARCHAR(100),
    tags TEXT[] DEFAULT ARRAY[]::TEXT[],
    is_published BOOLEAN DEFAULT FALSE,
    published_at TIMESTAMP,
    published_version INT,
    
    -- Configuration
    trigger_type VARCHAR(100),
    retry_policy JSONB,
    timeout_minutes INT,
    
    -- Metadata
    metadata JSONB DEFAULT '{}',
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    deleted_at TIMESTAMP,
    
    -- Constraints & Indexes
    CONSTRAINT fk_created_by FOREIGN KEY (created_by) REFERENCES users(id),
    CONSTRAINT check_name_not_empty CHECK (length(name) > 0),
    INDEX idx_tenant_id ON workflows(tenant_id),
    INDEX idx_status ON workflows(status),
    INDEX idx_created_by ON workflows(created_by),
    INDEX idx_published ON workflows(is_published),
    INDEX idx_created_at ON workflows(created_at DESC)
);

-- Workflow Versions (for versioning and rollback)
CREATE TABLE workflow.workflow_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL,
    version INT NOT NULL,
    definition_json JSONB NOT NULL,
    nodes JSONB NOT NULL,
    edges JSONB NOT NULL,
    description TEXT,
    created_by UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT fk_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE CASCADE,
    CONSTRAINT fk_created_by FOREIGN KEY (created_by) REFERENCES users(id),
    CONSTRAINT unique_version UNIQUE(workflow_id, version),
    INDEX idx_workflow_id ON workflow_versions(workflow_id),
    INDEX idx_created_at ON workflow_versions(created_at DESC)
);

-- Workflow Nodes
CREATE TABLE workflow.workflow_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL,
    version INT NOT NULL DEFAULT 1,
    node_type VARCHAR(100) NOT NULL,
    label VARCHAR(255) NOT NULL,
    description TEXT,
    sequence_order INT,
    is_start_node BOOLEAN DEFAULT FALSE,
    is_end_node BOOLEAN DEFAULT FALSE,
    
    -- Configuration
    configuration JSONB,
    input_schema JSONB,
    output_schema JSONB,
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT fk_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE CASCADE,
    CONSTRAINT check_label_not_empty CHECK (length(label) > 0),
    INDEX idx_workflow_id ON workflow_nodes(workflow_id),
    INDEX idx_node_type ON workflow_nodes(node_type)
);

-- Workflow Edges
CREATE TABLE workflow.workflow_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL,
    version INT NOT NULL DEFAULT 1,
    source_node_id UUID NOT NULL,
    target_node_id UUID NOT NULL,
    edge_type VARCHAR(100) DEFAULT 'DEFAULT',
    condition JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT fk_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE CASCADE,
    CONSTRAINT fk_source_node FOREIGN KEY (source_node_id) REFERENCES workflow_nodes(id),
    CONSTRAINT fk_target_node FOREIGN KEY (target_node_id) REFERENCES workflow_nodes(id),
    INDEX idx_workflow_id ON workflow_edges(workflow_id),
    INDEX idx_source_node ON workflow_edges(source_node_id),
    INDEX idx_target_node ON workflow_edges(target_node_id)
);

-- ========================================
-- EXECUTION & TRACKING TABLES
-- ========================================

CREATE TABLE workflow.workflow_executions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL,
    initiated_by UUID,
    session_id UUID,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    duration_ms INT,
    
    -- Input/Output
    input_data JSONB,
    output_data JSONB,
    
    -- Error Handling
    error_message TEXT,
    error_code VARCHAR(100),
    retry_attempt INT DEFAULT 0,
    
    -- Metadata
    execution_context JSONB,
    metadata JSONB DEFAULT '{}',
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints & Indexes
    CONSTRAINT fk_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id),
    CONSTRAINT fk_initiated_by FOREIGN KEY (initiated_by) REFERENCES users(id),
    CONSTRAINT check_status CHECK (status IN ('PENDING', 'RUNNING', 'SUCCESS', 'FAILED', 'CANCELLED')),
    INDEX idx_workflow_id ON workflow_executions(workflow_id),
    INDEX idx_status ON workflow_executions(status),
    INDEX idx_created_at ON workflow_executions(created_at DESC),
    INDEX idx_session_id ON workflow_executions(session_id)
);

-- Execution Steps
CREATE TABLE workflow.execution_steps (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id UUID NOT NULL,
    node_id UUID NOT NULL,
    step_number INT NOT NULL,
    status VARCHAR(50) NOT NULL,
    input_data JSONB,
    output_data JSONB,
    error_message TEXT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    duration_ms INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT fk_execution FOREIGN KEY (execution_id) REFERENCES workflow_executions(id) ON DELETE CASCADE,
    CONSTRAINT fk_node FOREIGN KEY (node_id) REFERENCES workflow_nodes(id),
    INDEX idx_execution_id ON execution_steps(execution_id),
    INDEX idx_step_number ON execution_steps(execution_id, step_number)
);

-- Execution Logs
CREATE TABLE workflow.execution_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    execution_id UUID NOT NULL,
    step_id UUID,
    log_level VARCHAR(20) NOT NULL,
    message TEXT NOT NULL,
    details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints
    CONSTRAINT fk_execution FOREIGN KEY (execution_id) REFERENCES workflow_executions(id) ON DELETE CASCADE,
    CONSTRAINT fk_step FOREIGN KEY (step_id) REFERENCES execution_steps(id) ON DELETE SET NULL,
    INDEX idx_execution_id ON execution_logs(execution_id),
    INDEX idx_created_at ON execution_logs(created_at DESC)
);

-- ========================================
-- AI TRACKING & SESSIONS
-- ========================================

CREATE TABLE workflow.workflow_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    workflow_id UUID,
    industry VARCHAR(100),
    status VARCHAR(50) DEFAULT 'ACTIVE',
    conversation_history JSONB DEFAULT '[]'::JSONB,
    current_workflow JSONB,
    metadata JSONB DEFAULT '{}',
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL '24 hours'),
    
    -- Constraints & Indexes
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id),
    CONSTRAINT fk_workflow FOREIGN KEY (workflow_id) REFERENCES workflows(id) ON DELETE SET NULL,
    INDEX idx_user_id ON workflow_sessions(user_id),
    INDEX idx_status ON workflow_sessions(status),
    INDEX idx_expires_at ON workflow_sessions(expires_at)
);

-- AI Trace Logs (for observability)
CREATE TABLE workflow.ai_trace_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL,
    execution_id UUID,
    model VARCHAR(100) NOT NULL,
    prompt_tokens INT,
    completion_tokens INT,
    total_tokens INT,
    latency_ms INT,
    cost_usd DECIMAL(10, 6),
    quality_score FLOAT,
    input_text TEXT,
    output_text TEXT,
    
    -- Metadata
    metadata JSONB DEFAULT '{}',
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints & Indexes
    CONSTRAINT fk_session FOREIGN KEY (session_id) REFERENCES workflow_sessions(id) ON DELETE CASCADE,
    CONSTRAINT fk_execution FOREIGN KEY (execution_id) REFERENCES workflow_executions(id) ON DELETE SET NULL,
    INDEX idx_session_id ON ai_trace_logs(session_id),
    INDEX idx_model ON ai_trace_logs(model),
    INDEX idx_created_at ON ai_trace_logs(created_at DESC)
);

-- ========================================
-- KNOWLEDGE BASE
-- ========================================

CREATE TABLE workflow.industry_knowledge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    industry VARCHAR(100) NOT NULL,
    domain VARCHAR(100),
    content_type VARCHAR(50),
    content TEXT NOT NULL,
    embedding vector(3072),
    metadata JSONB DEFAULT '{}',
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Indexes
    INDEX idx_industry ON industry_knowledge(industry),
    INDEX idx_domain ON industry_knowledge(domain),
    INDEX idx_embedding ON industry_knowledge USING hnsw(embedding vector_cosine_ops)
);

-- ========================================
-- AUDIT LOGGING
-- ========================================

CREATE TABLE workflow.audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID,
    action VARCHAR(100) NOT NULL,
    old_values JSONB,
    new_values JSONB,
    ip_address VARCHAR(50),
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Constraints & Indexes
    CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_user_id ON audit_logs(user_id),
    INDEX idx_entity ON audit_logs(entity_type, entity_id),
    INDEX idx_created_at ON audit_logs(created_at DESC)
);
```

### 3.2 Views for Common Queries

```sql
-- View: Active workflows with execution stats
CREATE VIEW workflow.v_active_workflows_with_stats AS
SELECT 
    w.id,
    w.name,
    w.status,
    COUNT(DISTINCT we.id) as execution_count,
    MAX(we.completed_at) as last_executed,
    AVG(we.duration_ms) as avg_duration_ms,
    SUM(CASE WHEN we.status = 'SUCCESS' THEN 1 ELSE 0 END)::FLOAT / 
        NULLIF(COUNT(we.id), 0) as success_rate
FROM workflow.workflows w
LEFT JOIN workflow.workflow_executions we ON w.id = we.workflow_id
WHERE w.deleted_at IS NULL AND w.is_published = TRUE
GROUP BY w.id, w.name, w.status;

-- View: User activity summary
CREATE VIEW workflow.v_user_activity_summary AS
SELECT 
    u.id,
    u.email,
    COUNT(DISTINCT w.id) as workflow_count,
    COUNT(DISTINCT we.id) as execution_count,
    MAX(we.created_at) as last_activity,
    COUNT(DISTINCT CASE WHEN we.status = 'SUCCESS' THEN we.id END) as successful_executions
FROM users u
LEFT JOIN workflow.workflows w ON u.id = w.created_by AND w.deleted_at IS NULL
LEFT JOIN workflow.workflow_executions we ON w.id = we.workflow_id
GROUP BY u.id, u.email;

-- View: AI model performance
CREATE VIEW workflow.v_ai_performance_stats AS
SELECT 
    model,
    COUNT(*) as call_count,
    AVG(latency_ms) as avg_latency_ms,
    MAX(latency_ms) as max_latency_ms,
    AVG(total_tokens) as avg_tokens,
    AVG(cost_usd) as avg_cost,
    AVG(quality_score) as avg_quality,
    SUM(cost_usd) as total_cost
FROM workflow.ai_trace_logs
WHERE created_at > CURRENT_TIMESTAMP - INTERVAL '30 days'
GROUP BY model;
```

---

## 4. EF Core Integration

### 4.1 Entity Models

```csharp
// Models/Entities/Workflow.cs
using System;
using System.Collections.Generic;

namespace WorkflowCreation.Models.Entities
{
    public class Workflow
    {
        public Guid Id { get; set; }
        public Guid TenantId { get; set; }
        public Guid CreatedBy { get; set; }
        
        public string Name { get; set; }
        public string Description { get; set; }
        public WorkflowStatus Status { get; set; }
        public string Category { get; set; }
        public List<string> Tags { get; set; }
        
        public bool IsPublished { get; set; }
        public DateTime? PublishedAt { get; set; }
        public int? PublishedVersion { get; set; }
        
        // Configuration
        public string TriggerType { get; set; }
        public JsonDocument RetryPolicy { get; set; }
        public int? TimeoutMinutes { get; set; }
        
        // Metadata
        public JsonDocument Metadata { get; set; }
        
        // Timestamps
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
        public DateTime? DeletedAt { get; set; }
        
        // Navigation Properties
        public User Creator { get; set; }
        public List<WorkflowVersion> Versions { get; set; }
        public List<WorkflowNode> Nodes { get; set; }
        public List<WorkflowEdge> Edges { get; set; }
        public List<WorkflowExecution> Executions { get; set; }
    }

    public enum WorkflowStatus
    {
        Draft,
        Active,
        Inactive,
        Archived
    }
}

// Models/Entities/WorkflowVersion.cs
public class WorkflowVersion
{
    public Guid Id { get; set; }
    public Guid WorkflowId { get; set; }
    public int Version { get; set; }
    
    public JsonDocument DefinitionJson { get; set; }
    public JsonDocument Nodes { get; set; }
    public JsonDocument Edges { get; set; }
    public string Description { get; set; }
    
    public Guid CreatedBy { get; set; }
    public DateTime CreatedAt { get; set; }
    
    // Navigation Properties
    public Workflow Workflow { get; set; }
    public User Creator { get; set; }
}

// Models/Entities/WorkflowNode.cs
public class WorkflowNode
{
    public Guid Id { get; set; }
    public Guid WorkflowId { get; set; }
    public int Version { get; set; }
    
    public string NodeType { get; set; }
    public string Label { get; set; }
    public string Description { get; set; }
    public int? SequenceOrder { get; set; }
    public bool IsStartNode { get; set; }
    public bool IsEndNode { get; set; }
    
    // Configuration
    public JsonDocument Configuration { get; set; }
    public JsonDocument InputSchema { get; set; }
    public JsonDocument OutputSchema { get; set; }
    
    public DateTime CreatedAt { get; set; }
    public DateTime UpdatedAt { get; set; }
    
    // Navigation Properties
    public Workflow Workflow { get; set; }
    public List<WorkflowEdge> OutgoingEdges { get; set; }
    public List<WorkflowEdge> IncomingEdges { get; set; }
}

// Models/Entities/WorkflowExecution.cs
public class WorkflowExecution
{
    public Guid Id { get; set; }
    public Guid WorkflowId { get; set; }
    public Guid? InitiatedBy { get; set; }
    public Guid? SessionId { get; set; }
    
    public ExecutionStatus Status { get; set; }
    public DateTime? StartedAt { get; set; }
    public DateTime? CompletedAt { get; set; }
    public int? DurationMs { get; set; }
    
    // Input/Output
    public JsonDocument InputData { get; set; }
    public JsonDocument OutputData { get; set; }
    
    // Error Handling
    public string ErrorMessage { get; set; }
    public string ErrorCode { get; set; }
    public int RetryAttempt { get; set; }
    
    public JsonDocument ExecutionContext { get; set; }
    public JsonDocument Metadata { get; set; }
    
    public DateTime CreatedAt { get; set; }
    
    // Navigation Properties
    public Workflow Workflow { get; set; }
    public User InitiatedByUser { get; set; }
    public List<ExecutionStep> Steps { get; set; }
    public List<ExecutionLog> Logs { get; set; }
}

public enum ExecutionStatus
{
    Pending,
    Running,
    Success,
    Failed,
    Cancelled
}
```

### 4.2 DbContext Configuration

```csharp
// Data/WorkflowDbContext.cs
using Microsoft.EntityFrameworkCore;
using WorkflowCreation.Models.Entities;

namespace WorkflowCreation.Data
{
    public class WorkflowDbContext : DbContext
    {
        public WorkflowDbContext(DbContextOptions<WorkflowDbContext> options)
            : base(options)
        {
        }

        // DbSets
        public DbSet<Workflow> Workflows { get; set; }
        public DbSet<WorkflowVersion> WorkflowVersions { get; set; }
        public DbSet<WorkflowNode> WorkflowNodes { get; set; }
        public DbSet<WorkflowEdge> WorkflowEdges { get; set; }
        public DbSet<WorkflowExecution> WorkflowExecutions { get; set; }
        public DbSet<ExecutionStep> ExecutionSteps { get; set; }
        public DbSet<ExecutionLog> ExecutionLogs { get; set; }
        public DbSet<WorkflowSession> WorkflowSessions { get; set; }
        public DbSet<AITraceLog> AITraceLogs { get; set; }
        public DbSet<IndustryKnowledge> IndustryKnowledge { get; set; }
        public DbSet<AuditLog> AuditLogs { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Configure Workflow entity
            modelBuilder.Entity<Workflow>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Id).HasDefaultValueSql("gen_random_uuid()");
                
                entity.Property(e => e.Name).IsRequired().HasMaxLength(255);
                entity.Property(e => e.Status).HasConversion<string>();
                entity.Property(e => e.Tags).HasConversion(
                    v => string.Join(',', v),
                    v => v.Split(',', System.StringSplitOptions.RemoveEmptyEntries).ToList()
                );

                entity.HasKey(e => e.Id);
                entity.HasIndex(e => e.TenantId);
                entity.HasIndex(e => e.CreatedBy);
                entity.HasIndex(e => e.Status);
                entity.HasIndex(e => e.IsPublished);

                // Soft delete
                entity.HasQueryFilter(e => e.DeletedAt == null);

                // Foreign keys
                entity.HasOne(e => e.Creator)
                    .WithMany()
                    .HasForeignKey(e => e.CreatedBy)
                    .OnDelete(DeleteBehavior.Restrict);

                // Relations
                entity.HasMany(e => e.Versions)
                    .WithOne(v => v.Workflow)
                    .HasForeignKey(v => v.WorkflowId)
                    .OnDelete(DeleteBehavior.Cascade);

                entity.HasMany(e => e.Nodes)
                    .WithOne(n => n.Workflow)
                    .HasForeignKey(n => n.WorkflowId)
                    .OnDelete(DeleteBehavior.Cascade);

                entity.HasMany(e => e.Executions)
                    .WithOne(ex => ex.Workflow)
                    .HasForeignKey(ex => ex.WorkflowId)
                    .OnDelete(DeleteBehavior.Cascade);
            });

            // Configure WorkflowNode entity
            modelBuilder.Entity<WorkflowNode>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Id).HasDefaultValueSql("gen_random_uuid()");

                entity.HasIndex(e => e.WorkflowId);
                entity.HasIndex(e => e.NodeType);

                entity.HasOne(e => e.Workflow)
                    .WithMany(w => w.Nodes)
                    .HasForeignKey(e => e.WorkflowId)
                    .OnDelete(DeleteBehavior.Cascade);

                entity.HasMany(e => e.OutgoingEdges)
                    .WithOne(edge => edge.SourceNode)
                    .HasForeignKey(edge => edge.SourceNodeId)
                    .OnDelete(DeleteBehavior.Restrict);

                entity.HasMany(e => e.IncomingEdges)
                    .WithOne(edge => edge.TargetNode)
                    .HasForeignKey(edge => edge.TargetNodeId)
                    .OnDelete(DeleteBehavior.Restrict);
            });

            // Configure WorkflowExecution entity
            modelBuilder.Entity<WorkflowExecution>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Id).HasDefaultValueSql("gen_random_uuid()");

                entity.Property(e => e.Status).HasConversion<string>();

                entity.HasIndex(e => e.WorkflowId);
                entity.HasIndex(e => e.Status);
                entity.HasIndex(e => e.CreatedAt).IsDescending();

                entity.HasMany(e => e.Steps)
                    .WithOne(s => s.Execution)
                    .HasForeignKey(s => s.ExecutionId)
                    .OnDelete(DeleteBehavior.Cascade);

                entity.HasMany(e => e.Logs)
                    .WithOne(l => l.Execution)
                    .HasForeignKey(l => l.ExecutionId)
                    .OnDelete(DeleteBehavior.Cascade);
            });

            // Configure AITraceLog entity
            modelBuilder.Entity<AITraceLog>(entity =>
            {
                entity.HasKey(e => e.Id);
                entity.Property(e => e.Id).HasDefaultValueSql("gen_random_uuid()");

                entity.HasIndex(e => e.SessionId);
                entity.HasIndex(e => e.Model);
                entity.HasIndex(e => e.CreatedAt).IsDescending();
            });

            // Configure seed data
            SeedData(modelBuilder);
        }

        private void SeedData(ModelBuilder modelBuilder)
        {
            // Add default templates, integrations, etc.
        }
    }
}
```

### 4.3 EF Core Migrations

```bash
# Create initial migration
dotnet ef migrations add InitialCreate --output-dir Data/Migrations

# Add pgvector support
dotnet ef migrations add AddPgVectorSupport

# Apply migrations
dotnet ef database update

# Create migration for specific DbContext
dotnet ef migrations add AddWorkflowVersioning -c WorkflowDbContext
```

### 4.4 Migration File Example

```csharp
// Data/Migrations/20240606120000_InitialCreate.cs
using Microsoft.EntityFrameworkCore.Migrations;
using Npgsql.EntityFrameworkCore.PostgreSQL.Metadata;
using System;

namespace WorkflowCreation.Data.Migrations
{
    public partial class InitialCreate : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            // Enable pgvector extension
            migrationBuilder.Sql("CREATE EXTENSION IF NOT EXISTS vector");

            // Create tables
            migrationBuilder.CreateTable(
                name: "workflow",
                columns: table => new
                {
                    id = table.Column<Guid>(type: "uuid", nullable: false, defaultValueSql: "gen_random_uuid()"),
                    tenant_id = table.Column<Guid>(type: "uuid", nullable: false),
                    created_by = table.Column<Guid>(type: "uuid", nullable: false),
                    name = table.Column<string>(type: "character varying(255)", nullable: false),
                    status = table.Column<string>(type: "character varying(50)", nullable: false),
                    created_at = table.Column<DateTime>(type: "timestamp without time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP"),
                    updated_at = table.Column<DateTime>(type: "timestamp without time zone", nullable: false, defaultValueSql: "CURRENT_TIMESTAMP")
                },
                constraints: table =>
                {
                    table.PrimaryKey("pk_workflow", x => x.id);
                });

            // Create indexes
            migrationBuilder.CreateIndex(
                name: "idx_tenant_id",
                table: "workflow",
                column: "tenant_id");
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(
                name: "workflow");

            migrationBuilder.Sql("DROP EXTENSION IF EXISTS vector");
        }
    }
}
```

---

## 5. Redis Strategy

### 5.1 Redis Connection Setup

```csharp
// .NET Configuration
services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = configuration.GetConnectionString("Redis");
    options.InstanceName = "workflow_";
});

public class RedisConnectionFactory
{
    private readonly IConnectionMultiplexer _multiplexer;

    public RedisConnectionFactory(string connectionString)
    {
        _multiplexer = ConnectionMultiplexer.Connect(connectionString);
    }

    public IDatabase GetDatabase(int dbIndex = 0) => _multiplexer.GetDatabase(dbIndex);
    public IServer GetServer() => _multiplexer.GetServer(_multiplexer.GetEndPoints().First());
    public ISubscriber GetSubscriber() => _multiplexer.GetSubscriber();
}
```

```python
# Python Configuration
import aioredis
from redis import Redis
from redis.asyncio import Redis as AsyncRedis

async def get_redis_client():
    """Get async Redis client"""
    redis = await AsyncRedis.from_url(
        os.getenv("REDIS_URL", "redis://localhost:6379"),
        encoding="utf-8",
        decode_responses=True
    )
    return redis

# Sync client for Hangfire
redis_client = Redis.from_url(os.getenv("REDIS_URL"))
```

### 5.2 Redis Key Patterns & Usage

```python
# Session Management - 24 hour TTL
session:{sessionId}:messages → JSON array of conversation messages
session:{sessionId}:workflow → Current workflow JSON
session:{sessionId}:context → User context and preferences

# Semantic Cache - 1 hour TTL
semcache:{promptHash} → Cached workflow generation result
semcache:{promptHash}:timestamp → Cache creation time
semcache:{promptHash}:hits → Cache hit counter

# Job Queue - Hangfire managed
hangfire:queues:{queue_name}
hangfire:job:{job_id}
hangfire:recurring-jobs

# Rate Limiting
ratelimit:{userId}:requests → Request count
ratelimit:{userId}:reset_at → Reset timestamp

# Example Usage
```

```csharp
// .NET - Session Storage
public class SessionService
{
    private readonly IDistributedCache _cache;
    
    public async Task SaveSessionAsync(string sessionId, WorkflowSession session)
    {
        var json = JsonSerializer.Serialize(session);
        await _cache.SetStringAsync(
            $"session:{sessionId}:workflow",
            json,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24)
            }
        );
    }
    
    public async Task<WorkflowSession> GetSessionAsync(string sessionId)
    {
        var json = await _cache.GetStringAsync($"session:{sessionId}:workflow");
        return json != null ? JsonSerializer.Deserialize<WorkflowSession>(json) : null;
    }
}

// .NET - Semantic Cache
public class SemanticCacheService
{
    private readonly IDistributedCache _cache;
    
    public async Task<string> GetOrGenerateAsync(
        string prompt,
        Func<Task<string>> generator)
    {
        var hash = ComputeHash(prompt);
        var cacheKey = $"semcache:{hash}";
        
        // Try cache first
        var cached = await _cache.GetStringAsync(cacheKey);
        if (cached != null)
        {
            await _cache.IncrementAsync($"{cacheKey}:hits");
            return cached;
        }
        
        // Generate new result
        var result = await generator();
        
        // Cache for 1 hour
        await _cache.SetStringAsync(
            cacheKey,
            result,
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            }
        );
        
        return result;
    }
}
```

```python
# Python - Session Storage
class SessionService:
    def __init__(self, redis: AsyncRedis):
        self.redis = redis
    
    async def save_session(self, session_id: str, session_data: dict):
        """Save session with 24-hour TTL"""
        await self.redis.setex(
            f"session:{session_id}:workflow",
            86400,  # 24 hours in seconds
            json.dumps(session_data)
        )
    
    async def get_session(self, session_id: str) -> dict:
        """Retrieve session data"""
        data = await self.redis.get(f"session:{session_id}:workflow")
        return json.loads(data) if data else None

# Python - Semantic Cache
class SemanticCacheService:
    def __init__(self, redis: AsyncRedis):
        self.redis = redis
    
    async def get_or_generate(
        self,
        prompt: str,
        generator_fn
    ) -> str:
        """Get cached workflow or generate new"""
        hash_key = hashlib.sha256(prompt.encode()).hexdigest()
        cache_key = f"semcache:{hash_key}"
        
        # Try cache first
        cached = await self.redis.get(cache_key)
        if cached:
            # Increment hit counter
            await self.redis.incr(f"{cache_key}:hits")
            return json.loads(cached)
        
        # Generate new result
        result = await generator_fn(prompt)
        
        # Cache for 1 hour
        await self.redis.setex(
            cache_key,
            3600,  # 1 hour
            json.dumps(result)
        )
        
        return result
```

### 5.3 Hangfire Configuration

```csharp
// .NET - Hangfire Setup
services.AddHangfire(configuration =>
{
    // Use Redis as job storage
    configuration
        .UseSimpleAssemblyNameTypeSerializer()
        .UseRecommendedSerializerSettings()
        .UseRedisStorage(
            configuration.GetConnectionString("Redis"),
            new RedisStorageOptions
            {
                Db = 1, // Use separate DB for Hangfire
                ExpirationTimeInMonths = 1
            }
        );
});

services.AddHangfireServer(options =>
{
    options.WorkerCount = Environment.ProcessorCount * 2;
    options.Queues = new[] { "default", "ai_generation", "exports" };
});

// Configure in Startup
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    Authorization = new[] { new HangfireAuthorizationFilter() }
});

// Enqueue background job
var jobId = BackgroundJob.Enqueue<WorkflowAIService>(
    service => service.GenerateAsync(request, CancellationToken.None)
);

// Schedule recurring job
RecurringJob.AddOrUpdate(
    "cleanup-expired-sessions",
    () => _sessionService.CleanupExpiredAsync(),
    Cron.Daily(2, 0) // Run at 2 AM daily
);
```

---

## 6. Azure Blob Storage

### 6.1 Blob Storage Setup

```csharp
// Configuration
services.AddSingleton(x =>
{
    var connectionString = configuration.GetConnectionString("AzureStorage");
    return new BlobServiceClient(connectionString);
});

public class BlobStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private const string ContainerName = "workflow-exports";
    
    public BlobStorageService(BlobServiceClient blobServiceClient)
    {
        _blobServiceClient = blobServiceClient;
    }
    
    public async Task<string> UploadWorkflowExportAsync(
        string tenantId,
        string workflowId,
        int version,
        string format,
        byte[] content,
        string contentType)
    {
        // Get container reference
        var container = _blobServiceClient.GetBlobContainerClient(ContainerName);
        await container.CreateIfNotExistsAsync();
        
        // Generate blob name
        var blobName = $"{tenantId}/{workflowId}/v{version}.{format}";
        var blobClient = container.GetBlobClient(blobName);
        
        // Upload blob
        using var stream = new MemoryStream(content);
        await blobClient.UploadAsync(stream, overwrite: true);
        
        // Set metadata
        var metadata = new Dictionary<string, string>
        {
            { "workflowId", workflowId },
            { "version", version.ToString() },
            { "format", format },
            { "uploadedAt", DateTime.UtcNow.ToString("O") }
        };
        
        await blobClient.SetMetadataAsync(metadata);
        
        // Generate SAS URI (valid for 1 hour)
        var sasUri = blobClient.GenerateSasUri(
            BlobSasPermissions.Read,
            DateTimeOffset.UtcNow.AddHours(1)
        );
        
        return sasUri.ToString();
    }
    
    public async Task DeleteWorkflowExportAsync(string tenantId, string workflowId, int version)
    {
        var container = _blobServiceClient.GetBlobContainerClient(ContainerName);
        var blobName = $"{tenantId}/{workflowId}/v{version}.*";
        
        // Delete all versions of this workflow
        await foreach (var blob in container.GetBlobsAsync(prefix: $"{tenantId}/{workflowId}"))
        {
            await container.DeleteBlobAsync(blob.Name);
        }
    }
    
    public async Task<BlobDownloadInfo> DownloadWorkflowExportAsync(
        string tenantId,
        string workflowId,
        int version,
        string format)
    {
        var container = _blobServiceClient.GetBlobContainerClient(ContainerName);
        var blobName = $"{tenantId}/{workflowId}/v{version}.{format}";
        var blobClient = container.GetBlobClient(blobName);
        
        return await blobClient.DownloadAsync();
    }
    
    public async Task<string> GetSasUrlAsync(
        string tenantId,
        string workflowId,
        int version,
        string format,
        int expiryHours = 1)
    {
        var container = _blobServiceClient.GetBlobContainerClient(ContainerName);
        var blobName = $"{tenantId}/{workflowId}/v{version}.{format}";
        var blobClient = container.GetBlobClient(blobName);
        
        var sasUri = blobClient.GenerateSasUri(
            BlobSasPermissions.Read,
            DateTimeOffset.UtcNow.AddHours(expiryHours)
        );
        
        return sasUri.ToString();
    }
}
```

### 6.2 Blob Storage Usage in Workflow Export

```csharp
// WorkflowExportService.cs
public class WorkflowExportService
{
    private readonly BlobStorageService _blobStorage;
    private readonly IWorkflowRepository _workflowRepository;
    
    public async Task<ExportResult> ExportWorkflowAsync(
        string workflowId,
        ExportFormat format,
        string userId,
        string tenantId)
    {
        // Get workflow
        var workflow = await _workflowRepository.GetWithDetailsAsync(workflowId);
        if (workflow == null)
            throw new NotFoundException("Workflow not found");
        
        // Generate export based on format
        byte[] content = format switch
        {
            ExportFormat.Json => ExportAsJson(workflow),
            ExportFormat.Yaml => ExportAsYaml(workflow),
            ExportFormat.Pdf => ExportAsPdf(workflow),
            _ => throw new ArgumentException("Unsupported format")
        };
        
        // Upload to Blob Storage
        var sasUrl = await _blobStorage.UploadWorkflowExportAsync(
            tenantId: tenantId,
            workflowId: workflowId,
            version: workflow.Version,
            format: format.ToString().ToLower(),
            content: content,
            contentType: GetContentType(format)
        );
        
        // Log audit event
        await _auditService.LogAsync(new AuditLog
        {
            UserId = userId,
            Action = "EXPORT",
            EntityType = "Workflow",
            EntityId = workflowId,
            NewValues = new { format, exportedAt = DateTime.UtcNow }
        });
        
        return new ExportResult
        {
            DownloadUrl = sasUrl,
            ExpiresAt = DateTime.UtcNow.AddHours(1),
            Format = format
        };
    }
    
    private byte[] ExportAsJson(Workflow workflow)
    {
        var dto = new WorkflowExportDto
        {
            Id = workflow.Id,
            Name = workflow.Name,
            Description = workflow.Description,
            Nodes = workflow.Nodes,
            Edges = workflow.Edges,
            ExportedAt = DateTime.UtcNow
        };
        
        var json = JsonSerializer.Serialize(dto, new JsonSerializerOptions { WriteIndented = true });
        return Encoding.UTF8.GetBytes(json);
    }
    
    private byte[] ExportAsYaml(Workflow workflow)
    {
        // Convert to YAML format
        // Implementation using YamlDotNet library
        var yaml = GenerateYaml(workflow);
        return Encoding.UTF8.GetBytes(yaml);
    }
    
    private byte[] ExportAsPdf(Workflow workflow)
    {
        // Generate PDF with workflow diagram
        // Implementation using QuestPDF or iTextSharp
        return GeneratePdf(workflow);
    }
    
    private string GetContentType(ExportFormat format) => format switch
    {
        ExportFormat.Json => "application/json",
        ExportFormat.Yaml => "application/x-yaml",
        ExportFormat.Pdf => "application/pdf",
        _ => "application/octet-stream"
    };
}

public enum ExportFormat
{
    Json,
    Yaml,
    Pdf
}
```

### 6.3 Blob Storage Lifecycle Management

```csharp
// Lifecycle policy for automatic cleanup
public class BlobLifecycleService
{
    private readonly BlobServiceClient _blobServiceClient;
    
    public async Task ConfigureLifecyclePolicyAsync()
    {
        var properties = await _blobServiceClient.GetServicePropertiesAsync();
        
        var policy = new BlobRetentionPolicy
        {
            Days = 90,
            Enabled = true
        };
        
        var rules = new List<BlobManagementRule>
        {
            new BlobManagementRule
            {
                Name = "delete-old-exports",
                Enabled = true,
                Type = "Lifecycle",
                Definition = new BlobManagementRuleDefinition
                {
                    Actions = new BlobManagementRuleAction
                    {
                        Delete = new DateAfterModification { DaysAfterModificationGreaterThan = 90 }
                    },
                    Filters = new BlobManagementRuleFilter
                    {
                        PrefixMatch = new[] { "workflow-exports/" }
                    }
                }
            }
        };
        
        // Configure in Azure Portal or via Azure CLI
        // az storage account management-policy create --account-name <name> --resource-group <rg> --policy @policy.json
    }
}
```

---

## 7. Data Access Patterns

### 7.1 Repository Pattern

```csharp
// Repositories/IWorkflowRepository.cs
public interface IWorkflowRepository
{
    Task<Workflow> GetByIdAsync(Guid id);
    Task<PagedResult<Workflow>> GetAllAsync(WorkflowFilter filter, int pageNumber, int pageSize);
    Task<Workflow> CreateAsync(Workflow workflow);
    Task<Workflow> UpdateAsync(Workflow workflow);
    Task DeleteAsync(Guid id);
    Task<List<Workflow>> GetByTenantAsync(Guid tenantId);
    Task<List<Workflow>> GetPublishedAsync();
}

// Repositories/WorkflowRepository.cs
public class WorkflowRepository : IWorkflowRepository
{
    private readonly WorkflowDbContext _context;
    
    public async Task<Workflow> GetByIdAsync(Guid id)
    {
        return await _context.Workflows
            .AsNoTracking()
            .Include(w => w.Nodes)
            .Include(w => w.Edges)
            .Include(w => w.Executions)
            .FirstOrDefaultAsync(w => w.Id == id);
    }
    
    public async Task<PagedResult<Workflow>> GetAllAsync(
        WorkflowFilter filter,
        int pageNumber,
        int pageSize)
    {
        var query = _context.Workflows.AsNoTracking();
        
        // Apply filters
        if (!string.IsNullOrEmpty(filter.Name))
            query = query.Where(w => w.Name.Contains(filter.Name));
        
        if (!string.IsNullOrEmpty(filter.Status))
            query = query.Where(w => w.Status.ToString() == filter.Status);
        
        if (filter.TenantId.HasValue)
            query = query.Where(w => w.TenantId == filter.TenantId);
        
        // Count total
        var totalCount = await query.CountAsync();
        
        // Get paginated results
        var items = await query
            .OrderByDescending(w => w.CreatedAt)
            .Skip((pageNumber - 1) * pageSize)
            .Take(pageSize)
            .ToListAsync();
        
        return new PagedResult<Workflow>
        {
            Items = items,
            TotalCount = totalCount,
            PageNumber = pageNumber,
            PageSize = pageSize
        };
    }
    
    public async Task<Workflow> CreateAsync(Workflow workflow)
    {
        _context.Workflows.Add(workflow);
        await _context.SaveChangesAsync();
        return workflow;
    }
    
    public async Task<Workflow> UpdateAsync(Workflow workflow)
    {
        workflow.UpdatedAt = DateTime.UtcNow;
        _context.Workflows.Update(workflow);
        await _context.SaveChangesAsync();
        return workflow;
    }
}
```

### 7.2 Query Optimization Examples

```csharp
// Optimize N+1 queries with projection
public async Task<List<WorkflowSummaryDto>> GetWorkflowSummariesAsync(Guid tenantId)
{
    return await _context.Workflows
        .AsNoTracking()
        .Where(w => w.TenantId == tenantId && w.DeletedAt == null)
        .Select(w => new WorkflowSummaryDto
        {
            Id = w.Id,
            Name = w.Name,
            Status = w.Status.ToString(),
            ExecutionCount = w.Executions.Count,
            LastExecuted = w.Executions.Max(e => e.CompletedAt),
            CreatedAt = w.CreatedAt
        })
        .OrderByDescending(w => w.CreatedAt)
        .ToListAsync();
}

// Batch loading for related entities
public async Task<Dictionary<Guid, List<WorkflowNode>>> GetNodesForWorkflowsAsync(List<Guid> workflowIds)
{
    var nodes = await _context.WorkflowNodes
        .AsNoTracking()
        .Where(n => workflowIds.Contains(n.WorkflowId))
        .ToListAsync();
    
    return nodes.GroupBy(n => n.WorkflowId)
        .ToDictionary(g => g.Key, g => g.ToList());
}

// Use SQL-specific features for better performance
public async Task<List<WorkflowExecutionStats>> GetExecutionStatsAsync(Guid workflowId)
{
    return await _context.WorkflowExecutions
        .FromSqlInterpolated($@"
            SELECT 
                DATE(we.created_at) as date,
                COUNT(*) as total_executions,
                SUM(CASE WHEN we.status = 'SUCCESS' THEN 1 ELSE 0 END) as successful,
                AVG(we.duration_ms) as avg_duration_ms
            FROM workflow.workflow_executions we
            WHERE we.workflow_id = {workflowId}
            GROUP BY DATE(we.created_at)
            ORDER BY date DESC
        ")
        .ToListAsync();
}
```

---

## 8. Caching Strategies

### 8.1 Multi-Level Caching

```
┌─────────────────────────────────────┐
│   Application Memory Cache          │
│   (L1 - Shortest lived)             │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   Redis Cache                       │
│   (L2 - Session/Semantic)           │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│   PostgreSQL                        │
│   (L3 - Persistent)                 │
└─────────────────────────────────────┘
```

### 8.2 Cache Implementation

```csharp
public class CacheService
{
    private readonly IDistributedCache _distributedCache;
    private readonly IMemoryCache _memoryCache;
    
    // L1: Memory cache - 5 minutes
    public async Task<T> GetOrSetMemoryCacheAsync<T>(
        string key,
        Func<Task<T>> factory,
        int minutes = 5)
    {
        if (_memoryCache.TryGetValue(key, out T value))
            return value;
        
        value = await factory();
        
        var cacheOptions = new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(minutes)
        };
        
        _memoryCache.Set(key, value, cacheOptions);
        return value;
    }
    
    // L2: Distributed cache (Redis) - 1 hour
    public async Task<T> GetOrSetDistributedCacheAsync<T>(
        string key,
        Func<Task<T>> factory,
        int minutes = 60)
    {
        var json = await _distributedCache.GetStringAsync(key);
        if (!string.IsNullOrEmpty(json))
            return JsonSerializer.Deserialize<T>(json);
        
        var value = await factory();
        
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(minutes)
        };
        
        await _distributedCache.SetStringAsync(
            key,
            JsonSerializer.Serialize(value),
            options
        );
        
        return value;
    }
    
    // Cache invalidation
    public async Task InvalidateCacheAsync(string pattern)
    {
        // Invalidate specific keys matching pattern
        // Implementation depends on Redis client used
    }
}
```

---

## 9. Performance Optimization

### 9.1 Database Indexing Strategy

```sql
-- Analyze query performance
EXPLAIN ANALYZE
SELECT * FROM workflow.workflows
WHERE tenant_id = $1 AND status = $2
ORDER BY created_at DESC
LIMIT 10;

-- Create missing indexes
CREATE INDEX CONCURRENTLY idx_workflows_tenant_status 
ON workflow.workflows(tenant_id, status, created_at DESC);

-- Partial index for active workflows
CREATE INDEX idx_active_workflows 
ON workflow.workflows(created_at DESC)
WHERE deleted_at IS NULL AND is_published = TRUE;

-- Vector index for semantic search
CREATE INDEX idx_embedding_hnsw 
ON workflow.industry_knowledge 
USING hnsw(embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Monitor index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### 9.2 Query Optimization

```sql
-- Use LIMIT with offset for pagination
SELECT * FROM workflow.workflows
WHERE tenant_id = $1
ORDER BY created_at DESC
LIMIT 20 OFFSET (page - 1) * 20;

-- Avoid SELECT * - use specific columns
SELECT id, name, status, created_at
FROM workflow.workflows
WHERE tenant_id = $1;

-- Use window functions for analytics
SELECT 
    id,
    name,
    execution_count,
    ROW_NUMBER() OVER (ORDER BY execution_count DESC) as rank
FROM workflow.workflows;

-- Use materialized views for complex queries
CREATE MATERIALIZED VIEW v_workflow_stats AS
SELECT 
    w.id,
    COUNT(DISTINCT we.id) as total_executions,
    AVG(we.duration_ms) as avg_duration,
    MAX(we.completed_at) as last_execution
FROM workflow.workflows w
LEFT JOIN workflow.workflow_executions we ON w.id = we.workflow_id
GROUP BY w.id;

CREATE INDEX idx_workflow_stats_executions 
ON v_workflow_stats(total_executions DESC);

-- Refresh materialized view periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY v_workflow_stats;
```

---

## 10. Security & Encryption

### 10.1 Encryption at Rest

```csharp
// Column encryption for sensitive data
public class EncryptionService
{
    private readonly IDataProtectionProvider _protectionProvider;
    
    public string EncryptSensitiveData(string plaintext, string purpose)
    {
        var protector = _protectionProvider.CreateProtector(purpose);
        return protector.Protect(plaintext);
    }
    
    public string DecryptSensitiveData(string ciphertext, string purpose)
    {
        var protector = _protectionProvider.CreateProtector(purpose);
        return protector.Unprotect(ciphertext);
    }
}

// Data Protection Key Store
services.AddDataProtection()
    .PersistKeysToStackExchangeRedis(
        ConnectionMultiplexer.Connect(redisConnection),
        "DataProtectionKeys"
    )
    .ProtectKeysWithAzureKeyVault(
        new Uri("https://keyvault.azure.net/keys/protection-key/version"),
        new DefaultAzureCredential()
    );
```

### 10.2 Row-Level Security (RLS)

```sql
-- Enable RLS on sensitive tables
ALTER TABLE workflow.workflows ENABLE ROW LEVEL SECURITY;
ALTER TABLE workflow.workflow_executions ENABLE ROW LEVEL SECURITY;

-- Create policy for tenant isolation
CREATE POLICY tenant_isolation_policy ON workflow.workflows
AS PERMISSIVE
FOR ALL
TO authenticated
USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Create policy for user-specific data
CREATE POLICY user_workflows ON workflow.workflows
AS PERMISSIVE
FOR SELECT
TO authenticated
USING (created_by = current_user_id() OR tenant_id IN (
    SELECT tenant_id FROM tenant_members WHERE user_id = current_user_id()
));

-- Set session variables for RLS
SET app.tenant_id = '550e8400-e29b-41d4-a716-446655440000';
SET app.user_id = '550e8400-e29b-41d4-a716-446655440001';
```

---

## 11. Backup & Disaster Recovery

### 11.1 Backup Strategy

```bash
# PostgreSQL automated backups
# Set up in Azure Database for PostgreSQL
az postgres server backup show --resource-group myResourceGroup --name myserver

# Manual backup
pg_dump -U dbuser -h hostname database_name > backup.sql

# Restore from backup
psql -U dbuser -h hostname database_name < backup.sql

# Configure backup retention
az postgres server update --name myserver --resource-group myResourceGroup \
    --backup-retention 35 \
    --geo-redundant-backup Enabled
```

### 11.2 Disaster Recovery Plan

```csharp
public class DisasterRecoveryService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly WorkflowDbContext _dbContext;
    
    public async Task CreateFullBackupAsync()
    {
        // Backup database
        var dumpStream = await DumpDatabaseAsync();
        
        // Upload to blob storage with timestamp
        var container = _blobServiceClient.GetBlobContainerClient("backups");
        await container.CreateIfNotExistsAsync();
        
        var blobName = $"database-backup-{DateTime.UtcNow:yyyyMMdd-HHmmss}.sql";
        var blobClient = container.GetBlobClient(blobName);
        
        await blobClient.UploadAsync(dumpStream, overwrite: true);
        
        // Set retention
        await blobClient.SetMetadataAsync(new Dictionary<string, string>
        {
            { "backup_type", "full" },
            { "created_at", DateTime.UtcNow.ToString("O") }
        });
    }
    
    public async Task RestoreFromBackupAsync(string backupDate)
    {
        var container = _blobServiceClient.GetBlobContainerClient("backups");
        var blobClient = container.GetBlobClient($"database-backup-{backupDate}.sql");
        
        var download = await blobClient.DownloadAsync();
        
        // Restore database from stream
        await RestoreDatabaseAsync(download.Value.Content);
    }
}
```

---

## 12. Monitoring & Maintenance

### 12.1 Health Checks

```csharp
public class DataLayerHealthCheck : IHealthCheck
{
    private readonly WorkflowDbContext _dbContext;
    private readonly IConnectionMultiplexer _redis;
    
    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        var data = new Dictionary<string, object>();
        
        // Check PostgreSQL
        try
        {
            await _dbContext.Database.ExecuteSqlRawAsync(
                "SELECT 1",
                cancellationToken
            );
            data["postgres"] = "healthy";
        }
        catch (Exception ex)
        {
            data["postgres"] = $"unhealthy: {ex.Message}";
            return HealthCheckResult.Unhealthy("PostgreSQL connection failed", data);
        }
        
        // Check Redis
        try
        {
            var db = _redis.GetDatabase();
            await db.ExecuteAsync("PING");
            data["redis"] = "healthy";
        }
        catch (Exception ex)
        {
            data["redis"] = $"unhealthy: {ex.Message}";
            return HealthCheckResult.Degraded("Redis connection failed", data);
        }
        
        return HealthCheckResult.Healthy("All data layer components healthy", data);
    }
}

// Register health check
services.AddHealthChecks()
    .AddCheck<DataLayerHealthCheck>("data-layer");
```

### 12.2 Monitoring Queries

```sql
-- Monitor table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'workflow'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Monitor slow queries
SELECT 
    query,
    calls,
    total_time,
    mean_time,
    max_time
FROM pg_stat_statements
WHERE query LIKE '%workflow%'
ORDER BY mean_time DESC
LIMIT 10;

-- Monitor index effectiveness
SELECT 
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    CASE WHEN idx_scan = 0 THEN 'Unused' ELSE 'Used' END as status
FROM pg_stat_user_indexes
WHERE schemaname = 'workflow';

-- Monitor connections
SELECT 
    datname,
    usename,
    application_name,
    state,
    COUNT(*) as count
FROM pg_stat_activity
GROUP BY datname, usename, application_name, state;
```

---

## 13. Data Migration

### 13.1 Migration Strategy

```csharp
public class DataMigrationService
{
    private readonly WorkflowDbContext _sourceContext;
    private readonly WorkflowDbContext _targetContext;
    
    public async Task MigrateAllWorkflowsAsync()
    {
        const int batchSize = 100;
        var skip = 0;
        bool hasMore = true;
        
        while (hasMore)
        {
            // Fetch batch from source
            var workflows = await _sourceContext.Workflows
                .Skip(skip)
                .Take(batchSize)
                .Include(w => w.Nodes)
                .Include(w => w.Edges)
                .ToListAsync();
            
            hasMore = workflows.Count == batchSize;
            
            if (workflows.Any())
            {
                // Migrate to target
                await _targetContext.Workflows.AddRangeAsync(workflows);
                await _targetContext.SaveChangesAsync();
                
                skip += batchSize;
            }
        }
    }
}
```

---

## 14. Compliance & Retention

### 14.1 Data Retention Policy

```sql
-- Archive old data
CREATE PROCEDURE archive_old_executions()
LANGUAGE plpgsql
AS $$
BEGIN
    -- Move executions older than 1 year to archive table
    INSERT INTO workflow.execution_archives
    SELECT * FROM workflow.workflow_executions
    WHERE created_at < CURRENT_DATE - INTERVAL '1 year';
    
    -- Delete from main table
    DELETE FROM workflow.workflow_executions
    WHERE created_at < CURRENT_DATE - INTERVAL '1 year';
END;
$$;

-- Schedule job
SELECT cron.schedule('archive_executions', '0 2 1 * *', 'CALL archive_old_executions()');
```

### 14.2 GDPR Compliance

```csharp
public class GDPRService
{
    public async Task ExportUserDataAsync(string userId)
    {
        // Export all user data in machine-readable format
        var workflows = await _workflowRepository.GetByUserAsync(userId);
        var executions = await _executionRepository.GetByUserAsync(userId);
        var sessions = await _sessionRepository.GetByUserAsync(userId);
        
        var data = new
        {
            workflows,
            executions,
            sessions,
            exportedAt = DateTime.UtcNow
        };
        
        return JsonSerializer.Serialize(data);
    }
    
    public async Task DeleteUserDataAsync(string userId)
    {
        // Delete all user workflows (soft delete with retention period)
        var workflows = await _workflowRepository.GetByUserAsync(userId);
        foreach (var workflow in workflows)
        {
            workflow.DeletedAt = DateTime.UtcNow;
        }
        
        await _workflowRepository.SaveChangesAsync();
    }
}
```

---

## Summary Table

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Relational DB** | PostgreSQL | ACID-compliant transactional data |
| **Vector DB** | pgvector | Semantic search for RAG |
| **Session Cache** | Redis | Conversation memory (24h TTL) |
| **Semantic Cache** | Redis | LLM response caching (1h TTL) |
| **Job Queue** | Hangfire + Redis | Background workflow generation |
| **Blob Storage** | Azure Blob | Immutable export artifacts |
| **ORM** | EF Core + SQLAlchemy | Type-safe data access |
| **Encryption** | Azure Key Vault | Encryption at rest & in transit |
| **Backup** | Azure Backup | Automated disaster recovery |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-06 | Initial LLD for Data Layer Architecture |
