# Pharmaceutical CI Platform - Project to Request to Response Flow

## Complete Workflow Sequence Diagram

```mermaid
sequenceDiagram
    participant Client as Client/Frontend
    participant MainAPI as Main API (Port 8000)
    participant Auth as Authentication Service
    participant DB as PostgreSQL Database
    participant TaskAPI as Task Service API (Port 8001)
    participant Agents as Agent System
    participant Synthesis as Synthesis Engine
    participant Storage as Storage Service

    %% Authentication Flow
    Client->>MainAPI: POST /api/v1/auth/login
    MainAPI->>Auth: Validate credentials
    Auth->>DB: Query user
    DB-->>Auth: User data
    Auth-->>MainAPI: JWT token
    MainAPI-->>Client: Authentication response

    %% Project Creation Flow
    Client->>MainAPI: POST /api/v1/projects/ (with JWT)
    MainAPI->>Auth: Validate JWT token
    Auth-->>MainAPI: User info
    MainAPI->>DB: Create project record
    DB-->>MainAPI: Project created
    MainAPI-->>Client: Project response

    %% Intelligence Request Creation Flow
    Client->>MainAPI: POST /api/v1/requests/ (with project_id)
    MainAPI->>Auth: Validate JWT token
    Auth-->>MainAPI: User info
    MainAPI->>DB: Create intelligence request
    DB-->>MainAPI: Request created
    MainAPI-->>Client: Request response

    %% Request Submission for Processing
    Client->>MainAPI: POST /api/v1/requests/{request_id}/submit
    MainAPI->>Auth: Validate JWT token
    Auth-->>MainAPI: User info
    MainAPI->>DB: Update request status to "processing"
    DB-->>MainAPI: Status updated
    MainAPI->>TaskAPI: POST /api/v1/orchestration/process
    Note over TaskAPI: Orchestrate agent processing

    %% Agent Processing Flow
    TaskAPI->>Agents: Execute Clinical Trials Agent
    Agents->>DB: Query clinical trials data
    DB-->>Agents: Clinical trials results
    Agents-->>TaskAPI: Clinical trials analysis

    TaskAPI->>Agents: Execute Market Access Agent
    Agents->>DB: Query market access data
    DB-->>Agents: Market access results
    Agents-->>TaskAPI: Market access analysis

    TaskAPI->>Agents: Execute Regulatory Agent
    Agents->>DB: Query regulatory data
    DB-->>Agents: Regulatory results
    Agents-->>TaskAPI: Regulatory analysis

    %% Synthesis and Report Generation
    TaskAPI->>Synthesis: Synthesize agent results
    Synthesis->>TaskAPI: Combined analysis
    TaskAPI->>Storage: Store synthesis results
    Storage-->>TaskAPI: Storage confirmation

    %% Report Generation
    TaskAPI->>MainAPI: POST /api/v1/reports/generate/
    MainAPI->>DB: Create generated report
    DB-->>MainAPI: Report created
    MainAPI->>Storage: Store report file
    Storage-->>MainAPI: File stored

    %% Update Request Status
    TaskAPI->>MainAPI: PUT /api/v1/requests/{request_id}
    MainAPI->>DB: Update request status to "completed"
    DB-->>MainAPI: Status updated
    MainAPI-->>TaskAPI: Update confirmation

    %% Final Response to Client
    TaskAPI-->>MainAPI: Processing complete
    MainAPI-->>Client: Request status updated
    Client->>MainAPI: GET /api/v1/requests/{request_id}
    MainAPI->>DB: Get request details
    DB-->>MainAPI: Request with results
    MainAPI-->>Client: Complete request response

    %% Report Retrieval
    Client->>MainAPI: GET /api/v1/reports/generated/{report_id}
    MainAPI->>DB: Get report metadata
    DB-->>MainAPI: Report metadata
    MainAPI->>Storage: Get report file
    Storage-->>MainAPI: Report file
    MainAPI-->>Client: Generated report

    %% Error Handling (Alternative Flow)
    Note over TaskAPI, Agents: If any agent fails
    TaskAPI->>MainAPI: PUT /api/v1/requests/{request_id}
    MainAPI->>DB: Update status to "failed"
    DB-->>MainAPI: Status updated
    MainAPI-->>Client: Error notification
```

## Key Components in the Flow

### 1. **Authentication Layer**
- JWT-based authentication
- User validation for all requests
- Session management

### 2. **Project Management**
- Create, read, update, delete projects
- Project statistics and metadata
- User-specific project isolation

### 3. **Intelligence Request Processing**
- Request creation with project association
- Status tracking (draft → processing → completed/failed)
- Agent orchestration triggers

### 4. **Agent System**
- **Clinical Trials Agent**: Analyzes clinical trial data
- **Market Access Agent**: Processes market access information
- **Regulatory Agent**: Handles regulatory compliance data
- **Synthesis Agent**: Combines all agent results

### 5. **Synthesis Engine**
- Cross-agent data correlation
- Intelligence synthesis
- Report generation

### 6. **Storage Service**
- File storage for reports
- Data persistence
- Retrieval mechanisms

### 7. **Response Generation**
- Structured report creation
- Multiple format support (PDF, JSON, etc.)
- Status tracking and notifications

## API Endpoints Used

### Main API (Port 8000)
- `POST /api/v1/auth/login` - Authentication
- `POST /api/v1/projects/` - Create project
- `POST /api/v1/requests/` - Create intelligence request
- `POST /api/v1/requests/{id}/submit` - Submit for processing
- `GET /api/v1/requests/{id}` - Get request status
- `POST /api/v1/reports/generate/` - Generate report
- `GET /api/v1/reports/generated/{id}` - Get generated report

### Task Service API (Port 8001)
- `POST /api/v1/orchestration/process` - Process request
- `GET /api/v1/agents/` - Get available agents
- `POST /api/v1/synthesis/` - Synthesize results

## Status Flow
1. **Draft** → Request created but not submitted
2. **Processing** → Submitted and being processed by agents
3. **Completed** → All agents completed, report generated
4. **Failed** → Error occurred during processing

## Error Handling
- Database connection failures
- Agent execution errors
- Storage service failures
- Authentication/authorization errors
- Network timeouts

This flow ensures a complete end-to-end processing pipeline from project creation through intelligence gathering to final report delivery. 
