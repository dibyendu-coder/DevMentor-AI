# Design Document: DevMentor AI

## Overview

DevMentor AI is a full-stack AI-powered learning and developer productivity platform built with Next.js (frontend), FastAPI (backend), PostgreSQL (database), and integrated with multiple LLM providers. The system employs a multi-agent AI architecture with dynamic mode switching, RAG-enhanced context retrieval, and comprehensive memory systems for personalized learning experiences.

### Key Design Principles

1. **Modularity**: Clear separation between frontend, backend services, and AI orchestration
2. **Scalability**: Stateless API design with connection pooling and caching
3. **Personalization**: Multi-layered memory system (conversation, learning, project) with vector embeddings
4. **Performance**: Streaming responses, code splitting, lazy loading, and optimized queries
5. **Security**: JWT authentication, input sanitization, encrypted secrets, rate limiting
6. **User Experience**: Real-time streaming, syntax highlighting, one-click actions, responsive design

## Architecture

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Frontend Layer                          │
│  Next.js 14 (App Router) + TypeScript + Tailwind + ShadCN UI   │
│                                                                 │
│  Pages: Landing, Auth, Dashboard, Chat, Project Builder,       │
│         Learning Planner, Analytics                             │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS/REST API
                         │ WebSocket (Streaming)
┌────────────────────────▼────────────────────────────────────────┐
│                        Backend Layer                            │
│              FastAPI + Python 3.11+ + Pydantic                  │
│                                                                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐  │
│  │   Auth   │ │    AI    │ │   RAG    │ │  Project Builder │  │
│  │ Service  │ │ Service  │ │ Service  │ │     Engine       │  │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Analytics Service                           │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
┌───────▼──────┐  ┌──────▼──────┐  ┌─────▼──────┐
│  PostgreSQL  │  │  Vector DB  │  │  LLM APIs  │
│   Database   │  │ FAISS/Pine  │  │ OpenAI/    │
│              │  │    cone     │  │ Gemini/    │
│              │  │             │  │ Claude     │
└──────────────┘  └─────────────┘  └────────────┘
```

### Request Flow

1. **User Request** → Frontend (Next.js)
2. **API Call** → Backend (FastAPI) with JWT authentication
3. **Service Routing** → Appropriate service (Auth/AI/RAG/Analytics)
4. **AI Processing** → AI Service orchestrates LLM with RAG context
5. **Vector Search** → RAG Service queries Vector DB for relevant context
6. **LLM Call** → External LLM API with enriched prompt
7. **Streaming Response** → Tokens streamed back through WebSocket
8. **Memory Storage** → Interaction stored in appropriate memory layer
9. **Frontend Update** → UI updates in real-time with streamed content


## Components and Interfaces

### Frontend Components

#### 1. Authentication Module

**Components:**
- `LoginPage`: Email/password form with validation
- `RegisterPage`: Registration form with password strength indicator
- `AuthProvider`: React Context for authentication state management
- `ProtectedRoute`: HOC for route protection

**State Management:**
```typescript
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}
```

**API Integration:**
- `POST /api/auth/register` - User registration
- `POST /api/auth/login` - User login
- `POST /api/auth/refresh` - Token refresh
- `GET /api/auth/me` - Get current user

#### 2. Chat Interface Module

**Components:**
- `ChatContainer`: Main chat layout with sidebar and message area
- `MessageList`: Scrollable message history with virtualization
- `MessageBubble`: Individual message with role-based styling
- `CodeBlock`: Syntax-highlighted code with copy button
- `InputBar`: Message input with send button and file upload
- `StreamingIndicator`: Animated typing indicator
- `ModeIndicator`: Visual indicator of current AI mode

**State Management:**
```typescript
interface ChatState {
  messages: Message[];
  isStreaming: boolean;
  currentMode: AIMode;
  sessionId: string;
}

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  metadata?: {
    codeBlocks?: CodeBlock[];
    mode?: AIMode;
  };
}
```

**WebSocket Integration:**
- Connect to `ws://backend/api/chat/stream`
- Send: `{ sessionId, message, mode? }`
- Receive: `{ type: 'token' | 'complete' | 'error', data }`

#### 3. Project Builder Module

**Components:**
- `ProjectBuilderWizard`: Multi-step form for project creation
- `IdeaClarification`: Step 1 - Capture project idea and answer questions
- `TechStackSelector`: Step 2 - Review and customize suggested tech stack
- `ArchitectureViewer`: Step 3 - Display architecture diagram
- `FolderStructureTree`: Step 4 - Interactive folder tree view
- `CodePreview`: Step 5 - Tabbed view of generated boilerplate
- `RoadmapTimeline`: Step 6 - Visual roadmap with steps
- `ExportOptions`: Export project plan as ZIP or PDF

**State Management:**
```typescript
interface ProjectBuilderState {
  step: number;
  projectIdea: string;
  clarifications: Record<string, string>;
  techStack: TechStack;
  architecture: string;
  folderStructure: FolderNode[];
  boilerplateCode: Record<string, string>;
  roadmap: RoadmapStep[];
}
```

#### 4. Dashboard Module

**Components:**
- `DashboardLayout`: Grid layout for stat cards and charts
- `StatCard`: Display metric with icon and trend
- `ProgressChart`: Line/bar chart for skill progression
- `SessionHistory`: List of recent sessions with filters
- `SkillRadar`: Radar chart for skill proficiency
- `QuickActions`: Buttons for common actions

**Data Fetching:**
- `GET /api/analytics/overview` - Dashboard summary stats
- `GET /api/analytics/progress` - Time-series progress data
- `GET /api/analytics/skills` - Skill proficiency data
- `GET /api/sessions` - Session history

#### 5. Learning Planner Module

**Components:**
- `LearningPlanForm`: Form to specify goals and availability
- `DailyPlanView`: Calendar view of daily study tasks
- `SkillRoadmap`: Multi-week roadmap with milestones
- `TaskCheckbox`: Checkable task items with completion tracking
- `ExportButton`: Export plan as PDF

### Backend Services

#### 1. Auth Service

**Responsibilities:**
- User registration and login
- Password hashing and verification
- JWT token generation and validation
- Session management

**API Endpoints:**
```python
POST   /api/auth/register
POST   /api/auth/login
POST   /api/auth/refresh
GET    /api/auth/me
POST   /api/auth/logout
```

**Core Functions:**
```python
def hash_password(password: str) -> str:
    """Hash password using bcrypt with 12 rounds"""
    
def verify_password(password: str, hash: str) -> bool:
    """Verify password against hash"""
    
def create_jwt_token(user_id: str) -> str:
    """Generate JWT with 24h expiration"""
    
def verify_jwt_token(token: str) -> dict:
    """Verify and decode JWT token"""
```

**Database Models:**
```python
class User(Base):
    id: UUID
    email: str (unique, indexed)
    password_hash: str
    created_at: datetime
    profile_data: JSON
```

#### 2. AI Service

**Responsibilities:**
- AI agent mode detection and switching
- LLM API orchestration
- Prompt engineering for each mode
- Response streaming
- Context injection from RAG

**API Endpoints:**
```python
POST   /api/chat/message
WS     /api/chat/stream
POST   /api/chat/detect-mode
GET    /api/chat/session/{session_id}
```

**Core Classes:**
```python
class AIOrchestrator:
    def detect_mode(self, message: str, history: list) -> AIMode
    def build_prompt(self, mode: AIMode, message: str, context: list) -> str
    async def stream_response(self, prompt: str) -> AsyncIterator[str]
    
class AIMode(Enum):
    TUTOR = "tutor"
    EXPLAINER = "explainer"
    DEBUGGER = "debugger"
    PROJECT_BUILDER = "project_builder"
    LEARNING_PLANNER = "learning_planner"
```

**Mode Detection Logic:**
```python
def detect_mode(message: str, history: list) -> AIMode:
    # Check for explicit mode keywords
    if any(kw in message.lower() for kw in ["explain", "what is", "how does"]):
        if contains_code(message):
            return AIMode.EXPLAINER
        return AIMode.TUTOR
    
    if any(kw in message.lower() for kw in ["bug", "error", "fix", "debug"]):
        return AIMode.DEBUGGER
    
    if any(kw in message.lower() for kw in ["build", "create project", "i want to make"]):
        return AIMode.PROJECT_BUILDER
    
    if any(kw in message.lower() for kw in ["learn", "study plan", "roadmap"]):
        return AIMode.LEARNING_PLANNER
    
    # Default to tutor for conceptual questions
    return AIMode.TUTOR
```

**Prompt Templates:**
```python
TUTOR_PROMPT = """You are an expert programming tutor. Explain concepts clearly with:
1. Simple introduction
2. Core concepts broken down
3. Practical examples with code
4. Common pitfalls

User's learning history: {learning_context}
User's question: {question}"""

EXPLAINER_PROMPT = """You are a code analysis expert. Analyze this code:
1. Identify the language and purpose
2. Explain the logic step-by-step
3. Highlight key patterns and techniques
4. Note any best practices or issues

Code: {code}"""

DEBUGGER_PROMPT = """You are a debugging expert. Analyze this code for issues:
1. Identify syntax errors
2. Find logical errors
3. Detect anti-patterns
4. Suggest specific fixes with line numbers

Code: {code}
Error message (if any): {error}"""

PROJECT_BUILDER_PROMPT = """You are a software architect. Help build this project:
Phase: {phase}
Project idea: {idea}
Previous context: {context}

Provide: {phase_instructions}"""
```

#### 3. RAG Service

**Responsibilities:**
- Vector embedding generation
- Semantic search across memory layers
- Context retrieval and ranking
- Memory storage and indexing

**API Endpoints:**
```python
POST   /api/rag/embed
POST   /api/rag/search
POST   /api/rag/store
GET    /api/rag/context/{user_id}
```

**Core Classes:**
```python
class RAGService:
    def __init__(self, vector_db: VectorDB, embedding_model: EmbeddingModel):
        self.vector_db = vector_db
        self.embedding_model = embedding_model
    
    async def generate_embedding(self, text: str) -> list[float]:
        """Generate embedding vector for text"""
        
    async def search_similar(
        self, 
        query: str, 
        top_k: int = 5,
        threshold: float = 0.7
    ) -> list[SearchResult]:
        """Search for semantically similar content"""
        
    async def store_memory(
        self,
        user_id: str,
        content: str,
        memory_type: MemoryType,
        metadata: dict
    ):
        """Store content with embedding in vector DB"""
        
    async def get_context(
        self,
        user_id: str,
        query: str,
        memory_types: list[MemoryType]
    ) -> list[ContextItem]:
        """Retrieve relevant context for query"""
```

**Memory Types:**
```python
class MemoryType(Enum):
    LEARNING = "learning"      # Concepts learned, explanations
    PROJECT = "project"        # Project artifacts, code
    CONVERSATION = "conversation"  # Recent chat history
    PROFILE = "profile"        # User preferences, goals
```

**Vector Storage Schema:**
```python
class VectorDocument:
    id: str
    user_id: str
    content: str
    embedding: list[float]  # 1536-dim for OpenAI embeddings
    memory_type: MemoryType
    metadata: dict
    timestamp: datetime
    similarity_score: float  # Populated during search
```

#### 4. Project Builder Engine

**Responsibilities:**
- Project idea clarification
- Tech stack recommendation
- Architecture diagram generation
- Folder structure creation
- Boilerplate code generation
- Roadmap creation

**API Endpoints:**
```python
POST   /api/project/start
POST   /api/project/clarify
POST   /api/project/generate-stack
POST   /api/project/generate-architecture
POST   /api/project/generate-structure
POST   /api/project/generate-code
POST   /api/project/generate-roadmap
GET    /api/project/{project_id}
POST   /api/project/{project_id}/export
```

**Core Classes:**
```python
class ProjectBuilderEngine:
    def __init__(self, ai_service: AIService, rag_service: RAGService):
        self.ai_service = ai_service
        self.rag_service = rag_service
    
    async def clarify_idea(self, idea: str) -> list[Question]:
        """Generate clarifying questions"""
        
    async def recommend_tech_stack(
        self,
        idea: str,
        clarifications: dict
    ) -> TechStack:
        """Suggest appropriate technologies"""
        
    async def generate_architecture(
        self,
        idea: str,
        tech_stack: TechStack
    ) -> str:
        """Generate text-based architecture diagram"""
        
    async def generate_folder_structure(
        self,
        tech_stack: TechStack
    ) -> FolderNode:
        """Generate project folder hierarchy"""
        
    async def generate_boilerplate(
        self,
        tech_stack: TechStack,
        folder_structure: FolderNode
    ) -> dict[str, str]:
        """Generate starter code for key files"""
        
    async def generate_roadmap(
        self,
        idea: str,
        architecture: str,
        tech_stack: TechStack
    ) -> list[RoadmapStep]:
        """Generate implementation roadmap"""
```

**Tech Stack Recommendation Logic:**
```python
def recommend_tech_stack(idea: str, clarifications: dict) -> TechStack:
    # Analyze project type
    project_type = classify_project_type(idea)
    
    # Consider scale and complexity
    scale = clarifications.get("scale", "small")
    
    # Build recommendation
    if project_type == "web_app":
        frontend = "Next.js + TypeScript + Tailwind"
        backend = "FastAPI" if needs_python else "Node.js + Express"
        database = "PostgreSQL" if needs_relational else "MongoDB"
    elif project_type == "mobile_app":
        frontend = "React Native + TypeScript"
        backend = "Firebase" if scale == "small" else "FastAPI"
    # ... more logic
    
    return TechStack(
        frontend=frontend,
        backend=backend,
        database=database,
        deployment=deployment,
        justification=explain_choices()
    )
```

#### 5. Analytics Service

**Responsibilities:**
- Calculate user progress metrics
- Generate skill proficiency scores
- Aggregate session statistics
- Provide dashboard data

**API Endpoints:**
```python
GET    /api/analytics/overview
GET    /api/analytics/progress
GET    /api/analytics/skills
GET    /api/analytics/sessions
```

**Core Functions:**
```python
async def calculate_overview_stats(user_id: str) -> OverviewStats:
    """Calculate total sessions, concepts, projects, time"""
    
async def calculate_skill_proficiency(user_id: str) -> dict[str, float]:
    """Calculate proficiency scores for each skill"""
    
async def get_progress_timeline(
    user_id: str,
    start_date: date,
    end_date: date
) -> list[ProgressPoint]:
    """Get time-series progress data"""
```

**Skill Proficiency Algorithm:**
```python
def calculate_skill_proficiency(user_id: str, skill: str) -> float:
    # Get all learning memory entries for skill
    entries = get_learning_entries(user_id, skill)
    
    # Factors: frequency, recency, complexity, comprehension
    frequency_score = len(entries) / 100  # Normalize
    recency_score = calculate_recency_decay(entries)
    complexity_score = average_complexity(entries)
    comprehension_score = average_comprehension(entries)
    
    # Weighted combination
    proficiency = (
        0.3 * frequency_score +
        0.2 * recency_score +
        0.3 * complexity_score +
        0.2 * comprehension_score
    )
    
    return min(proficiency, 1.0)  # Cap at 1.0
```


## Data Models

### Database Schema

#### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    profile_data JSONB DEFAULT '{}'::jsonb,
    CONSTRAINT email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}$')
);

CREATE INDEX idx_users_email ON users(email);
```

#### Sessions Table
```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    type VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    summary TEXT,
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_created_at ON sessions(created_at DESC);
```

#### Learning Memory Table
```sql
CREATE TABLE learning_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    concept VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    comprehension_level FLOAT CHECK (comprehension_level >= 0 AND comprehension_level <= 1),
    complexity_level FLOAT CHECK (complexity_level >= 0 AND complexity_level <= 1),
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_learning_memory_user_id ON learning_memory(user_id);
CREATE INDEX idx_learning_memory_concept ON learning_memory(concept);
CREATE INDEX idx_learning_memory_timestamp ON learning_memory(timestamp DESC);
```

#### Project Memory Table
```sql
CREATE TABLE project_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    project_name VARCHAR(255) NOT NULL,
    artifacts JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_project_memory_user_id ON project_memory(user_id);
CREATE INDEX idx_project_memory_created_at ON project_memory(created_at DESC);
```

#### Conversation Memory Table
```sql
CREATE TABLE conversation_memory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('user', 'assistant')),
    content TEXT NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}'::jsonb
);

CREATE INDEX idx_conversation_memory_session_id ON conversation_memory(session_id);
CREATE INDEX idx_conversation_memory_timestamp ON conversation_memory(timestamp);
```

### TypeScript Types (Frontend)

```typescript
// User and Auth
interface User {
  id: string;
  email: string;
  createdAt: Date;
  profileData: {
    name?: string;
    goals?: string[];
    skillLevel?: 'beginner' | 'intermediate' | 'advanced';
    interests?: string[];
  };
}

interface AuthResponse {
  token: string;
  user: User;
}

// Chat and Messages
type AIMode = 'tutor' | 'explainer' | 'debugger' | 'project_builder' | 'learning_planner';

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
  metadata?: {
    mode?: AIMode;
    codeBlocks?: CodeBlock[];
  };
}

interface CodeBlock {
  language: string;
  code: string;
  filename?: string;
}

// Project Builder
interface TechStack {
  frontend: string;
  backend: string;
  database: string;
  deployment: string;
  additional: string[];
  justification: Record<string, string>;
}

interface FolderNode {
  name: string;
  type: 'file' | 'directory';
  children?: FolderNode[];
  purpose?: string;
}

interface RoadmapStep {
  step: number;
  title: string;
  description: string;
  estimatedTime: string;
  complexity: 'easy' | 'medium' | 'hard';
  dependencies: number[];
}

interface ProjectArtifacts {
  idea: string;
  clarifications: Record<string, string>;
  techStack: TechStack;
  architecture: string;
  folderStructure: FolderNode;
  boilerplateCode: Record<string, string>;
  roadmap: RoadmapStep[];
}

// Analytics
interface OverviewStats {
  totalSessions: number;
  conceptsLearned: number;
  projectsCreated: number;
  totalTimeMinutes: number;
}

interface ProgressPoint {
  date: string;
  value: number;
  metric: string;
}

interface SkillProficiency {
  skill: string;
  proficiency: number; // 0-1
  lastPracticed: Date;
}

// Learning Planner
interface DailyTask {
  id: string;
  title: string;
  description: string;
  estimatedMinutes: number;
  completed: boolean;
  resources: string[];
}

interface LearningPlan {
  id: string;
  userId: string;
  goal: string;
  startDate: Date;
  endDate: Date;
  dailyTasks: Record<string, DailyTask[]>; // date -> tasks
  milestones: Milestone[];
}

interface Milestone {
  week: number;
  title: string;
  objectives: string[];
}
```

### Python Models (Backend)

```python
from pydantic import BaseModel, EmailStr, Field
from typing import Optional, List, Dict, Any
from datetime import datetime
from enum import Enum
from uuid import UUID

# User and Auth
class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)

class UserLogin(BaseModel):
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    id: UUID
    email: str
    created_at: datetime
    profile_data: Dict[str, Any]

class TokenResponse(BaseModel):
    token: str
    user: UserResponse

# Chat
class AIMode(str, Enum):
    TUTOR = "tutor"
    EXPLAINER = "explainer"
    DEBUGGER = "debugger"
    PROJECT_BUILDER = "project_builder"
    LEARNING_PLANNER = "learning_planner"

class ChatMessage(BaseModel):
    session_id: Optional[UUID] = None
    message: str
    mode: Optional[AIMode] = None

class MessageResponse(BaseModel):
    id: UUID
    role: str
    content: str
    timestamp: datetime
    metadata: Optional[Dict[str, Any]] = None

# Project Builder
class TechStack(BaseModel):
    frontend: str
    backend: str
    database: str
    deployment: str
    additional: List[str] = []
    justification: Dict[str, str]

class FolderNode(BaseModel):
    name: str
    type: str  # 'file' or 'directory'
    children: Optional[List['FolderNode']] = None
    purpose: Optional[str] = None

class RoadmapStep(BaseModel):
    step: int
    title: str
    description: str
    estimated_time: str
    complexity: str
    dependencies: List[int] = []

class ProjectArtifacts(BaseModel):
    idea: str
    clarifications: Dict[str, str]
    tech_stack: TechStack
    architecture: str
    folder_structure: FolderNode
    boilerplate_code: Dict[str, str]
    roadmap: List[RoadmapStep]

# Analytics
class OverviewStats(BaseModel):
    total_sessions: int
    concepts_learned: int
    projects_created: int
    total_time_minutes: int

class ProgressPoint(BaseModel):
    date: str
    value: float
    metric: str

class SkillProficiency(BaseModel):
    skill: str
    proficiency: float
    last_practiced: datetime

# RAG
class MemoryType(str, Enum):
    LEARNING = "learning"
    PROJECT = "project"
    CONVERSATION = "conversation"
    PROFILE = "profile"

class MemoryItem(BaseModel):
    user_id: UUID
    content: str
    memory_type: MemoryType
    metadata: Dict[str, Any]

class SearchResult(BaseModel):
    content: str
    similarity_score: float
    memory_type: MemoryType
    metadata: Dict[str, Any]
```

