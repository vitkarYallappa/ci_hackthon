# Detailed Agent Workflow Diagrams - Complete System Architecture

## 📋 **Overview**

This document provides comprehensive detailed sequence diagrams for single-agent and multi-agent workflows in the Pharmaceutical CI Platform, covering all phases from API call to backend processing across main_app, task_service, Celery, and Bedrock integration.

## 🎯 **System Architecture Components**

### **🏗️ Service Architecture:**
- **Main App** (Port 8000) - FastAPI application with authentication, project management, and API gateway
- **Task Service** (Port 8001) - Dedicated Celery microservice for task orchestration
- **Celery Workers** - Background task processing with Redis broker
- **Bedrock AI Service** - AWS Bedrock integration for AI processing
- **Database** - PostgreSQL with Alembic migrations
- **Redis** - Message broker and result backend
- **MinIO** - Object storage for files and results

---

## 🔄 **1. Single-Agent Processing Flow (Detailed)**

### **Complete Single-Agent Execution Sequence**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant AuthAPI
    participant IntelligenceService
    participant Database
    participant TaskService
    participant CeleryBroker
    participant CeleryWorker
    participant AgentRegistry
    participant SingleAgent
    participant AIService
    participant Bedrock
    participant MinIO
    participant SynthesisService

    Note over Client,SynthesisService: Phase 1: Request Creation & Authentication
    Client->>MainAPI: POST /api/v1/requests/
    Note over Client,MainAPI: {project_id, title, description, keywords, time_range, priority}
    
    MainAPI->>AuthAPI: Validate JWT token
    AuthAPI->>Database: SELECT * FROM users WHERE id = ?
    Database-->>AuthAPI: User record
    AuthAPI-->>MainAPI: User validated
    
    MainAPI->>IntelligenceService: create_intelligence_request(db, request, user_id)
    IntelligenceService->>Database: INSERT INTO intelligence_requests (...)
    Database-->>IntelligenceService: New request record
    IntelligenceService-->>MainAPI: IntelligenceRequest object
    MainAPI-->>Client: 201 Created + RequestResponse

    Note over Client,SynthesisService: Phase 2: Request Submission & Task Orchestration
    Client->>MainAPI: POST /api/v1/requests/{request_id}/submit
    MainAPI->>IntelligenceService: submit_request(db, request_id, user_id)
    IntelligenceService->>Database: UPDATE intelligence_requests SET status = 'processing'
    Database-->>IntelligenceService: Updated request
    
    IntelligenceService->>TaskService: POST /api/v1/tasks/execute-agent
    Note over IntelligenceService,TaskService: {agent_type, request_id, parameters, priority}
    
    TaskService->>CeleryBroker: execute_agent_task.delay(...)
    CeleryBroker-->>TaskService: Task ID
    TaskService-->>IntelligenceService: Task queued response
    IntelligenceService-->>MainAPI: Request submitted
    MainAPI-->>Client: 200 OK + Processing started

    Note over Client,SynthesisService: Phase 3: Celery Task Execution
    CeleryWorker->>CeleryBroker: Consume task from queue
    CeleryBroker-->>CeleryWorker: Task payload
    
    CeleryWorker->>AgentRegistry: get_agent(agent_type)
    AgentRegistry-->>CeleryWorker: Agent class
    
    CeleryWorker->>SingleAgent: Create agent instance
    SingleAgent->>CeleryWorker: Agent initialized
    
    Note over Client,SynthesisService: Phase 4: Data Acquisition
    CeleryWorker->>SingleAgent: execute(parameters)
    SingleAgent->>SingleAgent: acquire_data(request_data)
    
    alt Clinical Trials Agent
        SingleAgent->>SingleAgent: Query ClinicalTrials.gov API
        SingleAgent->>SingleAgent: Parse trial data
    else Regulatory Agent
        SingleAgent->>SingleAgent: Query FDA regulatory databases
        SingleAgent->>SingleAgent: Parse regulatory data
    else Market Access Agent
        SingleAgent->>SingleAgent: Query market data sources
        SingleAgent->>SingleAgent: Parse market data
    end
    
    SingleAgent-->>CeleryWorker: Raw data acquired

    Note over Client,SynthesisService: Phase 5: AI Processing
    CeleryWorker->>SingleAgent: process_data(raw_data)
    SingleAgent->>AIService: analyze_content(content, analysis_type, context)
    
    alt AI Service Type: Mock
        AIService->>AIService: Generate mock insights
        AIService-->>SingleAgent: Mock analysis results
    else AI Service Type: Bedrock
        AIService->>Bedrock: invoke_model(prompt)
        Note over AIService,Bedrock: {prompt, max_tokens, temperature}
        Bedrock-->>AIService: AI analysis response
        AIService->>AIService: parse_bedrock_response(response)
        AIService-->>SingleAgent: Bedrock analysis results
    end
    
    SingleAgent->>SingleAgent: generate_insights(processed_data)
    SingleAgent-->>CeleryWorker: ProcessingResult

    Note over Client,SynthesisService: Phase 6: Result Storage
    CeleryWorker->>MinIO: Store results
    MinIO-->>CeleryWorker: Storage paths
    
    CeleryWorker->>Database: INSERT INTO agent_results (...)
    Database-->>CeleryWorker: Result stored
    
    CeleryWorker->>CeleryBroker: Update task state
    CeleryBroker-->>CeleryWorker: State updated
    
    Note over Client,SynthesisService: Phase 7: Result Retrieval
    Client->>MainAPI: GET /api/v1/requests/{request_id}/status
    MainAPI->>IntelligenceService: get_request_status(db, request_id)
    IntelligenceService->>Database: SELECT * FROM intelligence_requests WHERE id = ?
    Database-->>IntelligenceService: Request with status
    IntelligenceService-->>MainAPI: Request status
    MainAPI-->>Client: 200 OK + Status response

    Client->>MainAPI: GET /api/v1/requests/{request_id}
    MainAPI->>IntelligenceService: get_request(db, request_id, user_id)
    IntelligenceService->>Database: SELECT * FROM intelligence_requests WHERE id = ?
    Database-->>IntelligenceService: Request with results
    IntelligenceService-->>MainAPI: Request with results
    MainAPI-->>Client: 200 OK + Complete request data
```

### **Single-Agent Error Handling Flow**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant TaskService
    participant CeleryWorker
    participant SingleAgent
    participant AIService
    participant Database

    Note over Client,Database: Scenario 1: Agent Execution Failure
    Client->>MainAPI: POST /api/v1/requests/{request_id}/submit
    MainAPI->>TaskService: Execute agent task
    TaskService->>CeleryWorker: Queue task
    CeleryWorker->>SingleAgent: Execute agent
    
    SingleAgent->>SingleAgent: acquire_data()
    SingleAgent-->>CeleryWorker: Exception (e.g., API timeout)
    
    CeleryWorker->>CeleryWorker: Handle exception
    CeleryWorker->>Database: UPDATE intelligence_requests SET status = 'failed'
    Database-->>CeleryWorker: Status updated
    CeleryWorker->>CeleryWorker: Retry task (max 3 attempts)
    
    alt Retry Successful
        CeleryWorker->>SingleAgent: Retry execution
        SingleAgent-->>CeleryWorker: Success
        CeleryWorker->>Database: UPDATE status = 'completed'
    else Max Retries Exceeded
        CeleryWorker->>Database: UPDATE status = 'failed'
        CeleryWorker->>Database: INSERT error details
    end

    Note over Client,Database: Scenario 2: AI Service Failure
    CeleryWorker->>SingleAgent: Execute agent
    SingleAgent->>AIService: analyze_content()
    AIService->>AIService: AI processing
    AIService-->>SingleAgent: Exception (e.g., Bedrock timeout)
    
    SingleAgent->>SingleAgent: Fallback to mock service
    SingleAgent->>AIService: Mock analysis
    AIService-->>SingleAgent: Mock results
    SingleAgent-->>CeleryWorker: ProcessingResult with fallback flag
```

---

## 🔄 **2. Multi-Agent Processing Flow (Detailed)**

### **Complete Multi-Agent Orchestration Sequence**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant AuthAPI
    participant IntelligenceService
    participant Database
    participant TaskService
    participant TaskOrchestrator
    participant CeleryBroker
    participant CeleryWorker1
    participant CeleryWorker2
    participant CeleryWorker3
    participant Agent1
    participant Agent2
    participant Agent3
    participant SynthesisEngine
    participant Correlator
    participant Analyzer
    participant Generator
    participant Bedrock
    participant MinIO
    participant SynthesisService

    Note over Client,SynthesisService: Phase 1: Multi-Agent Request Creation
    Client->>MainAPI: POST /api/v1/requests/
    Note over Client,MainAPI: {project_id, title, description, agents: ['clinical_trials', 'regulatory', 'market_access']}
    
    MainAPI->>AuthAPI: Validate JWT token
    AuthAPI-->>MainAPI: User validated
    
    MainAPI->>IntelligenceService: create_intelligence_request(db, request, user_id)
    IntelligenceService->>Database: INSERT INTO intelligence_requests (...)
    Database-->>IntelligenceService: New request record
    IntelligenceService-->>MainAPI: IntelligenceRequest object
    MainAPI-->>Client: 201 Created + RequestResponse

    Note over Client,SynthesisService: Phase 2: Multi-Agent Orchestration Initiation
    Client->>MainAPI: POST /api/v1/requests/{request_id}/submit
    MainAPI->>IntelligenceService: submit_request(db, request_id, user_id)
    IntelligenceService->>Database: UPDATE intelligence_requests SET status = 'processing'
    
    IntelligenceService->>TaskService: POST /api/v1/orchestrate
    Note over IntelligenceService,TaskService: {request_id, agent_configs, synthesis_required: true}
    
    TaskService->>TaskOrchestrator: orchestrate_multi_agent_request(...)
    TaskOrchestrator->>TaskOrchestrator: Initialize orchestration
    TaskOrchestrator-->>TaskService: Orchestration started
    TaskService-->>IntelligenceService: Orchestration response
    IntelligenceService-->>MainAPI: Multi-agent processing started
    MainAPI-->>Client: 200 OK + Processing started

    Note over Client,SynthesisService: Phase 3: Parallel Agent Execution
    TaskOrchestrator->>CeleryBroker: execute_agent_task.delay(agent1_config)
    TaskOrchestrator->>CeleryBroker: execute_agent_task.delay(agent2_config)
    TaskOrchestrator->>CeleryBroker: execute_agent_task.delay(agent3_config)
    
    CeleryBroker-->>TaskOrchestrator: Task IDs for all agents
    
    par Agent 1 Execution
        CeleryWorker1->>CeleryBroker: Consume agent1 task
        CeleryWorker1->>Agent1: Execute clinical trials agent
        Agent1->>Agent1: Query ClinicalTrials.gov
        Agent1->>Bedrock: Analyze trial data
        Bedrock-->>Agent1: AI analysis results
        Agent1-->>CeleryWorker1: Clinical trials results
        CeleryWorker1->>MinIO: Store agent1 results
        CeleryWorker1->>Database: Store agent1 results
        CeleryWorker1-->>TaskOrchestrator: Agent1 completed
    and Agent 2 Execution
        CeleryWorker2->>CeleryBroker: Consume agent2 task
        CeleryWorker2->>Agent2: Execute regulatory agent
        Agent2->>Agent2: Query FDA databases
        Agent2->>Bedrock: Analyze regulatory data
        Bedrock-->>Agent2: AI analysis results
        Agent2-->>CeleryWorker2: Regulatory results
        CeleryWorker2->>MinIO: Store agent2 results
        CeleryWorker2->>Database: Store agent2 results
        CeleryWorker2-->>TaskOrchestrator: Agent2 completed
    and Agent 3 Execution
        CeleryWorker3->>CeleryBroker: Consume agent3 task
        CeleryWorker3->>Agent3: Execute market access agent
        Agent3->>Agent3: Query market data sources
        Agent3->>Bedrock: Analyze market data
        Bedrock-->>Agent3: AI analysis results
        Agent3-->>CeleryWorker3: Market access results
        CeleryWorker3->>MinIO: Store agent3 results
        CeleryWorker3->>Database: Store agent3 results
        CeleryWorker3-->>TaskOrchestrator: Agent3 completed
    end

    Note over Client,SynthesisService: Phase 4: Agent Result Monitoring
    TaskOrchestrator->>TaskOrchestrator: _monitor_agent_tasks(agent_tasks)
    
    loop Monitor until all agents complete
        TaskOrchestrator->>CeleryBroker: Check task status for each agent
        CeleryBroker-->>TaskOrchestrator: Task status updates
        TaskOrchestrator->>TaskOrchestrator: Update progress tracking
    end
    
    TaskOrchestrator->>TaskOrchestrator: Collect all agent results
    TaskOrchestrator-->>TaskOrchestrator: All agents completed

    Note over Client,SynthesisService: Phase 5: Cross-Agent Synthesis
    TaskOrchestrator->>SynthesisEngine: synthesize_intelligence(agent_results, request_context)
    
    SynthesisEngine->>Correlator: correlate_agent_results(agent_results, request_context)
    Correlator->>Correlator: Temporal correlation analysis
    Correlator->>Correlator: Competitive correlation analysis
    Correlator->>Correlator: Regulatory correlation analysis
    Correlator->>Correlator: Market correlation analysis
    Correlator->>Correlator: Technology correlation analysis
    Correlator-->>SynthesisEngine: Cross-agent correlations
    
    SynthesisEngine->>Analyzer: analyze_patterns(agent_results, correlations, request_context)
    Analyzer->>Analyzer: Market trends analysis
    Analyzer->>Analyzer: Risk pattern detection
    Analyzer->>Analyzer: Opportunity pattern detection
    Analyzer->>Analyzer: Technology trend analysis
    Analyzer-->>SynthesisEngine: Pattern analysis results
    
    SynthesisEngine->>Generator: generate_insights(agent_results, correlations, patterns, request_context)
    Generator->>Bedrock: Generate cross-agent insights
    Bedrock-->>Generator: AI-generated insights
    Generator->>Generator: Create strategic recommendations
    Generator->>Generator: Extract key findings
    Generator-->>SynthesisEngine: Generated insights
    
    SynthesisEngine->>Generator: generate_executive_summary(insights, request_context)
    Generator->>Bedrock: Generate executive summary
    Bedrock-->>Generator: Executive summary
    Generator-->>SynthesisEngine: Executive summary
    
    SynthesisEngine->>SynthesisEngine: calculate_confidence_metrics(agent_results, insights)
    SynthesisEngine-->>TaskOrchestrator: SynthesisResult

    Note over Client,SynthesisService: Phase 6: Synthesis Storage & Integration
    TaskOrchestrator->>MinIO: Store synthesis results
    MinIO-->>TaskOrchestrator: Synthesis storage paths
    
    TaskOrchestrator->>Database: INSERT INTO synthesis_results (...)
    Database-->>TaskOrchestrator: Synthesis stored
    
    TaskOrchestrator->>Database: UPDATE intelligence_requests SET status = 'completed'
    Database-->>TaskOrchestrator: Request status updated
    
    TaskOrchestrator-->>TaskService: Orchestration completed
    TaskService-->>IntelligenceService: Multi-agent processing complete
    IntelligenceService-->>MainAPI: Processing complete
    MainAPI-->>Client: Results available

    Note over Client,SynthesisService: Phase 7: Result Retrieval & API Access
    Client->>MainAPI: GET /api/v1/requests/{request_id}
    MainAPI->>IntelligenceService: get_request(db, request_id, user_id)
    IntelligenceService->>Database: SELECT * FROM intelligence_requests WHERE id = ?
    Database-->>IntelligenceService: Request with agent results
    
    IntelligenceService->>SynthesisService: get_synthesis_results(request_id)
    SynthesisService->>Database: SELECT * FROM synthesis_results WHERE request_id = ?
    Database-->>SynthesisService: Synthesis results
    SynthesisService-->>IntelligenceService: Synthesis data
    IntelligenceService-->>MainAPI: Complete request with synthesis
    MainAPI-->>Client: 200 OK + Complete multi-agent results
```

### **Multi-Agent Synthesis API Flow**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant SynthesisService
    participant TaskService
    participant SynthesisEngine
    participant Database
    participant MinIO

    Note over Client,MinIO: Synthesis API Endpoints Flow
    Client->>MainAPI: GET /api/v1/synthesis/{request_id}/insights
    MainAPI->>SynthesisService: get_synthesis_insights(request_id)
    SynthesisService->>Database: SELECT * FROM synthesis_results WHERE request_id = ?
    Database-->>SynthesisService: Synthesis data
    SynthesisService-->>MainAPI: Cross-agent insights
    MainAPI-->>Client: 200 OK + Insights

    Client->>MainAPI: GET /api/v1/synthesis/{request_id}/executive-summary
    MainAPI->>SynthesisService: get_executive_summary(request_id)
    SynthesisService->>Database: SELECT executive_summary FROM synthesis_results WHERE request_id = ?
    Database-->>SynthesisService: Executive summary
    SynthesisService-->>MainAPI: Executive summary
    MainAPI-->>Client: 200 OK + Executive summary

    Client->>MainAPI: GET /api/v1/synthesis/{request_id}/recommendations
    MainAPI->>SynthesisService: get_strategic_recommendations(request_id)
    SynthesisService->>Database: SELECT strategic_recommendations FROM synthesis_results WHERE request_id = ?
    Database-->>SynthesisService: Strategic recommendations
    SynthesisService-->>MainAPI: Recommendations
    MainAPI-->>Client: 200 OK + Strategic recommendations

    Client->>MainAPI: GET /api/v1/synthesis/{request_id}/competitive-landscape
    MainAPI->>SynthesisService: get_competitive_landscape(request_id)
    SynthesisService->>Database: SELECT competitive_analysis FROM synthesis_results WHERE request_id = ?
    Database-->>SynthesisService: Competitive landscape data
    SynthesisService-->>MainAPI: Competitive landscape
    MainAPI-->>Client: 200 OK + Competitive landscape
```

---

## 🔄 **3. Bedrock AI Integration Flow (Detailed)**

### **Complete Bedrock AI Processing Sequence**

```mermaid
sequenceDiagram
    participant Agent
    participant AIService
    participant BedrockService
    participant BedrockAPI
    participant AWS
    participant FallbackService

    Note over Agent,FallbackService: Bedrock AI Service Integration
    Agent->>AIService: analyze_content(content, analysis_type, context)
    
    AIService->>AIService: get_ai_service() - Factory pattern
    AIService->>BedrockService: BedrockAIService()
    
    BedrockService->>BedrockService: _build_analysis_prompt(content, analysis_type, context)
    Note over BedrockService: Construct pharmaceutical-specific prompt
    
    BedrockService->>BedrockAPI: invoke_model(modelId, body)
    Note over BedrockService,BedrockAPI: {prompt, max_tokens, temperature, messages}
    
    BedrockAPI->>AWS: AWS Bedrock API call
    AWS-->>BedrockAPI: Model response
    
    alt Successful Response
        BedrockAPI-->>BedrockService: Response body
        BedrockService->>BedrockService: _parse_bedrock_response(response_body)
        BedrockService->>BedrockService: _calculate_confidence(response_body)
        BedrockService-->>AIService: Parsed insights with confidence
        AIService-->>Agent: Analysis results
    else ThrottlingException
        BedrockAPI-->>BedrockService: ThrottlingException
        BedrockService->>BedrockService: Exponential backoff
        BedrockService->>BedrockAPI: Retry with backoff
        BedrockAPI-->>BedrockService: Success on retry
        BedrockService-->>AIService: Analysis results
        AIService-->>Agent: Analysis results
    else ModelNotReadyException
        BedrockAPI-->>BedrockService: ModelNotReadyException
        BedrockService->>FallbackService: _fallback_to_mock(content, analysis_type, context)
        FallbackService-->>BedrockService: Mock analysis results
        BedrockService-->>AIService: Fallback results
        AIService-->>Agent: Fallback analysis results
    else Other Exception
        BedrockAPI-->>BedrockService: ClientError
        BedrockService->>FallbackService: _fallback_to_mock(content, analysis_type, context)
        FallbackService-->>BedrockService: Mock analysis results
        BedrockService-->>AIService: Fallback results
        AIService-->>Agent: Fallback analysis results
    end
```

### **Bedrock Prompt Construction Flow**

```mermaid
sequenceDiagram
    participant Agent
    participant BedrockService
    participant PromptBuilder

    Note over Agent,PromptBuilder: Pharmaceutical-Specific Prompt Construction
    Agent->>BedrockService: analyze_content(content, analysis_type, context)
    
    BedrockService->>PromptBuilder: _construct_pharmaceutical_prompt(content, analysis_type, context)
    
    PromptBuilder->>PromptBuilder: Extract keywords from context
    PromptBuilder->>PromptBuilder: Extract time range from context
    PromptBuilder->>PromptBuilder: Build domain-specific instructions
    
    alt Analysis Type: Clinical Trials
        PromptBuilder->>PromptBuilder: Add clinical trial analysis instructions
        PromptBuilder->>PromptBuilder: Include trial phase analysis
        PromptBuilder->>PromptBuilder: Add safety profile analysis
    else Analysis Type: Regulatory
        PromptBuilder->>PromptBuilder: Add regulatory analysis instructions
        PromptBuilder->>PromptBuilder: Include compliance assessment
        PromptBuilder->>PromptBuilder: Add approval timeline analysis
    else Analysis Type: Market Access
        PromptBuilder->>PromptBuilder: Add market access analysis instructions
        PromptBuilder->>PromptBuilder: Include pricing analysis
        PromptBuilder->>PromptBuilder: Add reimbursement assessment
    end
    
    PromptBuilder->>PromptBuilder: Construct final prompt
    PromptBuilder-->>BedrockService: Complete pharmaceutical prompt
    BedrockService->>BedrockAPI: invoke_model with constructed prompt
```

---

## 🔄 **4. Celery Task Management Flow (Detailed)**

### **Complete Celery Task Lifecycle**

```mermaid
sequenceDiagram
    participant TaskService
    participant CeleryApp
    participant CeleryBroker
    participant CeleryWorker
    participant TaskMonitor
    participant Database
    participant Redis

    Note over TaskService,Redis: Celery Task Lifecycle Management
    TaskService->>CeleryApp: execute_agent_task.delay(agent_type, request_id, parameters)
    
    CeleryApp->>CeleryBroker: Send task to queue
    Note over CeleryApp,CeleryBroker: {task_id, agent_type, request_id, parameters, priority}
    CeleryBroker->>Redis: Store task in Redis queue
    Redis-->>CeleryBroker: Task stored
    CeleryBroker-->>CeleryApp: Task queued
    CeleryApp-->>TaskService: Task ID returned
    
    CeleryWorker->>CeleryBroker: Consume task from queue
    CeleryBroker->>Redis: Get task from queue
    Redis-->>CeleryBroker: Task payload
    CeleryBroker-->>CeleryWorker: Task received
    
    CeleryWorker->>CeleryWorker: Update task state to 'PROGRESS'
    CeleryWorker->>Database: UPDATE task_status SET status = 'running'
    Database-->>CeleryWorker: Status updated
    
    Note over TaskService,Redis: Task Execution with Progress Updates
    loop Progress Updates
        CeleryWorker->>CeleryWorker: Execute agent processing
        CeleryWorker->>CeleryBroker: update_state(state='PROGRESS', meta={progress})
        CeleryBroker->>Redis: Store progress update
        Redis-->>CeleryBroker: Progress stored
    end
    
    CeleryWorker->>CeleryWorker: Complete task execution
    CeleryWorker->>CeleryBroker: update_state(state='SUCCESS', meta={result})
    CeleryBroker->>Redis: Store final result
    Redis-->>CeleryBroker: Result stored
    
    CeleryWorker->>Database: UPDATE task_status SET status = 'completed'
    Database-->>CeleryWorker: Status updated
    
    Note over TaskService,Redis: Task Monitoring & Status Retrieval
    TaskMonitor->>CeleryBroker: Check task status
    CeleryBroker->>Redis: Get task result
    Redis-->>CeleryBroker: Task result
    CeleryBroker-->>TaskMonitor: Task status and result
    
    TaskMonitor->>Database: UPDATE intelligence_requests SET status = 'completed'
    Database-->>TaskMonitor: Request status updated
```

### **Celery Error Handling & Retry Flow**

```mermaid
sequenceDiagram
    participant CeleryWorker
    participant Agent
    participant AIService
    participant CeleryBroker
    participant Database
    participant Redis

    Note over CeleryWorker,Redis: Error Handling & Retry Logic
    CeleryWorker->>Agent: Execute agent
    Agent->>AIService: Process with AI
    AIService-->>Agent: Exception (e.g., API timeout)
    Agent-->>CeleryWorker: Exception propagated
    
    CeleryWorker->>CeleryWorker: Handle exception
    CeleryWorker->>CeleryBroker: update_state(state='FAILURE', meta={error})
    CeleryBroker->>Redis: Store error state
    Redis-->>CeleryBroker: Error stored
    
    CeleryWorker->>CeleryWorker: Check retry count
    alt Retry Count < Max Retries
        CeleryWorker->>CeleryWorker: Calculate retry delay (exponential backoff)
        CeleryWorker->>CeleryBroker: Retry task with delay
        CeleryBroker->>Redis: Schedule retry
        Redis-->>CeleryBroker: Retry scheduled
        
        Note over CeleryWorker,Redis: Retry Execution
        CeleryWorker->>Agent: Retry execution
        Agent->>AIService: Retry AI processing
        AIService-->>Agent: Success on retry
        Agent-->>CeleryWorker: Success
        CeleryWorker->>CeleryBroker: update_state(state='SUCCESS')
        CeleryBroker->>Redis: Store success
    else Max Retries Exceeded
        CeleryWorker->>Database: UPDATE task_status SET status = 'failed'
        Database-->>CeleryWorker: Status updated
        CeleryWorker->>Database: INSERT error details
        Database-->>CeleryWorker: Error logged
        CeleryWorker->>CeleryBroker: Final failure state
        CeleryBroker->>Redis: Store final failure
    end
```

---

## 🔄 **5. Database Integration Flow (Detailed)**

### **Complete Database Operations Sequence**

```mermaid
sequenceDiagram
    participant MainAPI
    participant IntelligenceService
    participant UserRepository
    participant RequestRepository
    participant AgentResultRepository
    participant SynthesisRepository
    participant Database
    participant Alembic

    Note over MainAPI,Alembic: Database Operations Flow
    MainAPI->>IntelligenceService: create_intelligence_request(db, request, user_id)
    
    IntelligenceService->>UserRepository: get_user_by_id(db, user_id)
    UserRepository->>Database: SELECT * FROM users WHERE id = ?
    Database-->>UserRepository: User record
    UserRepository-->>IntelligenceService: User object
    
    IntelligenceService->>RequestRepository: create_request(db, request_data)
    RequestRepository->>Database: INSERT INTO intelligence_requests (...)
    Database-->>RequestRepository: New request ID
    RequestRepository-->>IntelligenceService: Request object
    
    Note over MainAPI,Alembic: Agent Result Storage
    IntelligenceService->>AgentResultRepository: create_agent_result(db, result_data)
    AgentResultRepository->>Database: INSERT INTO agent_results (...)
    Database-->>AgentResultRepository: Result stored
    AgentResultRepository-->>IntelligenceService: Agent result object
    
    Note over MainAPI,Alembic: Synthesis Result Storage
    IntelligenceService->>SynthesisRepository: create_synthesis_result(db, synthesis_data)
    SynthesisRepository->>Database: INSERT INTO synthesis_results (...)
    Database-->>SynthesisRepository: Synthesis stored
    SynthesisRepository-->>IntelligenceService: Synthesis result object
    
    Note over MainAPI,Alembic: Result Retrieval
    IntelligenceService->>RequestRepository: get_request(db, request_id, user_id)
    RequestRepository->>Database: SELECT * FROM intelligence_requests WHERE id = ?
    Database-->>RequestRepository: Request record
    RequestRepository-->>IntelligenceService: Request object
    
    IntelligenceService->>AgentResultRepository: get_agent_results(db, request_id)
    AgentResultRepository->>Database: SELECT * FROM agent_results WHERE request_id = ?
    Database-->>AgentResultRepository: Agent results
    AgentResultRepository-->>IntelligenceService: Agent results list
    
    IntelligenceService->>SynthesisRepository: get_synthesis_result(db, request_id)
    SynthesisRepository->>Database: SELECT * FROM synthesis_results WHERE request_id = ?
    Database-->>SynthesisRepository: Synthesis result
    SynthesisRepository-->>IntelligenceService: Synthesis result object
```

### **Database Migration Flow**

```mermaid
sequenceDiagram
    participant Alembic
    participant Database
    participant MigrationScript

    Note over Alembic,Database: Database Migration Process
    Alembic->>Database: Check current migration version
    Database-->>Alembic: Current version
    
    Alembic->>MigrationScript: Load migration scripts
    MigrationScript-->>Alembic: Available migrations
    
    loop Apply pending migrations
        Alembic->>MigrationScript: Get next migration
        MigrationScript-->>Alembic: Migration script
        
        Alembic->>Database: Execute migration
        Note over Alembic,Database: CREATE TABLE, ALTER TABLE, etc.
        Database-->>Alembic: Migration applied
        
        Alembic->>Database: Update migration version
        Database-->>Alembic: Version updated
    end
    
    Alembic->>Database: Verify schema consistency
    Database-->>Alembic: Schema verification complete
```

---

## 🔄 **6. Microservice Communication Flow (Detailed)**

### **Main App ↔ Task Service Communication**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant IntelligenceService
    participant HTTPXClient
    participant TaskService
    participant TaskOrchestrator
    participant CeleryApp

    Note over Client,CeleryApp: Microservice Communication Flow
    Client->>MainAPI: POST /api/v1/requests/{request_id}/submit
    
    MainAPI->>IntelligenceService: submit_request(db, request_id, user_id)
    IntelligenceService->>Database: UPDATE intelligence_requests SET status = 'processing'
    Database-->>IntelligenceService: Status updated
    
    IntelligenceService->>HTTPXClient: POST /api/v1/tasks/execute-agent
    Note over IntelligenceService,HTTPXClient: {agent_type, request_id, parameters, priority}
    
    HTTPXClient->>TaskService: HTTP request to task service
    TaskService->>TaskOrchestrator: Process task request
    TaskOrchestrator->>CeleryApp: Queue task
    CeleryApp-->>TaskOrchestrator: Task ID
    TaskOrchestrator-->>TaskService: Task queued response
    TaskService-->>HTTPXClient: HTTP response with task ID
    HTTPXClient-->>IntelligenceService: Task service response
    IntelligenceService-->>MainAPI: Processing started
    MainAPI-->>Client: 200 OK + Task queued

    Note over Client,CeleryApp: Status Checking Flow
    Client->>MainAPI: GET /api/v1/requests/{request_id}/status
    MainAPI->>IntelligenceService: get_request_status(db, request_id)
    IntelligenceService->>HTTPXClient: GET /api/v1/tasks/status/{task_id}
    
    HTTPXClient->>TaskService: HTTP request for task status
    TaskService->>TaskOrchestrator: Get task status
    TaskOrchestrator->>CeleryApp: Check task status
    CeleryApp-->>TaskOrchestrator: Task status
    TaskOrchestrator-->>TaskService: Task status
    TaskService-->>HTTPXClient: HTTP response with status
    HTTPXClient-->>IntelligenceService: Task status
    IntelligenceService-->>MainAPI: Request status
    MainAPI-->>Client: 200 OK + Status response
```

### **Synthesis Service Communication**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant SynthesisService
    participant HTTPXClient
    participant TaskService
    participant SynthesisEngine
    participant Database

    Note over Client,Database: Synthesis Service Communication
    Client->>MainAPI: POST /api/v1/synthesis/execute
    Note over Client,MainAPI: {request_id, agent_results, request_context}
    
    MainAPI->>SynthesisService: execute_synthesis(request_id, agent_results, request_context)
    SynthesisService->>Database: Save synthesis request
    Database-->>SynthesisService: Request saved
    
    SynthesisService->>HTTPXClient: POST /api/v1/synthesis/synthesize
    Note over SynthesisService,HTTPXClient: {request_id, agent_results, request_context}
    
    HTTPXClient->>TaskService: HTTP request to task service
    TaskService->>SynthesisEngine: synthesize_intelligence(agent_results, request_context)
    SynthesisEngine->>SynthesisEngine: Process synthesis
    SynthesisEngine-->>TaskService: Synthesis result
    TaskService-->>HTTPXClient: HTTP response with synthesis
    HTTPXClient-->>SynthesisService: Synthesis result
    SynthesisService->>Database: Save synthesis result
    Database-->>SynthesisService: Result saved
    SynthesisService-->>MainAPI: Synthesis complete
    MainAPI-->>Client: 200 OK + Synthesis result
```

---

## 🔄 **7. Complete System Architecture Overview**

### **Master System Flow**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant AuthAPI
    participant IntelligenceService
    participant SynthesisService
    participant TaskService
    participant TaskOrchestrator
    participant CeleryWorkers
    participant Agents
    participant AIService
    participant Bedrock
    participant Database
    participant Redis
    participant MinIO

    Note over Client,MinIO: Complete System Architecture Flow
    
    Note over Client,MinIO: User Authentication & Request Creation
    Client->>MainAPI: Authenticate & Create Request
    MainAPI->>AuthAPI: Validate user
    AuthAPI->>Database: Verify user
    Database-->>AuthAPI: User verified
    AuthAPI-->>MainAPI: User validated
    MainAPI->>IntelligenceService: Create intelligence request
    IntelligenceService->>Database: Store request
    Database-->>IntelligenceService: Request stored
    IntelligenceService-->>MainAPI: Request created
    MainAPI-->>Client: Request created

    Note over Client,MinIO: Single-Agent Processing Path
    Client->>MainAPI: Submit single agent request
    MainAPI->>TaskService: Execute single agent
    TaskService->>CeleryWorkers: Queue single agent task
    CeleryWorkers->>Agents: Execute agent
    Agents->>AIService: Process with AI
    AIService->>Bedrock: AI analysis
    Bedrock-->>AIService: AI results
    AIService-->>Agents: Processed results
    Agents-->>CeleryWorkers: Agent results
    CeleryWorkers->>MinIO: Store results
    CeleryWorkers->>Database: Store results
    CeleryWorkers-->>TaskService: Task complete
    TaskService-->>MainAPI: Processing complete
    MainAPI-->>Client: Single agent results

    Note over Client,MinIO: Multi-Agent Processing Path
    Client->>MainAPI: Submit multi-agent request
    MainAPI->>TaskService: Execute multi-agent orchestration
    TaskService->>TaskOrchestrator: Orchestrate agents
    TaskOrchestrator->>CeleryWorkers: Queue multiple agent tasks
    
    par Parallel Agent Execution
        CeleryWorkers->>Agents: Execute agent 1
        Agents->>AIService: Process agent 1
        AIService->>Bedrock: AI analysis for agent 1
        Bedrock-->>AIService: AI results for agent 1
        AIService-->>Agents: Agent 1 results
        Agents-->>CeleryWorkers: Agent 1 complete
    and Parallel Agent Execution
        CeleryWorkers->>Agents: Execute agent 2
        Agents->>AIService: Process agent 2
        AIService->>Bedrock: AI analysis for agent 2
        Bedrock-->>AIService: AI results for agent 2
        AIService-->>Agents: Agent 2 results
        Agents-->>CeleryWorkers: Agent 2 complete
    and Parallel Agent Execution
        CeleryWorkers->>Agents: Execute agent 3
        Agents->>AIService: Process agent 3
        AIService->>Bedrock: AI analysis for agent 3
        Bedrock-->>AIService: AI results for agent 3
        AIService-->>Agents: Agent 3 results
        Agents-->>CeleryWorkers: Agent 3 complete
    end
    
    TaskOrchestrator->>SynthesisEngine: Synthesize all agent results
    SynthesisEngine->>Bedrock: Cross-agent analysis
    Bedrock-->>SynthesisEngine: Synthesis results
    SynthesisEngine-->>TaskOrchestrator: Synthesis complete
    TaskOrchestrator->>MinIO: Store synthesis
    TaskOrchestrator->>Database: Store synthesis
    TaskOrchestrator-->>TaskService: Orchestration complete
    TaskService-->>MainAPI: Multi-agent processing complete
    MainAPI-->>Client: Synthesized results

    Note over Client,MinIO: Result Retrieval & API Access
    Client->>MainAPI: Retrieve results and insights
    MainAPI->>IntelligenceService: Get request results
    IntelligenceService->>Database: Retrieve agent results
    Database-->>IntelligenceService: Agent results
    IntelligenceService->>SynthesisService: Get synthesis results
    SynthesisService->>Database: Retrieve synthesis
    Database-->>SynthesisService: Synthesis results
    SynthesisService-->>IntelligenceService: Synthesis data
    IntelligenceService-->>MainAPI: Complete results
    MainAPI-->>Client: Complete intelligence results
```

---

## 🔧 **Technical Implementation Details**

### **Key Components & Files:**

#### **Main App (Port 8000):**
- `main_app/main.py` - FastAPI application with CORS and middleware
- `main_app/app/api/v1/requests.py` - Intelligence request endpoints
- `main_app/app/api/v1/synthesis.py` - Synthesis API endpoints
- `main_app/app/services/intelligence_service.py` - Business logic
- `main_app/app/services/synthesis_service.py` - Synthesis business logic
- `main_app/app/repositories/` - Database repositories
- `main_app/app/core/security.py` - JWT authentication
- `main_app/app/core/database.py` - Database connection
- `main_app/app/core/config.py` - Configuration settings

#### **Task Service (Port 8001):**
- `task_service/main.py` - Celery task service application
- `task_service/app/tasks/agent_tasks.py` - Agent execution tasks
- `task_service/app/tasks/synthesis_tasks.py` - Synthesis tasks
- `task_service/app/services/task_orchestration.py` - Task orchestration
- `task_service/app/services/synthesis/engine.py` - Synthesis engine
- `task_service/app/services/synthesis/correlator.py` - Cross-agent correlation
- `task_service/app/services/synthesis/analyzer.py` - Pattern analysis
- `task_service/app/services/synthesis/generator.py` - Report generation
- `task_service/app/api/v1/tasks.py` - Task management endpoints
- `task_service/app/api/v1/orchestration.py` - Orchestration endpoints
- `task_service/app/api/v1/synthesis.py` - Synthesis endpoints

#### **Celery Configuration:**
- `task_service/app/core/celery_app.py` - Celery application setup
- `task_service/app/core/config.py` - Task service configuration
- Redis as message broker and result backend
- Task routing and queue management

#### **AI Service Integration:**
- `main_app/app/services/ai_services/ai_service.py` - AI service abstraction
- `main_app/app/services/ai_services/ai_bedrock.py` - AWS Bedrock implementation
- `main_app/app/services/ai_services/ai_mock.py` - Mock AI service
- Factory pattern for AI service selection

#### **Database Schema:**
- `main_app/alembic/versions/` - Database migrations
- `main_app/app/models/` - SQLAlchemy models
- PostgreSQL with UUID primary keys
- JSON fields for flexible data storage

### **Configuration Settings:**

#### **Environment Variables:**
```bash
# Main App
DATABASE_URL=postgresql://postgres:password@localhost:5432/pharma_ci
REDIS_URL=redis://localhost:6379/0
TASK_SERVICE_URL=http://localhost:8001
AI_SERVICE_TYPE=mock  # or bedrock
BEDROCK_MODEL_ID=anthropic.claude-3-sonnet-20240229-v1:0

# Task Service
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
MAIN_API_URL=http://localhost:8000
```

#### **Celery Task Routes:**
```python
task_routes={
    'app.tasks.agent_tasks.execute_agent_task': {'queue': 'agent_processing'},
    'app.tasks.agent_tasks.execute_multi_agent_task': {'queue': 'multi_agent'},
    'app.tasks.synthesis_tasks.synthesize_intelligence_task': {'queue': 'synthesis'},
    'app.tasks.synthesis_tasks.correlate_agent_results_task': {'queue': 'synthesis'},
    'app.tasks.synthesis_tasks.generate_executive_summary_task': {'queue': 'synthesis'},
}
```

### **Security Features:**
- JWT-based authentication
- Password hashing with bcrypt
- Role-based access control
- Input validation with Pydantic
- CORS middleware configuration

### **Error Handling:**
- Comprehensive exception handling
- Retry logic with exponential backoff
- Fallback mechanisms for AI services
- Database transaction management
- Task failure recovery

### **Performance Optimizations:**
- Database connection pooling
- Redis caching
- Async/await patterns
- Task queue optimization
- Parallel agent execution

---

## 🎯 **Summary**

This comprehensive detailed workflow covers:

- **Complete single-agent processing** from API call to result storage
- **Advanced multi-agent orchestration** with synthesis capabilities
- **AWS Bedrock AI integration** with fallback mechanisms
- **Celery task management** with monitoring and retry logic
- **Microservice communication** between main app and task service
- **Database operations** with proper transaction management
- **Error handling** for all failure scenarios
- **Security considerations** at every layer

The diagrams provide complete technical details for developers with ~2 years of experience, covering all files in main_app, task_service, Celery, and Bedrock integration with accurate implementation details matching the actual codebase. 
