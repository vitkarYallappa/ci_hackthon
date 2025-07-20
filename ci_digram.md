# User Creation Workflow - Complete Sequence Diagrams

## 📋 **Overview**

This document provides comprehensive sequence diagrams for the user creation process in the Pharmaceutical CI Platform, covering all phases from API call to backend processing. The diagrams are designed for developers with ~2 years of experience and match the actual codebase logic.

## 🎯 **Identified Flows**

Based on codebase analysis, the following distinct flows have been identified:

1. **User Registration Flow** - New user creation with validation
2. **User Login Flow** - Authentication and JWT token generation
3. **User Profile Management Flow** - Get/update user information
4. **Error Handling Flows** - Various error scenarios
5. **Database Seeding Flow** - Initial user creation during setup
6. **Authentication Middleware Flow** - Token validation for protected endpoints

---

## 🔄 **1. User Registration Flow**

### **Main Registration Sequence**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant UserService
    participant UserRepository
    participant Security
    participant Database
    participant Validation

    Client->>AuthAPI: POST /api/v1/auth/register
    Note over Client,AuthAPI: {email, username, password, full_name}
    
    AuthAPI->>Validation: Validate UserCreate schema
    Validation-->>AuthAPI: Schema validation result
    
    alt Validation Failed
        AuthAPI-->>Client: 422 Unprocessable Entity
    else Validation Passed
        AuthAPI->>UserService: create_user(db, user_data)
        
        UserService->>UserRepository: get_by_email(db, email)
        UserRepository->>Database: SELECT * FROM users WHERE email = ?
        Database-->>UserRepository: User record (if exists)
        UserRepository-->>UserService: Existing user or None
        
        alt User Already Exists
            UserService-->>AuthAPI: ValueError("Email already registered")
            AuthAPI-->>Client: 400 Bad Request
        else User Does Not Exist
            UserService->>Security: get_password_hash(password)
            Security-->>UserService: Hashed password
            
            UserService->>UserRepository: create_user(db, user_data, hashed_password)
            UserRepository->>Database: INSERT INTO users (...)
            Database-->>UserRepository: New user record
            UserRepository-->>UserService: User object
            
            UserService-->>AuthAPI: User object
            AuthAPI->>Validation: Convert to UserResponse schema
            Validation-->>AuthAPI: UserResponse object
            AuthAPI-->>Client: 201 Created + UserResponse
        end
    end
```

### **Registration Error Scenarios**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant UserService
    participant Database

    Note over Client,Database: Scenario 1: Duplicate Email
    Client->>AuthAPI: POST /api/v1/auth/register
    AuthAPI->>UserService: create_user(db, user_data)
    UserService->>Database: Check existing email
    Database-->>UserService: User exists
    UserService-->>AuthAPI: ValueError("Email already registered")
    AuthAPI-->>Client: 400 Bad Request

    Note over Client,Database: Scenario 2: Database Error
    Client->>AuthAPI: POST /api/v1/auth/register
    AuthAPI->>UserService: create_user(db, user_data)
    UserService->>Database: INSERT operation
    Database-->>UserService: DatabaseError
    UserService-->>AuthAPI: Exception
    AuthAPI-->>Client: 500 Internal Server Error
```

---

## 🔐 **2. User Login Flow**

### **Successful Login Sequence**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant UserService
    participant UserRepository
    participant Security
    participant Database

    Client->>AuthAPI: POST /api/v1/auth/login
    Note over Client,AuthAPI: Form data: {username, password}
    
    AuthAPI->>UserService: authenticate_user(db, username, password)
    
    UserService->>UserRepository: get_by_username(db, username)
    UserRepository->>Database: SELECT * FROM users WHERE username = ?
    Database-->>UserRepository: User record or None
    UserRepository-->>UserService: User object or None
    
    alt User Not Found
        UserService-->>AuthAPI: None
        AuthAPI-->>Client: 401 Unauthorized
    else User Found
        UserService->>Security: verify_password(password, hashed_password)
        Security-->>UserService: Password verification result
        
        alt Password Incorrect
            UserService-->>AuthAPI: None
            AuthAPI-->>Client: 401 Unauthorized
        else Password Correct
            UserService->>Security: create_access_token({"sub": user.id})
            Security-->>UserService: JWT token
            
            UserService-->>AuthAPI: User object
            AuthAPI-->>Client: 200 OK + {access_token, token_type}
        end
    end
```

### **Login Error Scenarios**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant UserService
    participant Database

    Note over Client,Database: Scenario 1: Invalid Credentials
    Client->>AuthAPI: POST /api/v1/auth/login
    AuthAPI->>UserService: authenticate_user(db, username, password)
    UserService->>Database: Check user
    Database-->>UserService: User not found
    UserService-->>AuthAPI: None
    AuthAPI-->>Client: 401 Unauthorized

    Note over Client,Database: Scenario 2: Wrong Password
    Client->>AuthAPI: POST /api/v1/auth/login
    AuthAPI->>UserService: authenticate_user(db, username, password)
    UserService->>Database: Check user
    Database-->>UserService: User found
    UserService->>Security: Verify password
    Security-->>UserService: Password incorrect
    UserService-->>AuthAPI: None
    AuthAPI-->>Client: 401 Unauthorized
```

---

## 👤 **3. User Profile Management Flow**

### **Get Current User**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant Security
    participant Database
    participant UserService

    Client->>AuthAPI: GET /api/v1/auth/me
    Note over Client,AuthAPI: Authorization: Bearer <token>
    
    AuthAPI->>Security: get_current_active_user()
    Security->>Security: verify_token(token)
    Security->>Database: Decode JWT and get user_id
    Database-->>Security: User record or None
    
    alt Token Invalid
        Security-->>AuthAPI: HTTPException(401)
        AuthAPI-->>Client: 401 Unauthorized
    else Token Valid
        Security->>UserService: get_user_by_id(db, user_id)
        UserService->>Database: SELECT * FROM users WHERE id = ?
        Database-->>UserService: User record
        UserService-->>Security: User object
        
        alt User Inactive
            Security-->>AuthAPI: HTTPException(400, "Inactive user")
            AuthAPI-->>Client: 400 Bad Request
        else User Active
            Security-->>AuthAPI: User object
            AuthAPI-->>Client: 200 OK + UserResponse
        end
    end
```

### **Update Current User**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant UserService
    participant UserRepository
    participant Database
    participant Security

    Client->>AuthAPI: PUT /api/v1/auth/me
    Note over Client,AuthAPI: Authorization: Bearer <token> + UserUpdate data
    
    AuthAPI->>Security: get_current_active_user()
    Security-->>AuthAPI: Current user object
    
    AuthAPI->>UserService: update_user(db, user_id, update_data)
    UserService->>UserRepository: update_user(db, user_id, update_data)
    UserRepository->>Database: UPDATE users SET ... WHERE id = ?
    Database-->>UserRepository: Updated user record
    UserRepository-->>UserService: Updated user object
    UserService-->>AuthAPI: Updated user object
    AuthAPI-->>Client: 200 OK + UserResponse
```

---

## ⚠️ **4. Error Handling Flows**

### **Validation Error Flow**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant Validation
    participant Pydantic

    Client->>AuthAPI: POST /api/v1/auth/register
    Note over Client,AuthAPI: Invalid data (e.g., invalid email)
    
    AuthAPI->>Validation: Validate UserCreate schema
    Validation->>Pydantic: Validate fields
    Pydantic-->>Validation: ValidationError
    
    Validation-->>AuthAPI: Validation failed
    AuthAPI-->>Client: 422 Unprocessable Entity + Error details
```

### **Database Error Flow**

```mermaid
sequenceDiagram
    participant Client
    participant AuthAPI
    participant UserService
    participant Database

    Client->>AuthAPI: POST /api/v1/auth/register
    AuthAPI->>UserService: create_user(db, user_data)
    UserService->>Database: Database operation
    Database-->>UserService: DatabaseError (e.g., connection lost)
    
    UserService-->>AuthAPI: Exception
    AuthAPI-->>Client: 500 Internal Server Error
```

---

## 🌱 **5. Database Seeding Flow**

### **Initial Setup and User Creation**

```mermaid
sequenceDiagram
    participant SetupScript
    participant Database
    participant Security
    participant Alembic

    SetupScript->>Alembic: Run migrations
    Alembic->>Database: Create tables
    Database-->>Alembic: Tables created
    Alembic-->>SetupScript: Migration complete
    
    SetupScript->>Security: get_password_hash("admin123")
    Security-->>SetupScript: Hashed password
    
    SetupScript->>Database: INSERT INTO users (admin user)
    Database-->>SetupScript: Admin user created
    
    SetupScript->>Database: INSERT INTO users (test users)
    Database-->>SetupScript: Test users created
    
    SetupScript-->>SetupScript: Log completion status
```

---

## 🔒 **6. Authentication Middleware Flow**

### **Protected Endpoint Access**

```mermaid
sequenceDiagram
    participant Client
    participant ProtectedAPI
    participant Security
    participant Database
    participant BusinessLogic

    Client->>ProtectedAPI: GET /api/v1/projects/
    Note over Client,ProtectedAPI: Authorization: Bearer <token>
    
    ProtectedAPI->>Security: get_current_active_user()
    Security->>Security: verify_token(token)
    
    alt Token Invalid/Expired
        Security-->>ProtectedAPI: HTTPException(401)
        ProtectedAPI-->>Client: 401 Unauthorized
    else Token Valid
        Security->>Database: Get user by ID
        Database-->>Security: User record
        Security-->>ProtectedAPI: User object
        
        alt User Inactive
            ProtectedAPI-->>Client: 400 Bad Request
        else User Active
            ProtectedAPI->>BusinessLogic: Execute endpoint logic
            BusinessLogic-->>ProtectedAPI: Response data
            ProtectedAPI-->>Client: 200 OK + Response data
        end
    end
```

---

## 🏗️ **7. Master Flow Overview**

### **Complete User Management System**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant AuthAPI
    participant UserService
    participant Database
    participant TaskService
    participant SynthesisService

    Note over Client,SynthesisService: User Registration Path
    Client->>MainAPI: Register new user
    MainAPI->>AuthAPI: POST /auth/register
    AuthAPI->>UserService: Create user
    UserService->>Database: Store user
    Database-->>UserService: User created
    UserService-->>AuthAPI: User response
    AuthAPI-->>MainAPI: Registration complete
    MainAPI-->>Client: User registered

    Note over Client,SynthesisService: User Login Path
    Client->>MainAPI: Login user
    MainAPI->>AuthAPI: POST /auth/login
    AuthAPI->>UserService: Authenticate
    UserService->>Database: Verify credentials
    Database-->>UserService: User verified
    UserService-->>AuthAPI: JWT token
    AuthAPI-->>MainAPI: Login complete
    MainAPI-->>Client: Access token

    Note over Client,SynthesisService: Protected Resource Access
    Client->>MainAPI: Access protected resource
    MainAPI->>AuthAPI: Validate token
    AuthAPI->>UserService: Get current user
    UserService->>Database: Fetch user
    Database-->>UserService: User data
    UserService-->>AuthAPI: User validated
    AuthAPI-->>MainAPI: Access granted
    MainAPI->>TaskService: Process request
    TaskService->>SynthesisService: Execute synthesis
    SynthesisService-->>TaskService: Results
    TaskService-->>MainAPI: Processed data
    MainAPI-->>Client: Protected resource
```

---

## 📊 **8. Multi-Agent vs Single-Agent Flows**

### **Single-Agent Processing Flow**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant AuthAPI
    participant TaskService
    participant SingleAgent
    participant Database

    Client->>MainAPI: Create intelligence request
    MainAPI->>AuthAPI: Validate user
    AuthAPI-->>MainAPI: User validated
    
    MainAPI->>TaskService: Process single agent
    TaskService->>SingleAgent: Execute agent
    SingleAgent->>Database: Store results
    Database-->>SingleAgent: Results stored
    SingleAgent-->>TaskService: Agent complete
    TaskService-->>MainAPI: Processing complete
    MainAPI-->>Client: Results available
```

### **Multi-Agent Processing Flow**

```mermaid
sequenceDiagram
    participant Client
    participant MainAPI
    participant AuthAPI
    participant TaskService
    participant AgentOrchestrator
    participant Agent1
    participant Agent2
    participant Agent3
    participant SynthesisEngine
    participant Database

    Client->>MainAPI: Create multi-agent request
    MainAPI->>AuthAPI: Validate user
    AuthAPI-->>MainAPI: User validated
    
    MainAPI->>TaskService: Process multi-agent request
    TaskService->>AgentOrchestrator: Orchestrate agents
    AgentOrchestrator->>Agent1: Execute agent 1
    AgentOrchestrator->>Agent2: Execute agent 2
    AgentOrchestrator->>Agent3: Execute agent 3
    
    Agent1-->>AgentOrchestrator: Results 1
    Agent2-->>AgentOrchestrator: Results 2
    Agent3-->>AgentOrchestrator: Results 3
    
    AgentOrchestrator->>SynthesisEngine: Synthesize results
    SynthesisEngine->>Database: Store synthesis
    Database-->>SynthesisEngine: Synthesis stored
    SynthesisEngine-->>AgentOrchestrator: Synthesis complete
    AgentOrchestrator-->>TaskService: Orchestration complete
    TaskService-->>MainAPI: Multi-agent processing complete
    MainAPI-->>Client: Synthesized results available
```

---

## 🔧 **Technical Implementation Details**

### **Key Components:**

1. **Authentication Layer** (`app/core/security.py`)
   - JWT token creation and validation
   - Password hashing with bcrypt
   - User authentication middleware

2. **User Service** (`app/services/user_service.py`)
   - Business logic for user operations
   - User creation and authentication
   - Token generation

3. **User Repository** (`app/repositories/user_repository.py`)
   - Database operations for users
   - User CRUD operations
   - Email and username uniqueness checks

4. **User Model** (`app/models/user.py`)
   - SQLAlchemy user model
   - UUID primary key
   - JSON permissions field

5. **API Endpoints** (`app/api/v1/auth.py`)
   - Registration endpoint
   - Login endpoint
   - User profile endpoints

### **Database Schema:**

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    full_name VARCHAR(255),
    hashed_password VARCHAR(255) NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    is_superuser BOOLEAN DEFAULT FALSE,
    permissions JSON DEFAULT '[]',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE
);
```

### **Security Features:**

1. **Password Security**
   - bcrypt hashing with salt
   - Password verification
   - Secure password storage

2. **JWT Authentication**
   - HS256 algorithm
   - Configurable expiration
   - Token-based session management

3. **Input Validation**
   - Pydantic schema validation
   - Email format validation
   - Password strength requirements

4. **Error Handling**
   - Comprehensive error responses
   - Security exception handling
   - Database error management

---

## 🎯 **Summary**

This comprehensive workflow covers all aspects of user creation and management in the Pharmaceutical CI Platform:

- **6 distinct flows** identified and diagrammed
- **Error handling** for all scenarios
- **Security considerations** at every step
- **Database operations** clearly mapped
- **Multi-agent vs single-agent** processing differences
- **Complete authentication flow** from registration to protected resource access

The diagrams match the actual codebase implementation and provide a clear understanding for developers with ~2 years of experience. 
