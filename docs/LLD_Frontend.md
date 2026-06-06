# Low-Level Design (LLD) - Front-End
## AI-Powered Workflow Creation System

**Tech Stack:** React, TypeScript, Redux/Context API, Material-UI, Axios

---

## Table of Contents
1. [System Architecture](#system-architecture)
2. [Component Hierarchy](#component-hierarchy)
3. [State Management](#state-management)
4. [API Integration](#api-integration)
5. [UI/UX Components](#uiux-components)
6. [AI Integration Points](#ai-integration-points)
7. [Workflow Editor](#workflow-editor)
8. [Authentication & Authorization](#authentication--authorization)
9. [Error Handling & Validation](#error-handling--validation)
10. [Performance Optimization](#performance-optimization)
11. [Security Considerations](#security-considerations)
12. [Testing Strategy](#testing-strategy)
13. [Deployment & Build Process](#deployment--build-process)

---

## 1. System Architecture

### 1.1 Front-End Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     React Application                       │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  UI Layer        │  │  State Layer     │                 │
│  │  (Components)    │  │  (Redux/Context) │                 │
│  └──────────────────┘  └──────────────────┘                 │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  API Layer       │  │  AI Services     │                 │
│  │  (Axios/Fetch)   │  │  (LLM, ML)       │                 │
│  └──────────────────┘  └──────────────────┘                 │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │  Utils/Helpers   │  │  Auth Module     │                 │
│  │  (Validators)    │  │  (JWT, OAuth)    │                 │
│  └──────────────────┘  └──────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
            ┌───────▼────────┐  ┌──────▼──────────┐
            │  .NET Backend  │  │  AI Service     │
            │  REST API      │  │  (Python/Azure) │
            └────────────────┘  └─────────────────┘
```

### 1.2 Layered Architecture

- **Presentation Layer:** React Components, Pages
- **State Management Layer:** Redux Store, Actions, Reducers
- **API Integration Layer:** Axios instances, API client services
- **Business Logic Layer:** Validators, formatters, helpers
- **AI Integration Layer:** LLM integration, AI service calls

---

## 2. Component Hierarchy

### 2.1 High-Level Component Structure

```
App
├── Layout
│   ├── Header
│   ├── Sidebar
│   └── Footer
├── Router
│   ├── Dashboard Page
│   │   ├── WorkflowList
│   │   ├── WorkflowStats
│   │   └── RecentWorkflows
│   ├── Workflow Editor Page
│   │   ├── WorkflowCanvas
│   │   ├── NodePalette
│   │   ├── PropertiesPanel
│   │   └── AIAssistant
│   ├── Workflow Details Page
│   │   ├── WorkflowInfo
│   │   ├── ExecutionHistory
│   │   └── Analytics
│   ├── Settings Page
│   ├── Profile Page
│   └── NotFound Page
└── Auth Components
    ├── LoginForm
    ├── RegisterForm
    └── OAuth Integration
```

### 2.2 Core Components

#### **A. Layout Components**

```typescript
// Header Component
├── Logo
├── Navigation Menu
├── User Profile Dropdown
├── Notifications Bell
└── Search Bar

// Sidebar Component
├── Navigation Links
├── Recent Workflows
├── Quick Actions
└── Collapsible Menu

// Footer Component
└── Links & Info
```

#### **B. Workflow Editor Components**

```typescript
// WorkflowEditor
├── Toolbar
│   ├── Save Button
│   ├── Publish Button
│   ├── AI Assist Button
│   ├── Undo/Redo
│   └── Preview
├── WorkflowCanvas (React Flow)
│   ├── Nodes (Task, Decision, Loop, etc.)
│   ├── Edges (Connections)
│   └── Grid Background
├── NodePalette (Sidebar)
│   ├── Task Nodes
│   ├── Decision Nodes
│   ├── Loop Nodes
│   ├── Integration Nodes
│   └── Custom Nodes
├── PropertiesPanel (Right Sidebar)
│   ├── Node Properties Form
│   ├── Validation Messages
│   └── Advanced Settings
└── AIAssistant Panel
    ├── AI Suggestions
    ├── Chat Interface
    └── Auto-Generate Options
```

#### **C. Dashboard Components**

```typescript
// Dashboard
├── WorkflowList
│   ├── Table/Grid View
│   ├── Filters & Search
│   ├── Pagination
│   └── Bulk Actions
├── WorkflowStats
│   ├── Total Workflows Card
│   ├── Active Workflows Card
│   ├── Success Rate Chart
│   └── Execution Time Chart
├── RecentWorkflows
└── Create Workflow Button
```

#### **D. Workflow Execution Components**

```typescript
// ExecutionHistory
├── Execution List
├── Execution Details
├── Execution Timeline
├── Log Viewer
└── Error Details
```

---

## 3. State Management

### 3.1 Redux Store Structure

```typescript
store/
├── slices/
│   ├── authSlice.ts
│   │   └── { user, token, isAuthenticated, loading, error }
│   ├── workflowSlice.ts
│   │   └── { workflows, currentWorkflow, loading, error }
│   ├── editorSlice.ts
│   │   └── { nodes, edges, selectedNode, history, isDirty }
│   ├── uiSlice.ts
│   │   └── { theme, sidebarOpen, notifications, modals }
│   ├── aiSlice.ts
│   │   └── { suggestions, aiLoading, aiError, chatHistory }
│   └── executionSlice.ts
│       └── { executions, currentExecution, logs }
├── selectors/
│   ├── authSelectors.ts
│   ├── workflowSelectors.ts
│   ├── editorSelectors.ts
│   └── aiSelectors.ts
└── middleware/
    ├── apiMiddleware.ts
    └── aiMiddleware.ts
```

### 3.2 State Shape Example

```typescript
// Auth State
{
  auth: {
    user: { id, email, name, role },
    token: 'jwt_token',
    isAuthenticated: true,
    loading: false,
    error: null
  },
  
  // Workflow State
  workflows: {
    items: [
      { id, name, description, status, createdAt, updatedAt },
      ...
    ],
    currentWorkflow: { id, name, nodes, edges, config },
    loading: false,
    error: null
  },
  
  // Editor State
  editor: {
    nodes: [
      { id, type, position, data, selected },
      ...
    ],
    edges: [
      { id, source, target, animated, data },
      ...
    ],
    selectedNode: { id, type },
    history: { past: [], present: {}, future: [] },
    isDirty: false
  },
  
  // AI State
  ai: {
    suggestions: [],
    chatHistory: [],
    aiLoading: false,
    aiError: null
  }
}
```

### 3.3 Actions

```typescript
// Auth Actions
- LOGIN_REQUEST / LOGIN_SUCCESS / LOGIN_FAILURE
- LOGOUT
- REFRESH_TOKEN
- REGISTER_REQUEST / REGISTER_SUCCESS / REGISTER_FAILURE

// Workflow Actions
- FETCH_WORKFLOWS_REQUEST / SUCCESS / FAILURE
- CREATE_WORKFLOW_REQUEST / SUCCESS / FAILURE
- UPDATE_WORKFLOW_REQUEST / SUCCESS / FAILURE
- DELETE_WORKFLOW_REQUEST / SUCCESS / FAILURE
- GET_WORKFLOW_DETAILS_REQUEST / SUCCESS / FAILURE

// Editor Actions
- ADD_NODE
- REMOVE_NODE
- UPDATE_NODE
- ADD_EDGE
- REMOVE_EDGE
- SELECT_NODE
- DESELECT_NODE
- UNDO
- REDO
- MARK_DIRTY
- MARK_CLEAN

// AI Actions
- GET_AI_SUGGESTIONS_REQUEST / SUCCESS / FAILURE
- SEND_AI_CHAT_MESSAGE_REQUEST / SUCCESS / FAILURE
- AI_GENERATE_WORKFLOW_REQUEST / SUCCESS / FAILURE
```

---

## 4. API Integration

### 4.1 API Service Layer

```typescript
services/
├── api/
│   ├── apiClient.ts (Axios instance with interceptors)
│   ├── authService.ts
│   ├── workflowService.ts
│   ├── executionService.ts
│   ├── aiService.ts
│   └── integrationService.ts
└── interceptors/
    ├── authInterceptor.ts
    ├── errorInterceptor.ts
    └── requestInterceptor.ts
```

### 4.2 API Client Setup

```typescript
// apiClient.ts
const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  timeout: 30000,
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request Interceptor
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('authToken');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response Interceptor
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Handle unauthorized
    }
    return Promise.reject(error);
  }
);
```

### 4.3 API Service Methods

#### **Authentication Service**

```typescript
authService {
  login(email, password): Promise<{ token, user }>
  register(userData): Promise<{ user }>
  logout(): Promise<void>
  refreshToken(): Promise<{ token }>
  verifyOTP(email, otp): Promise<{ verified }>
  resetPassword(email): Promise<void>
}
```

#### **Workflow Service**

```typescript
workflowService {
  getAll(filters, pagination): Promise<{ items, total }>
  getById(id): Promise<workflow>
  create(workflowData): Promise<workflow>
  update(id, workflowData): Promise<workflow>
  delete(id): Promise<void>
  publish(id): Promise<workflow>
  duplicate(id): Promise<workflow>
  export(id): Promise<blob>
  import(file): Promise<workflow>
}
```

#### **AI Service**

```typescript
aiService {
  getSuggestions(workflowContext): Promise<suggestions>
  generateWorkflow(description): Promise<workflow>
  optimizeWorkflow(workflowId): Promise<suggestions>
  validateWorkflow(workflow): Promise<{ isValid, errors }>
  chatWithAI(message, context): Promise<response>
}
```

#### **Execution Service**

```typescript
executionService {
  executeWorkflow(id): Promise<executionId>
  getExecutionHistory(workflowId, pagination): Promise<executions>
  getExecutionDetails(executionId): Promise<executionDetails>
  getLogs(executionId): Promise<logs>
  cancelExecution(executionId): Promise<void>
  retryExecution(executionId): Promise<executionId>
}
```

### 4.4 Error Handling

```typescript
// API Error Class
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public data?: any
  ) {
    super(message);
  }
}

// Error Response Format
{
  statusCode: number,
  message: string,
  errors: [{ field, message }],
  timestamp: ISO8601,
  path: string
}
```

---

## 5. UI/UX Components

### 5.1 Component Library (Material-UI Based)

```typescript
components/
├── common/
│   ├── Button.tsx
│   ├── Input.tsx
│   ├── Select.tsx
│   ├── Modal.tsx
│   ├── Toast/Notification.tsx
│   ├── Loading.tsx
│   ├── Card.tsx
│   ├── Badge.tsx
│   └── Avatar.tsx
├── forms/
│   ├── WorkflowForm.tsx
│   ├── NodePropertiesForm.tsx
│   ├── AuthForm.tsx
│   └── FilterForm.tsx
├── tables/
│   ├── DataTable.tsx
│   ├── WorkflowTable.tsx
│   └── ExecutionTable.tsx
├── charts/
│   ├── LineChart.tsx
│   ├── BarChart.tsx
│   ├── PieChart.tsx
│   └── TimelineChart.tsx
└── workflow/
    ├── WorkflowNode.tsx
    ├── WorkflowEdge.tsx
    ├── NodePalette.tsx
    └── PropertiesPanel.tsx
```

### 5.2 Theme & Styling

```typescript
theme/
├── colors.ts
│   ├── Primary Color Palette
│   ├── Secondary Color Palette
│   ├── Status Colors (Success, Error, Warning, Info)
│   └── Neutral Colors
├── typography.ts
│   ├── Font Families
│   ├── Font Sizes
│   └── Font Weights
├── spacing.ts
├── breakpoints.ts (Responsive Design)
├── shadows.ts
└── theme.ts (MUI Theme Configuration)
```

### 5.3 Responsive Design

```typescript
// Breakpoints
xs: 0px      // Mobile
sm: 600px    // Tablet
md: 960px    // Small Desktop
lg: 1280px   // Desktop
xl: 1920px   // Large Desktop

// Usage
<Grid container spacing={2}>
  <Grid item xs={12} sm={6} md={4} lg={3}>
    {/* Component */}
  </Grid>
</Grid>
```

---

## 6. AI Integration Points

### 6.1 AI Features

#### **A. Smart Workflow Generation**
- User describes workflow in natural language
- AI generates workflow nodes and connections
- User reviews and modifies the generated workflow

```typescript
// AI Generation Flow
User Input → LLM Processing → Workflow Structure → UI Rendering
```

#### **B. Workflow Optimization**
- AI analyzes current workflow
- Provides suggestions for optimization
- Recommends best practices

#### **C. Auto-Completion**
- Smart node property suggestions
- Auto-fill based on context
- Error prediction and prevention

#### **D. Interactive AI Assistant**
- Chat interface for workflow help
- Real-time suggestions
- Debugging assistance

### 6.2 AI Service Integration

```typescript
// AI Chat Component
<AIChatPanel
  onSendMessage={handleAIMessage}
  onApplySuggestion={handleApplySuggestion}
  suggestions={aiSuggestions}
  loading={aiLoading}
/>

// AI Suggestion Handler
const handleAIMessage = async (message: string) => {
  dispatch(sendAIChatMessage(message))
    .then((response) => {
      updateWorkflowWithSuggestions(response);
    })
    .catch((error) => {
      showErrorNotification(error);
    });
};
```

### 6.3 LLM Integration

```typescript
// AI Service Call
const generateWorkflow = async (description: string) => {
  const response = await aiService.generateWorkflow({
    description,
    templates: availableTemplates,
    userPreferences: userSettings
  });
  
  return {
    nodes: response.nodes,
    edges: response.edges,
    config: response.configuration,
    explanation: response.explanation
  };
};
```

---

## 7. Workflow Editor

### 7.1 React Flow Integration

```typescript
// Workflow Editor Setup
import ReactFlow, { 
  MiniMap, 
  Controls, 
  Background,
  useNodesState,
  useEdgesState
} from 'reactflow';

const WorkflowEditor = () => {
  const [nodes, setNodes, onNodesChange] = useNodesState(initialNodes);
  const [edges, setEdges, onEdgesChange] = useEdgesState(initialEdges);
  
  const onConnect = (connection) => {
    setEdges((eds) => addEdge(connection, eds));
  };
  
  return (
    <ReactFlow
      nodes={nodes}
      edges={edges}
      onNodesChange={onNodesChange}
      onEdgesChange={onEdgesChange}
      onConnect={onConnect}
      nodeTypes={nodeTypes}
      edgeTypes={edgeTypes}
    >
      <Background />
      <Controls />
      <MiniMap />
    </ReactFlow>
  );
};
```

### 7.2 Node Types

```typescript
// Available Node Types
1. TaskNode - Represents a single task/action
2. DecisionNode - Conditional branching
3. LoopNode - Iterative operations
4. TriggerNode - Workflow start point
5. EndNode - Workflow end point
6. WebhookNode - Webhook integration
7. IntegrationNode - Third-party service integration
8. ApprovalNode - Manual approval required
9. NotificationNode - Send notifications
10. DataTransformNode - Data transformation/mapping
```

### 7.3 Custom Node Component

```typescript
const CustomNode = ({ data, selected }) => {
  return (
    <div className={`node ${selected ? 'selected' : ''}`}>
      <div className="node-header">
        <Icon type={data.type} />
        <span>{data.label}</span>
      </div>
      <div className="node-content">
        {data.description && <p>{data.description}</p>}
      </div>
      <Handle type="target" position={Position.Top} />
      <Handle type="source" position={Position.Bottom} />
    </div>
  );
};
```

### 7.4 Undo/Redo Mechanism

```typescript
// History Management
class EditorHistory {
  private history: WorkflowState[] = [];
  private currentIndex: number = -1;
  
  push(state: WorkflowState): void {
    this.history = this.history.slice(0, this.currentIndex + 1);
    this.history.push(state);
    this.currentIndex++;
  }
  
  undo(): WorkflowState | null {
    if (this.currentIndex > 0) {
      this.currentIndex--;
      return this.history[this.currentIndex];
    }
    return null;
  }
  
  redo(): WorkflowState | null {
    if (this.currentIndex < this.history.length - 1) {
      this.currentIndex++;
      return this.history[this.currentIndex];
    }
    return null;
  }
}
```

---

## 8. Authentication & Authorization

### 8.1 Auth Flow

```typescript
// Login Flow
1. User enters credentials
2. Submit to /api/auth/login
3. Receive JWT token
4. Store in localStorage
5. Redirect to dashboard
6. Token included in subsequent requests

// Token Refresh Flow
1. Check token expiration
2. If expired, call /api/auth/refresh
3. Receive new token
4. Update in localStorage
5. Continue operation
```

### 8.2 Protected Routes

```typescript
// ProtectedRoute Component
const ProtectedRoute = ({ component: Component, requiredRole }) => {
  const { isAuthenticated, user } = useSelector(selectAuth);
  
  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && !hasRole(user, requiredRole)) {
    return <Navigate to="/unauthorized" />;
  }
  
  return <Component />;
};

// Route Configuration
<Routes>
  <Route path="/login" element={<Login />} />
  <Route
    path="/dashboard"
    element={<ProtectedRoute component={Dashboard} />}
  />
  <Route
    path="/admin"
    element={<ProtectedRoute component={Admin} requiredRole="ADMIN" />}
  />
</Routes>
```

### 8.3 Role-Based Access Control

```typescript
// RBAC Setup
const roles = {
  ADMIN: ['read', 'create', 'update', 'delete', 'publish', 'manage_users'],
  EDITOR: ['read', 'create', 'update', 'publish'],
  VIEWER: ['read'],
  GUEST: ['read']
};

// Permission Check
const hasPermission = (user, permission) => {
  return roles[user.role].includes(permission);
};
```

---

## 9. Error Handling & Validation

### 9.1 Validation Strategy

```typescript
// Form Validation
import { yup } from 'yup';

const workflowSchema = yup.object().shape({
  name: yup.string().required('Name is required').min(3).max(100),
  description: yup.string().max(500),
  nodes: yup.array().min(1, 'At least one node required'),
  edges: yup.array()
});

// Usage with React Hook Form
const { register, handleSubmit, formState: { errors } } = useForm({
  resolver: yupResolver(workflowSchema)
});
```

### 9.2 Error Boundary

```typescript
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };
  
  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }
  
  componentDidCatch(error, errorInfo) {
    logger.error('Error caught:', error, errorInfo);
  }
  
  render() {
    if (this.state.hasError) {
      return <ErrorFallback error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

### 9.3 Notification System

```typescript
// Toast Notifications
const notify = (type, message, duration = 3000) => {
  dispatch(addNotification({
    id: generateId(),
    type, // 'success', 'error', 'warning', 'info'
    message,
    duration
  }));
};

// Usage
try {
  await workflowService.create(workflow);
  notify('success', 'Workflow created successfully');
} catch (error) {
  notify('error', error.message);
}
```

---

## 10. Performance Optimization

### 10.1 Code Splitting

```typescript
// Lazy Loading Routes
const Dashboard = lazy(() => import('./pages/Dashboard'));
const WorkflowEditor = lazy(() => import('./pages/WorkflowEditor'));
const Settings = lazy(() => import('./pages/Settings'));

// Route Configuration
<Suspense fallback={<Loading />}>
  <Routes>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/editor/:id" element={<WorkflowEditor />} />
  </Routes>
</Suspense>
```

### 10.2 Memoization

```typescript
// Prevent Unnecessary Re-renders
const WorkflowItem = memo(({ workflow, onSelect }) => {
  return (
    <Card onClick={() => onSelect(workflow)}>
      {/* Content */}
    </Card>
  );
}, (prevProps, nextProps) => {
  return prevProps.workflow.id === nextProps.workflow.id;
});
```

### 10.3 Virtual Scrolling

```typescript
// For Large Lists
import { VariableSizeList } from 'react-window';

const WorkflowList = ({ workflows }) => (
  <VariableSizeList
    height={600}
    itemCount={workflows.length}
    itemSize={index => 80}
  >
    {({ index, style }) => (
      <div style={style}>
        <WorkflowItem workflow={workflows[index]} />
      </div>
    )}
  </VariableSizeList>
);
```

### 10.4 API Response Caching

```typescript
// Redux with Reselect
import { createSelector } from 'reselect';

const selectWorkflows = (state) => state.workflows.items;
const selectWorkflowFilter = (state) => state.ui.filter;

export const selectFilteredWorkflows = createSelector(
  [selectWorkflows, selectWorkflowFilter],
  (workflows, filter) => {
    return workflows.filter(w => w.status === filter);
  }
);
```

### 10.5 Bundle Optimization

```typescript
// webpack.config.js
optimization: {
  splitChunks: {
    chunks: 'all',
    cacheGroups: {
      vendor: {
        test: /[\\/]node_modules[\\/]/,
        name: 'vendors',
        priority: 10
      },
      common: {
        minChunks: 2,
        priority: 5,
        reuseExistingChunk: true
      }
    }
  }
}
```

---

## 11. Security Considerations

### 11.1 Input Sanitization

```typescript
import DOMPurify from 'dompurify';

const sanitizedContent = DOMPurify.sanitize(userInput);
```

### 11.2 CSRF Protection

```typescript
// Include CSRF token in requests
apiClient.interceptors.request.use((config) => {
  const csrfToken = document.querySelector('meta[name="csrf-token"]')?.content;
  if (csrfToken) {
    config.headers['X-CSRF-Token'] = csrfToken;
  }
  return config;
});
```

### 11.3 Secure Storage

```typescript
// Use HttpOnly cookies for tokens (not localStorage)
// Or use secure session storage
const storeToken = (token) => {
  // Prefer: Document.cookie with HttpOnly flag (set by backend)
  // Fallback: sessionStorage (cleared on tab close)
  sessionStorage.setItem('authToken', token);
};
```

### 11.4 Content Security Policy

```typescript
// Set CSP headers
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self' 'unsafe-inline';" />
```

### 11.5 XSS Prevention

```typescript
// Use React's built-in XSS protection
// Avoid dangerouslySetInnerHTML
const safe = <div>{userContent}</div>;  // Safe

// NOT: const unsafe = <div dangerouslySetInnerHTML={{ __html: userContent }} />;
```

---

## 12. Testing Strategy

### 12.1 Unit Testing

```typescript
// Jest + React Testing Library
import { render, screen, fireEvent } from '@testing-library/react';

describe('WorkflowList', () => {
  it('should render workflow items', () => {
    render(<WorkflowList workflows={mockWorkflows} />);
    expect(screen.getByText(mockWorkflows[0].name)).toBeInTheDocument();
  });
  
  it('should handle workflow selection', () => {
    const handleSelect = jest.fn();
    render(<WorkflowList workflows={mockWorkflows} onSelect={handleSelect} />);
    fireEvent.click(screen.getByText(mockWorkflows[0].name));
    expect(handleSelect).toHaveBeenCalledWith(mockWorkflows[0]);
  });
});
```

### 12.2 Integration Testing

```typescript
// Test API integration
describe('WorkflowService', () => {
  it('should fetch workflows from API', async () => {
    const workflows = await workflowService.getAll();
    expect(workflows).toHaveLength(5);
  });
});
```

### 12.3 E2E Testing

```typescript
// Cypress tests
describe('Workflow Creation Flow', () => {
  it('should create a new workflow', () => {
    cy.visit('/dashboard');
    cy.contains('Create Workflow').click();
    cy.get('#workflow-name').type('New Workflow');
    cy.contains('Save').click();
    cy.contains('Workflow created successfully').should('be.visible');
  });
});
```

### 12.4 Test Coverage

```typescript
// Aim for coverage targets
- Statements: 80%
- Branches: 75%
- Functions: 80%
- Lines: 80%
```

---

## 13. Deployment & Build Process

### 13.1 Build Configuration

```typescript
// .env files
.env.development
.env.staging
.env.production

// Environment Variables
REACT_APP_API_URL=https://api.example.com
REACT_APP_AUTH_DOMAIN=auth.example.com
REACT_APP_AI_SERVICE_URL=https://ai.example.com
REACT_APP_LOG_LEVEL=info
```

### 13.2 Build Process

```bash
# Development
npm start

# Staging
npm run build:staging

# Production
npm run build:production
```

### 13.3 Docker Deployment

```dockerfile
# Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 13.4 CI/CD Pipeline

```yaml
# GitHub Actions
name: Deploy
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
      - run: npm ci
      - run: npm test
      - run: npm run build
      - run: npm run deploy
```

---

## File Structure

```
src/
├── components/
│   ├── common/
│   ├── forms/
│   ├── tables/
│   ├── charts/
│   └── workflow/
├── pages/
│   ├── Dashboard.tsx
│   ├── WorkflowEditor.tsx
│   ├── Settings.tsx
│   └── Profile.tsx
├── services/
│   ├── api/
│   ├── auth/
│   └── ai/
├── store/
│   ├── slices/
│   ├── selectors/
│   ├── middleware/
│   └── store.ts
├── hooks/
│   ├── useAuth.ts
│   ├── useWorkflow.ts
│   └── useNotification.ts
├── utils/
│   ├── validators.ts
│   ├── formatters.ts
│   ├── helpers.ts
│   └── constants.ts
├── theme/
│   ├── colors.ts
│   ├── typography.ts
│   └── theme.ts
├── types/
│   ├── workflow.ts
│   ├── auth.ts
│   ├── api.ts
│   └── ui.ts
├── hooks/
├── App.tsx
├── index.tsx
└── styles/
    └── global.css
```

---

## Dependencies

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.8.0",
    "@reduxjs/toolkit": "^1.9.0",
    "react-redux": "^8.1.0",
    "@mui/material": "^5.11.0",
    "@mui/icons-material": "^5.11.0",
    "reactflow": "^11.7.0",
    "axios": "^1.3.0",
    "react-hook-form": "^7.43.0",
    "yup": "^1.1.0",
    "typescript": "^4.9.0",
    "date-fns": "^2.29.0",
    "lodash": "^4.17.21"
  },
  "devDependencies": {
    "@testing-library/react": "^14.0.0",
    "@testing-library/jest-dom": "^5.16.0",
    "jest": "^29.0.0",
    "cypress": "^13.0.0",
    "eslint": "^8.30.0",
    "prettier": "^2.8.0"
  }
}
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-06 | Initial LLD for Front-End Architecture |

---

## Glossary

- **LLD:** Low-Level Design
- **React Flow:** Library for building node-based UIs
- **Redux:** State management library
- **JWT:** JSON Web Token
- **RBAC:** Role-Based Access Control
- **XSS:** Cross-Site Scripting
- **CSRF:** Cross-Site Request Forgery
- **E2E:** End-to-End Testing
