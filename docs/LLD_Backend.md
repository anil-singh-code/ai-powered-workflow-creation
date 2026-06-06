# Low-Level Design (LLD) - Back-End
## AI-Powered Workflow Creation System

**Tech Stack:** .NET 7+, C#, Entity Framework Core, SQL Server, Azure AI Services

---

## Table of Contents
1. [System Architecture](#system-architecture)
2. [API Design](#api-design)
3. [Database Schema](#database-schema)
4. [Service Layer Architecture](#service-layer-architecture)
5. [AI/ML Integration](#aiml-integration)
6. [Authentication & Authorization](#authentication--authorization)
7. [Workflow Engine](#workflow-engine)
8. [Integration Services](#integration-services)
9. [Caching Strategy](#caching-strategy)
10. [Error Handling & Logging](#error-handling--logging)
11. [Performance Optimization](#performance-optimization)
12. [Security Considerations](#security-considerations)
13. [Testing Strategy](#testing-strategy)
14. [Deployment & DevOps](#deployment--devops)

---

## 1. System Architecture

### 1.1 Back-End Architecture Overview

```
┌──────────────────────���──────────────────────────────────────────┐
│                     .NET Backend Application                    │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────┐  ┌──────────────────────┐              │
│  │  API Layer           │  │  Controller Layer    │              │
│  │  (REST Endpoints)    │  │  (Request Handling)  │              │
│  └──────────────────────┘  └──────────────────────┘              │
│  ┌──────────────────────┐  ┌──────────────────────┐              │
│  │  Service Layer       │  │  Business Logic      │              │
│  │  (Core Services)     │  │  (Workflows, Exec)   │              │
│  └──────────────────────┘  └──────────────────────┘              │
│  ┌──────────────────────┐  ┌──────────────────────┐              │
│  │  Repository Layer    │  │  Data Access         │              │
│  │  (Data Access)       │  │  (EF Core)           │              │
│  └──────────────────────┘  └──────────────────────┘              │
│  ┌──────────────────────┐  ┌──────────────────────┐              │
│  │  Cache Layer         │  │  Integration Layer   │              │
│  │  (Redis/In-Memory)   │  │  (External Services) │              │
│  └──────────────────────┘  └──────────────────────┘              │
│  ┌──────────────────────┐  ┌──────────────────────┐              │
│  │  AI Service Layer    │  │  Notification Layer  │              │
│  │  (LLM Integration)   │  │  (Events, Webhooks)  │              │
│  └──────────────────────┘  └──────────────────────┘              │
└─────────────────────────────────────────────────────────────────┘
                              │
                ┌─────────────┼─────────────┐
                │             │             │
        ┌───────▼─────┐ ┌────▼────┐ ┌────▼──────────┐
        │ SQL Server  │ │  Redis  │ │ Azure AI Svc  │
        │  Database   │ │  Cache  │ │ (LLM, ML)     │
        └─────────────┘ └─────────┘ └───────────────┘
```

### 1.2 Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  API Layer - HTTP Endpoints, Request/Response              │
├─────────────────────────────────────────────────────────────┤
│  Controllers - Route Handling, Input Validation             │
├─────────────────────────────────────────────────────────────┤
│  Service Layer - Business Logic, Orchestration              │
├─────────────────────────────────────────────────────────────┤
│  Repository Layer - Data Access Abstraction                 │
├─────────────────────────────────────────────────────────────┤
│  Data Layer - EF Core, Database Models                      │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure - Cache, Logging, Configuration             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. API Design

### 2.1 API Endpoints

#### **Authentication Endpoints**

```
POST   /api/v1/auth/login              - User login
POST   /api/v1/auth/register           - User registration
POST   /api/v1/auth/logout             - User logout
POST   /api/v1/auth/refresh-token      - Refresh JWT token
POST   /api/v1/auth/verify-otp         - Verify OTP
POST   /api/v1/auth/reset-password     - Reset password
GET    /api/v1/auth/me                 - Get current user
```

#### **Workflow Endpoints**

```
GET    /api/v1/workflows               - List all workflows (with pagination/filters)
POST   /api/v1/workflows               - Create new workflow
GET    /api/v1/workflows/{id}          - Get workflow details
PUT    /api/v1/workflows/{id}          - Update workflow
DELETE /api/v1/workflows/{id}          - Delete workflow
POST   /api/v1/workflows/{id}/publish  - Publish workflow
POST   /api/v1/workflows/{id}/duplicate - Duplicate workflow
POST   /api/v1/workflows/{id}/export   - Export workflow (JSON/YAML)
POST   /api/v1/workflows/import        - Import workflow
GET    /api/v1/workflows/{id}/versions - Get workflow versions
POST   /api/v1/workflows/{id}/versions/{versionId}/restore - Restore version
```

#### **Execution Endpoints**

```
POST   /api/v1/workflows/{id}/execute  - Execute workflow
GET    /api/v1/workflows/{id}/executions - Get execution history
GET    /api/v1/executions/{executionId} - Get execution details
GET    /api/v1/executions/{executionId}/logs - Get execution logs
GET    /api/v1/executions/{executionId}/status - Get execution status
POST   /api/v1/executions/{executionId}/cancel - Cancel execution
POST   /api/v1/executions/{executionId}/retry - Retry execution
```

#### **AI Endpoints**

```
POST   /api/v1/ai/suggest              - Get workflow suggestions
POST   /api/v1/ai/generate             - Generate workflow from description
POST   /api/v1/ai/optimize             - Optimize workflow
POST   /api/v1/ai/validate             - Validate workflow
POST   /api/v1/ai/chat                 - Chat with AI assistant
GET    /api/v1/ai/templates            - Get AI templates
```

#### **Integration Endpoints**

```
GET    /api/v1/integrations            - List available integrations
GET    /api/v1/integrations/{type}     - Get integration config
POST   /api/v1/integrations/{type}/connect - Connect integration
POST   /api/v1/integrations/{type}/test    - Test integration
GET    /api/v1/integrations/{type}/actions - List available actions
```

#### **User/Admin Endpoints**

```
GET    /api/v1/users                   - List users (admin only)
GET    /api/v1/users/{id}              - Get user details
PUT    /api/v1/users/{id}              - Update user
DELETE /api/v1/users/{id}              - Delete user (admin only)
GET    /api/v1/admin/analytics         - Get system analytics
GET    /api/v1/admin/logs              - Get system logs
```

### 2.2 Request/Response Format

#### **Standard Request Format**

```json
{
  "id": "uuid",
  "name": "string",
  "description": "string",
  "metadata": {
    "key": "value"
  }
}
```

#### **Standard Response Format**

```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "name": "string",
    "createdAt": "2026-06-06T10:30:00Z"
  },
  "message": "Operation completed successfully",
  "timestamp": "2026-06-06T10:30:00Z"
}
```

#### **Paginated Response**

```json
{
  "success": true,
  "data": [
    { "id": "1", "name": "Workflow 1" },
    { "id": "2", "name": "Workflow 2" }
  ],
  "pagination": {
    "pageNumber": 1,
    "pageSize": 10,
    "totalCount": 50,
    "totalPages": 5,
    "hasNextPage": true,
    "hasPreviousPage": false
  },
  "timestamp": "2026-06-06T10:30:00Z"
}
```

#### **Error Response**

```json
{
  "success": false,
  "error": {
    "code": "WORKFLOW_NOT_FOUND",
    "message": "Workflow with ID 123 not found",
    "details": {
      "workflowId": "123"
    }
  },
  "timestamp": "2026-06-06T10:30:00Z"
}
```

### 2.3 API Versioning

```
Strategy: URL versioning (/api/v1/, /api/v2/)

Benefits:
- Clear version distinction
- Easier backward compatibility
- Client can explicitly choose version
```

### 2.4 Authentication Header

```
Authorization: Bearer {jwt_token}

JWT Payload:
{
  "sub": "user-id",
  "email": "user@example.com",
  "roles": ["USER", "EDITOR"],
  "permissions": ["read:workflow", "create:workflow"],
  "iat": 1609459200,
  "exp": 1609545600
}
```

---

## 3. Database Schema

### 3.1 Entity Relationship Diagram

```
Users (1) ──────────┬─────── (M) Workflows
         │          │
         │          └─────── (M) WorkflowExecutions
         │
         └─────────────────── (M) AuditLogs

Workflows (1) ──┬────────── (M) WorkflowNodes
                ├────────── (M) WorkflowEdges
                ├────────── (M) WorkflowVersions
                └────────── (M) WorkflowExecutions

WorkflowNodes (1) ──────── (M) NodeProperties

WorkflowExecutions (1) ──────── (M) ExecutionLogs
                        ──────── (M) ExecutionSteps
                        
Integrations (1) ──────────── (M) IntegrationCredentials

Templates (1) ──────────────── (M) TemplateNodes
```

### 3.2 Database Tables

#### **Users Table**

```sql
CREATE TABLE Users (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Email NVARCHAR(256) NOT NULL UNIQUE,
    PasswordHash NVARCHAR(MAX) NOT NULL,
    FirstName NVARCHAR(100),
    LastName NVARCHAR(100),
    PhoneNumber NVARCHAR(20),
    ProfilePictureUrl NVARCHAR(MAX),
    Role NVARCHAR(50) NOT NULL DEFAULT 'USER', -- USER, EDITOR, ADMIN
    IsEmailVerified BIT DEFAULT 0,
    IsActive BIT DEFAULT 1,
    LastLoginAt DATETIME2,
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 DEFAULT GETUTCDATE(),
    DeletedAt DATETIME2 NULL
);
```

#### **Workflows Table**

```sql
CREATE TABLE Workflows (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    CreatedById UNIQUEIDENTIFIER NOT NULL,
    Name NVARCHAR(255) NOT NULL,
    Description NVARCHAR(MAX),
    Status NVARCHAR(50) NOT NULL DEFAULT 'DRAFT', -- DRAFT, ACTIVE, INACTIVE, ARCHIVED
    IsPublished BIT DEFAULT 0,
    PublishedAt DATETIME2 NULL,
    Category NVARCHAR(100),
    Tags NVARCHAR(MAX), -- JSON array
    Metadata NVARCHAR(MAX), -- JSON
    TriggerType NVARCHAR(100), -- MANUAL, SCHEDULED, WEBHOOK, EVENT
    RetryPolicy NVARCHAR(MAX), -- JSON
    TimeoutMinutes INT,
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 DEFAULT GETUTCDATE(),
    DeletedAt DATETIME2 NULL,
    FOREIGN KEY (CreatedById) REFERENCES Users(Id)
);
```

#### **WorkflowNodes Table**

```sql
CREATE TABLE WorkflowNodes (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    WorkflowId UNIQUEIDENTIFIER NOT NULL,
    NodeType NVARCHAR(100) NOT NULL, -- Task, Decision, Loop, Trigger, End, Webhook, etc.
    Label NVARCHAR(255) NOT NULL,
    Description NVARCHAR(MAX),
    SequenceOrder INT,
    IsStartNode BIT DEFAULT 0,
    IsEndNode BIT DEFAULT 0,
    Configuration NVARCHAR(MAX), -- JSON
    InputSchema NVARCHAR(MAX), -- JSON Schema
    OutputSchema NVARCHAR(MAX), -- JSON Schema
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (WorkflowId) REFERENCES Workflows(Id) ON DELETE CASCADE
);
```

#### **WorkflowEdges Table**

```sql
CREATE TABLE WorkflowEdges (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    WorkflowId UNIQUEIDENTIFIER NOT NULL,
    SourceNodeId UNIQUEIDENTIFIER NOT NULL,
    TargetNodeId UNIQUEIDENTIFIER NOT NULL,
    EdgeType NVARCHAR(100) DEFAULT 'DEFAULT', -- DEFAULT, SUCCESS, FAILURE, CONDITIONAL
    Condition NVARCHAR(MAX), -- JSON (for conditional edges)
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (WorkflowId) REFERENCES Workflows(Id) ON DELETE CASCADE,
    FOREIGN KEY (SourceNodeId) REFERENCES WorkflowNodes(Id),
    FOREIGN KEY (TargetNodeId) REFERENCES WorkflowNodes(Id)
);
```

#### **WorkflowExecutions Table**

```sql
CREATE TABLE WorkflowExecutions (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    WorkflowId UNIQUEIDENTIFIER NOT NULL,
    InitiatedById UNIQUEIDENTIFIER,
    Status NVARCHAR(50) DEFAULT 'PENDING', -- PENDING, RUNNING, SUCCESS, FAILED, CANCELLED
    StartedAt DATETIME2,
    CompletedAt DATETIME2 NULL,
    Duration INT, -- milliseconds
    Input NVARCHAR(MAX), -- JSON
    Output NVARCHAR(MAX), -- JSON
    ErrorMessage NVARCHAR(MAX),
    ErrorCode NVARCHAR(100),
    RetryAttempt INT DEFAULT 0,
    ExecutionContextId NVARCHAR(MAX), -- JSON
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (WorkflowId) REFERENCES Workflows(Id),
    FOREIGN KEY (InitiatedById) REFERENCES Users(Id)
);

CREATE INDEX IDX_WorkflowExecutions_WorkflowId ON WorkflowExecutions(WorkflowId);
CREATE INDEX IDX_WorkflowExecutions_Status ON WorkflowExecutions(Status);
CREATE INDEX IDX_WorkflowExecutions_CreatedAt ON WorkflowExecutions(CreatedAt DESC);
```

#### **ExecutionSteps Table**

```sql
CREATE TABLE ExecutionSteps (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    ExecutionId UNIQUEIDENTIFIER NOT NULL,
    NodeId UNIQUEIDENTIFIER NOT NULL,
    StepNumber INT NOT NULL,
    Status NVARCHAR(50), -- PENDING, RUNNING, SUCCESS, FAILED, SKIPPED
    Input NVARCHAR(MAX), -- JSON
    Output NVARCHAR(MAX), -- JSON
    ErrorMessage NVARCHAR(MAX),
    StartedAt DATETIME2,
    CompletedAt DATETIME2 NULL,
    Duration INT, -- milliseconds
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (ExecutionId) REFERENCES WorkflowExecutions(Id) ON DELETE CASCADE,
    FOREIGN KEY (NodeId) REFERENCES WorkflowNodes(Id)
);
```

#### **ExecutionLogs Table**

```sql
CREATE TABLE ExecutionLogs (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    ExecutionId UNIQUEIDENTIFIER NOT NULL,
    ExecutionStepId UNIQUEIDENTIFIER,
    LogLevel NVARCHAR(20), -- INFO, WARNING, ERROR, DEBUG
    Message NVARCHAR(MAX),
    Details NVARCHAR(MAX), -- JSON
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (ExecutionId) REFERENCES WorkflowExecutions(Id) ON DELETE CASCADE,
    FOREIGN KEY (ExecutionStepId) REFERENCES ExecutionSteps(Id)
);

CREATE INDEX IDX_ExecutionLogs_ExecutionId ON ExecutionLogs(ExecutionId);
```

#### **Integrations Table**

```sql
CREATE TABLE Integrations (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    Name NVARCHAR(255) NOT NULL,
    IntegrationType NVARCHAR(100) NOT NULL, -- HTTP, Database, ThirdPartyAPI, etc.
    Description NVARCHAR(MAX),
    BaseUrl NVARCHAR(MAX),
    IsActive BIT DEFAULT 1,
    IsBuiltIn BIT DEFAULT 0,
    Configuration NVARCHAR(MAX), -- JSON
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    UpdatedAt DATETIME2 DEFAULT GETUTCDATE()
);
```

#### **AuditLogs Table**

```sql
CREATE TABLE AuditLogs (
    Id UNIQUEIDENTIFIER PRIMARY KEY DEFAULT NEWID(),
    UserId UNIQUEIDENTIFIER,
    EntityType NVARCHAR(100),
    EntityId NVARCHAR(MAX),
    Action NVARCHAR(100), -- CREATE, UPDATE, DELETE, PUBLISH, EXECUTE
    OldValues NVARCHAR(MAX), -- JSON
    NewValues NVARCHAR(MAX), -- JSON
    IpAddress NVARCHAR(50),
    UserAgent NVARCHAR(MAX),
    CreatedAt DATETIME2 DEFAULT GETUTCDATE(),
    FOREIGN KEY (UserId) REFERENCES Users(Id)
);

CREATE INDEX IDX_AuditLogs_UserId ON AuditLogs(UserId);
CREATE INDEX IDX_AuditLogs_CreatedAt ON AuditLogs(CreatedAt DESC);
```

---

## 4. Service Layer Architecture

### 4.1 Service Structure

```csharp
// Project Structure
/Services
  /Authentication
    - IAuthenticationService.cs
    - AuthenticationService.cs
    - JwtTokenService.cs
  /Workflow
    - IWorkflowService.cs
    - WorkflowService.cs
    - IWorkflowValidationService.cs
    - WorkflowValidationService.cs
  /Execution
    - IWorkflowExecutionService.cs
    - WorkflowExecutionService.cs
    - IExecutionEngineService.cs
    - ExecutionEngineService.cs
  /AI
    - IAIService.cs
    - AIService.cs
    - ILLMIntegrationService.cs
    - LLMIntegrationService.cs
  /Integration
    - IIntegrationService.cs
    - IntegrationService.cs
  /Notification
    - INotificationService.cs
    - NotificationService.cs
```

### 4.2 Core Services

#### **AuthenticationService**

```csharp
public interface IAuthenticationService
{
    Task<AuthResponse> LoginAsync(LoginRequest request);
    Task<AuthResponse> RegisterAsync(RegisterRequest request);
    Task LogoutAsync(string userId);
    Task<AuthResponse> RefreshTokenAsync(string refreshToken);
    Task<bool> VerifyOtpAsync(string email, string otp);
    Task SendPasswordResetAsync(string email);
    Task<bool> ResetPasswordAsync(string token, string newPassword);
}

public class AuthenticationService : IAuthenticationService
{
    private readonly IUserRepository _userRepository;
    private readonly IJwtTokenService _jwtTokenService;
    private readonly IPasswordHasher<User> _passwordHasher;
    private readonly ILogger<AuthenticationService> _logger;
    
    public async Task<AuthResponse> LoginAsync(LoginRequest request)
    {
        var user = await _userRepository.GetByEmailAsync(request.Email);
        if (user == null || !_passwordHasher.VerifyHashedPassword(user, user.PasswordHash, request.Password))
        {
            throw new UnauthorizedAccessException("Invalid credentials");
        }
        
        var token = _jwtTokenService.GenerateToken(user);
        var refreshToken = _jwtTokenService.GenerateRefreshToken();
        
        user.LastLoginAt = DateTime.UtcNow;
        await _userRepository.UpdateAsync(user);
        
        return new AuthResponse
        {
            AccessToken = token,
            RefreshToken = refreshToken,
            User = MapToUserDto(user)
        };
    }
}
```

#### **WorkflowService**

```csharp
public interface IWorkflowService
{
    Task<WorkflowDto> CreateAsync(CreateWorkflowRequest request, string userId);
    Task<WorkflowDto> GetByIdAsync(string id);
    Task<PagedResult<WorkflowDto>> GetAllAsync(WorkflowFilter filter, int pageNumber, int pageSize);
    Task<WorkflowDto> UpdateAsync(string id, UpdateWorkflowRequest request, string userId);
    Task DeleteAsync(string id, string userId);
    Task<WorkflowDto> PublishAsync(string id, string userId);
    Task<WorkflowDto> DuplicateAsync(string id, string userId);
    Task<byte[]> ExportAsync(string id, ExportFormat format);
    Task<WorkflowDto> ImportAsync(byte[] data, string userId);
}

public class WorkflowService : IWorkflowService
{
    private readonly IWorkflowRepository _workflowRepository;
    private readonly IWorkflowValidationService _validationService;
    private readonly IAuditService _auditService;
    private readonly ICache _cache;
    
    public async Task<WorkflowDto> CreateAsync(CreateWorkflowRequest request, string userId)
    {
        // Validate input
        var validation = await _validationService.ValidateWorkflowAsync(request);
        if (!validation.IsValid)
            throw new ValidationException(validation.Errors);
        
        // Create workflow
        var workflow = new Workflow
        {
            Name = request.Name,
            Description = request.Description,
            CreatedById = userId,
            Status = WorkflowStatus.Draft
        };
        
        await _workflowRepository.CreateAsync(workflow);
        await _auditService.LogAsync(new AuditLog
        {
            UserId = userId,
            Action = AuditAction.Create,
            EntityType = nameof(Workflow),
            EntityId = workflow.Id.ToString()
        });
        
        return MapToDto(workflow);
    }
}
```

#### **WorkflowExecutionService**

```csharp
public interface IWorkflowExecutionService
{
    Task<WorkflowExecutionDto> ExecuteAsync(string workflowId, Dictionary<string, object> input, string userId);
    Task<PagedResult<WorkflowExecutionDto>> GetExecutionHistoryAsync(string workflowId, int pageNumber, int pageSize);
    Task<WorkflowExecutionDto> GetExecutionDetailsAsync(string executionId);
    Task<List<ExecutionLogDto>> GetExecutionLogsAsync(string executionId);
    Task CancelExecutionAsync(string executionId);
    Task RetryExecutionAsync(string executionId);
}

public class WorkflowExecutionService : IWorkflowExecutionService
{
    private readonly IExecutionEngineService _executionEngine;
    private readonly IWorkflowExecutionRepository _executionRepository;
    private readonly INotificationService _notificationService;
    
    public async Task<WorkflowExecutionDto> ExecuteAsync(string workflowId, Dictionary<string, object> input, string userId)
    {
        var workflow = await _workflowRepository.GetByIdAsync(workflowId);
        if (workflow == null || !workflow.IsPublished)
            throw new BusinessException("Workflow not found or not published");
        
        var execution = new WorkflowExecution
        {
            Id = Guid.NewGuid(),
            WorkflowId = workflowId,
            InitiatedById = userId,
            Status = ExecutionStatus.Pending,
            Input = JsonSerializer.Serialize(input)
        };
        
        await _executionRepository.CreateAsync(execution);
        
        // Queue execution
        _ = _executionEngine.ExecuteAsync(execution);
        
        return MapToDto(execution);
    }
}
```

---

## 5. AI/ML Integration

### 5.1 LLM Service Implementation

```csharp
public interface ILLMIntegrationService
{
    Task<string> GenerateAsync(string prompt, GenerationOptions options = null);
    Task<T> GenerateStructuredAsync<T>(string prompt, string jsonSchema = null);
    Task<string> ChatAsync(string message, List<ChatMessage> history);
}

public class AzureOpenAIService : ILLMIntegrationService
{
    private readonly OpenAIClient _client;
    private readonly ILogger<AzureOpenAIService> _logger;
    private const string DeploymentId = "workflow-generation";
    
    public async Task<string> GenerateAsync(string prompt, GenerationOptions options = null)
    {
        try
        {
            var chatCompletionsOptions = new ChatCompletionsOptions
            {
                DeploymentName = DeploymentId,
                Messages =
                {
                    new ChatMessage(ChatRole.System, "You are a workflow automation expert."),
                    new ChatMessage(ChatRole.User, prompt)
                },
                Temperature = options?.Temperature ?? 0.7f,
                MaxTokens = options?.MaxTokens ?? 2000,
                TopP = 0.95f
            };
            
            var response = await _client.GetChatCompletionsAsync(chatCompletionsOptions);
            return response.Value.Choices[0].Message.Content;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "LLM generation failed");
            throw;
        }
    }
}
```

---

## 6. Authentication & Authorization

### 6.1 JWT Token Implementation

```csharp
public class JwtTokenService
{
    private readonly IConfiguration _configuration;
    
    public string GenerateToken(User user, int expiryMinutes = 60)
    {
        var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Secret"]));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);
        
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim("Role", user.Role)
        };
        
        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(expiryMinutes),
            signingCredentials: credentials
        );
        
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### 6.2 Authorization Policies

```csharp
public static class AuthorizationPolicies
{
    public static void AddAuthorizationPolicies(this IServiceCollection services)
    {
        services.AddAuthorizationBuilder()
            .AddPolicy("AdminOnly", policy => 
                policy.RequireRole("ADMIN"))
            .AddPolicy("EditorOrAbove", policy => 
                policy.RequireRole("EDITOR", "ADMIN"));
    }
}
```

---

## 7. Workflow Engine

### 7.1 Execution Engine Service

```csharp
public class ExecutionEngineService : IExecutionEngineService
{
    public async Task ExecuteAsync(WorkflowExecution execution)
    {
        try
        {
            execution.Status = ExecutionStatus.Running;
            execution.StartedAt = DateTime.UtcNow;
            await _executionRepository.UpdateAsync(execution);
            
            var workflow = await _workflowRepository.GetByIdAsync(execution.WorkflowId);
            var nodes = await GetOrderedNodesAsync(workflow);
            
            var context = new ExecutionContext
            {
                Execution = execution,
                Workflow = workflow,
                Variables = JsonSerializer.Deserialize<Dictionary<string, object>>(execution.Input)
            };
            
            // Execute nodes in sequence
            foreach (var node in nodes)
            {
                await ExecuteNodeAsync(context, node);
                
                if (context.ShouldStop)
                    break;
            }
            
            execution.Status = ExecutionStatus.Success;
            execution.Output = JsonSerializer.Serialize(context.Variables);
        }
        catch (Exception ex)
        {
            execution.Status = ExecutionStatus.Failed;
            execution.ErrorMessage = ex.Message;
        }
        finally
        {
            execution.CompletedAt = DateTime.UtcNow;
            execution.Duration = (int)(execution.CompletedAt.Value - execution.StartedAt.Value).TotalMilliseconds;
            await _executionRepository.UpdateAsync(execution);
        }
    }
}
```

---

## 8. Integration Services

### 8.1 Integration Service Pattern

```csharp
public interface IIntegrationService
{
    Task<Dictionary<string, object>> ExecuteAsync(
        string integrationType, 
        string action, 
        Dictionary<string, object> parameters);
    Task<bool> TestConnectionAsync(string integrationType, ICredentials credentials);
    Task<List<ActionDefinition>> GetActionsAsync(string integrationType);
}
```

### 8.2 HTTP Integration Handler

```csharp
public class HttpIntegrationHandler : IIntegrationHandler
{
    public async Task<Dictionary<string, object>> ExecuteAsync(string action, Dictionary<string, object> parameters)
    {
        var httpConfig = JsonSerializer.Deserialize<HttpNodeConfig>(parameters["config"].ToString());
        
        using var client = new HttpClient();
        
        var request = new HttpRequestMessage
        {
            Method = new HttpMethod(httpConfig.Method),
            RequestUri = new Uri(httpConfig.Url)
        };
        
        if (httpConfig.Body != null)
            request.Content = new StringContent(
                JsonSerializer.Serialize(httpConfig.Body),
                Encoding.UTF8,
                "application/json"
            );
        
        var response = await client.SendAsync(request);
        var content = await response.Content.ReadAsStringAsync();
        
        return new Dictionary<string, object>
        {
            { "statusCode", (int)response.StatusCode },
            { "body", JsonSerializer.Deserialize<object>(content) }
        };
    }
}
```

---

## 9. Caching Strategy

### 9.1 Cache Implementation

```csharp
public interface ICache
{
    T Get<T>(string key);
    Task<T> GetAsync<T>(string key);
    void Set<T>(string key, T value, TimeSpan? expiration = null);
    Task SetAsync<T>(string key, T value, TimeSpan? expiration = null);
    void Remove(string key);
    Task RemoveAsync(string key);
}

public class DistributedCacheService : ICache
{
    private readonly IDistributedCache _distributedCache;
    
    public T Get<T>(string key)
    {
        var value = _distributedCache.GetString(key);
        return value == null ? default : JsonSerializer.Deserialize<T>(value);
    }
    
    public void Set<T>(string key, T value, TimeSpan? expiration = null)
    {
        _distributedCache.SetString(
            key,
            JsonSerializer.Serialize(value),
            new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = expiration }
        );
    }
}
```

---

## 10. Error Handling & Logging

### 10.1 Custom Exceptions

```csharp
public abstract class ApplicationException : Exception
{
    public string Code { get; set; }
    public object Details { get; set; }
    
    public ApplicationException(string message, string code = null, object details = null)
        : base(message)
    {
        Code = code;
        Details = details;
    }
}

public class ValidationException : ApplicationException
{
    public ValidationException(List<string> errors)
        : base("Validation failed", "VALIDATION_ERROR", errors) { }
}

public class BusinessException : ApplicationException
{
    public BusinessException(string message)
        : base(message, "BUSINESS_ERROR") { }
}

public class NotFoundException : ApplicationException
{
    public NotFoundException(string resource, string id)
        : base($"{resource} with ID {id} not found", "NOT_FOUND") { }
}
```

### 10.2 Global Exception Handler

```csharp
public class GlobalExceptionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionMiddleware> _logger;
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await HandleExceptionAsync(context, ex);
        }
    }
    
    private static Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";
        var response = new ErrorResponse();
        
        switch (exception)
        {
            case ValidationException ve:
                context.Response.StatusCode = StatusCodes.Status400BadRequest;
                response = new ErrorResponse
                {
                    Code = ve.Code,
                    Message = ve.Message,
                    Details = ve.Details
                };
                break;
            case NotFoundException nf:
                context.Response.StatusCode = StatusCodes.Status404NotFound;
                response.Code = nf.Code;
                response.Message = nf.Message;
                break;
            default:
                context.Response.StatusCode = StatusCodes.Status500InternalServerError;
                response.Code = "INTERNAL_SERVER_ERROR";
                response.Message = "An unexpected error occurred";
                break;
        }
        
        return context.Response.WriteAsJsonAsync(response);
    }
}
```

---

## 11. Performance Optimization

### 11.1 Database Query Optimization

```csharp
// Include related entities to avoid N+1 queries
var workflows = await _dbContext.Workflows
    .AsNoTracking()
    .Include(w => w.Nodes)
    .Include(w => w.Edges)
    .Where(w => w.CreatedById == userId)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// Use projections to select only needed columns
var workflowDtos = await _dbContext.Workflows
    .AsNoTracking()
    .Where(w => w.CreatedById == userId)
    .Select(w => new WorkflowSummaryDto
    {
        Id = w.Id,
        Name = w.Name,
        Status = w.Status,
        CreatedAt = w.CreatedAt
    })
    .ToListAsync();
```

### 11.2 Async/Await Pattern

```csharp
// Always use async operations
public async Task<WorkflowDto> GetByIdAsync(string id)
{
    return await _repository.GetByIdAsync(id);
}

// Parallel operations when possible
var workflows = await Task.WhenAll(
    _workflowRepository.GetByIdAsync(id1),
    _workflowRepository.GetByIdAsync(id2),
    _workflowRepository.GetByIdAsync(id3)
);
```

---

## 12. Security Considerations

### 12.1 Input Validation & Sanitization

```csharp
public class CreateWorkflowRequest
{
    [Required(ErrorMessage = "Name is required")]
    [StringLength(255, MinimumLength = 3)]
    public string Name { get; set; }
    
    [StringLength(5000)]
    public string Description { get; set; }
}
```

### 12.2 CORS Configuration

```csharp
services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", builder =>
    {
        builder
            .WithOrigins("https://app.example.com")
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

app.UseCors("AllowFrontend");
```

### 12.3 Rate Limiting

```csharp
services.AddRateLimiting(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(httpContext =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: httpContext.User.FindFirst(ClaimTypes.NameIdentifier)?.Value,
            factory: partition => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 100,
                Window = TimeSpan.FromMinutes(1)
            }));
});

app.UseRateLimiter();
```

---

## 13. Testing Strategy

### 13.1 Unit Testing

```csharp
[TestClass]
public class WorkflowServiceTests
{
    private Mock<IWorkflowRepository> _mockRepository;
    private WorkflowService _service;
    
    [TestInitialize]
    public void Setup()
    {
        _mockRepository = new Mock<IWorkflowRepository>();
        _service = new WorkflowService(_mockRepository.Object);
    }
    
    [TestMethod]
    public async Task CreateAsync_WithValidInput_ReturnsWorkflow()
    {
        // Arrange
        var request = new CreateWorkflowRequest { Name = "Test Workflow" };
        var expectedWorkflow = new Workflow { Id = Guid.NewGuid(), Name = "Test Workflow" };
        
        _mockRepository.Setup(r => r.CreateAsync(It.IsAny<Workflow>()))
            .ReturnsAsync(expectedWorkflow);
        
        // Act
        var result = await _service.CreateAsync(request, "user-1");
        
        // Assert
        Assert.IsNotNull(result);
        Assert.AreEqual("Test Workflow", result.Name);
    }
}
```

---

## 14. Deployment & DevOps

### 14.1 Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src

COPY ["WorkflowCreation.sln", "."]
COPY ["src/", "src/"]

RUN dotnet restore
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime
WORKDIR /app

COPY --from=publish /app/publish .

EXPOSE 80
ENV ASPNETCORE_URLS=http://+:80

ENTRYPOINT ["dotnet", "WorkflowCreation.Api.dll"]
```

### 14.2 docker-compose.yml

```yaml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "5000:80"
    environment:
      - ConnectionStrings__DefaultConnection=Server=sqlserver;Database=WorkflowDb;User Id=sa;Password=YourPassword123!
      - Redis__ConnectionString=redis:6379
    depends_on:
      - sqlserver
      - redis

  sqlserver:
    image: mcr.microsoft.com/mssql/server:2019-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=YourPassword123!
    ports:
      - "1433:1433"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

### 14.3 Startup Configuration

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddSwaggerGen();

// Database
builder.Services.AddDbContext<WorkflowDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Cache
builder.Services.AddStackExchangeRedisCache(options =>
    options.Configuration = builder.Configuration.GetConnectionString("Redis"));

// Services
builder.Services.AddScoped<IAuthenticationService, AuthenticationService>();
builder.Services.AddScoped<IWorkflowService, WorkflowService>();
builder.Services.AddScoped<IWorkflowExecutionService, WorkflowExecutionService>();

var app = builder.Build();

app.UseMiddleware<GlobalExceptionMiddleware>();
app.UseSwagger();
app.UseRouting();
app.UseCors("AllowFrontend");
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.Run();
```

---

## Project Structure

```
WorkflowCreation.Api/
├── Controllers/
│   ├── AuthController.cs
│   ├── WorkflowController.cs
│   ├── ExecutionController.cs
│   └── AIController.cs
├── Services/
│   ├── Authentication/
│   ├── Workflow/
│   ├── Execution/
│   ├── AI/
│   └── Integration/
├── Repositories/
│   ├── IRepository.cs
│   ├── IWorkflowRepository.cs
│   └── WorkflowRepository.cs
├── Models/
│   ├── Entities/
│   ├── DTOs/
│   └── Requests/
├── Data/
│   ├── WorkflowDbContext.cs
│   └── Migrations/
├── Middleware/
│   └── GlobalExceptionMiddleware.cs
└── Program.cs
```

---

## Dependencies

```xml
<ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="7.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="7.0.0" />
    <PackageReference Include="StackExchange.Redis" Version="2.6.0" />
    <PackageReference Include="Azure.AI.OpenAI" Version="1.0.0" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    <PackageReference Include="FluentValidation" Version="11.0.0" />
</ItemGroup>
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-06 | Initial LLD for Back-End Architecture |

---

## Glossary

- **LLD:** Low-Level Design
- **EF Core:** Entity Framework Core
- **JWT:** JSON Web Token
- **CORS:** Cross-Origin Resource Sharing
- **DTOs:** Data Transfer Objects
