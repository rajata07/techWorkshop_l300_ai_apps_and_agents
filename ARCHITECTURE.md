# Zava AI Platform - Architecture Documentation

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Component Descriptions](#component-descriptions)
3. [Data Flow and Interaction Patterns](#data-flow-and-interaction-patterns)
4. [Security and Compliance](#security-and-compliance)
5. [Implementation Guidance](#implementation-guidance)
6. [Deployment and Operations](#deployment-and-operations)

---

## Architecture Overview

The Zava AI Platform implements a **7-layer enterprise architecture** designed for scalability, security, and observability. The architecture follows Azure AI best practices and Responsible AI principles.

### Architectural Principles

1. **Defense in Depth**: Multiple security layers from API Gateway to data access controls
2. **Separation of Concerns**: Clear boundaries between user interface, orchestration, agents, and data
3. **Read-Only Data Access**: Fabric Lakehouse enforces read-only access with mandatory data masking
4. **Policy-Driven Governance**: Centralized Policy Engine for discount validation, data masking, and safety checks
5. **Observable by Design**: Comprehensive logging and monitoring at every layer
6. **Model Governance**: Azure AI Foundry manages model registry, evaluation, and lifecycle

### Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│ Layer 1: Users                                              │
│ - Customer Web App, Marketing Staff, Developers            │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 2: API & Security                                     │
│ - API Management, Entra ID, Policy Engine, Key Vault       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 3: Orchestration                                      │
│ - Orchestrator Service, Agent Manager, Fallback/Escalation │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 4: Agents                                             │
│ - Customer Design, Marketing Analytics, DevOps Agents      │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 5: Model & Retrieval                                  │
│ - Azure OpenAI, AI Foundry, AI Search (RAG)                │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 6: Data                                               │
│ - Fabric Lakehouse (read-only), Product DB                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ Layer 7: DevOps & Ops                                       │
│ - GitHub Enterprise, Actions, Copilot, AKS, Monitor        │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Descriptions

### Layer 1: Users

#### Customer Web App
- **Purpose**: Primary interface for customers to interact with design and recommendation agents
- **Technology**: Web-based chat interface with JavaScript SDK
- **Authentication**: OAuth 2.0 via Microsoft Entra ID
- **Responsibilities**:
  - Render chat UI and design tools
  - Handle user input and file uploads
  - Display AI-generated recommendations
  - Manage session state

#### Marketing Staff
- **Purpose**: Access analytics and insights from marketing data
- **Access Level**: Authenticated internal users with elevated privileges
- **Responsibilities**:
  - Query aggregated sales and campaign metrics
  - Review AI-generated marketing insights
  - Escalate complex scenarios requiring human judgment

#### Developer
- **Purpose**: Development and maintenance of the platform
- **Access Level**: GitHub Enterprise access with role-based permissions
- **Responsibilities**:
  - Code development and review
  - Infrastructure as Code (IaC) management
  - Utilize GitHub Copilot for development assistance

---

### Layer 2: API & Security

#### Azure API Management (APIM)
- **Purpose**: Centralized API gateway with security, throttling, and observability
- **Key Features**:
  - Rate limiting and quota management
  - Request/response transformation
  - API versioning and routing
  - OAuth 2.0 and JWT validation
  - WAF integration for DDoS protection
- **Configuration**:
  ```json
  {
    "rateLimit": "100 requests/minute",
    "timeout": "30s",
    "retryPolicy": "exponential-backoff",
    "cors": "enabled"
  }
  ```

#### Microsoft Entra ID
- **Purpose**: Identity and access management
- **Key Features**:
  - Single Sign-On (SSO)
  - Multi-factor authentication (MFA)
  - Conditional Access policies
  - Role-Based Access Control (RBAC)
- **Integration Points**:
  - APIM for token validation
  - Agent authorization
  - Administrative access control

#### Policy Engine
- **Purpose**: Centralized policy enforcement for business rules and safety
- **Responsibilities**:
  1. **Data Masking**: Ensure PII and sensitive data is masked before returning to clients
  2. **Discount Validation**: Enforce business rules for discount limits and eligibility
  3. **Safety Checks**: Content filtering, prompt injection detection, toxic content blocking
  4. **Compliance**: GDPR, CCPA, and industry-specific regulations
- **Architecture**:
  ```
  Request → Policy Engine → [Masking] → [Validation] → [Safety] → Response
  ```
- **Implementation**: Azure Functions or containerized service with rule engine

#### Azure Key Vault
- **Purpose**: Secrets and certificate management
- **Stored Secrets**:
  - Azure OpenAI API keys
  - Fabric connection strings
  - Database credentials
  - Certificates for mutual TLS
- **Access Pattern**: Managed Identity integration for zero-credential access

---

### Layer 3: Orchestration

#### Orchestrator Service
- **Purpose**: Central routing and workflow coordination
- **Responsibilities**:
  1. **Intent Classification**: Determine user intent from natural language
  2. **Prompt Engineering**: Assemble context-aware prompts with grounding data
  3. **Routing**: Direct requests to appropriate agents
  4. **Grounding**: Inject relevant context from RAG and data sources
  5. **Response Assembly**: Aggregate multi-agent responses
- **Technology Stack**:
  - Runtime: Azure Container Apps or AKS
  - Framework: FastAPI (Python) or ASP.NET Core (C#)
  - State Management: Redis Cache for session tracking
- **Example Flow**:
  ```python
  async def orchestrate(request):
      intent = await classify_intent(request.message)
      context = await gather_grounding_data(intent)
      agent = select_agent(intent)
      response = await agent.execute(request, context)
      return await policy_check(response)
  ```

#### Agent Manager
- **Purpose**: Manage agent lifecycle and load balancing
- **Responsibilities**:
  - Agent registration and discovery
  - Health monitoring
  - Load balancing across agent instances
  - Scaling based on demand
- **Scaling Policy**:
  - CPU threshold: 70%
  - Queue depth: 100 messages
  - Min replicas: 2, Max replicas: 10

#### Fallback / Human Escalation
- **Purpose**: Handle scenarios beyond AI capability
- **Trigger Conditions**:
  - Low confidence score (< 0.6)
  - Sensitive topics detected
  - Repeated failed attempts
  - Explicit user request for human support
- **Integration**: Service Bus queue for escalation routing to support teams

---

### Layer 4: Agents

#### Customer Design Agent
- **Purpose**: Assist customers with product design and recommendations
- **Capabilities**:
  - Product catalog search via RAG
  - Design suggestions based on preferences
  - Visual design preview generation
  - Order configuration assistance
- **AI Components**:
  - LLM: Azure OpenAI GPT-4 for reasoning
  - RAG: Azure AI Search for product lookup
  - Embeddings: text-embedding-ada-002
- **Example Interaction**:
  ```
  User: "I need a blue t-shirt with my company logo"
  Agent: [Product Search] → [Design Options] → [Price Calculation] → Response
  ```

#### Marketing Analytics Agent
- **Purpose**: Provide data-driven marketing insights
- **Capabilities**:
  - Campaign performance analysis
  - Customer segmentation insights
  - Trend identification
  - Recommendation generation
- **Data Access**:
  - Source: Microsoft Fabric Lakehouse (read-only)
  - Constraints: **Always return masked/aggregated data only**
  - No direct access to raw customer records
- **Compliance**: 
  - All PII masked via Policy Engine
  - Aggregation threshold: minimum 10 records
  - Audit logging for all queries

#### DevOps Agent (Copilot Assistant)
- **Purpose**: Accelerate development and operations workflows
- **Capabilities**:
  - Code suggestions via GitHub Copilot
  - Infrastructure as Code generation
  - Pull request creation and management
  - Build pipeline assistance
- **Integration Points**:
  - GitHub Enterprise for repository access
  - GitHub Actions for CI/CD triggers
  - GitHub Copilot for AI-assisted coding
- **Security**: Read-only access to production environments

---

### Layer 5: Model & Retrieval

#### Azure OpenAI
- **Purpose**: Large Language Model (LLM) inference
- **Models Deployed**:
  - GPT-4 (reasoning and generation)
  - GPT-3.5-Turbo (faster responses)
  - text-embedding-ada-002 (embeddings)
- **Configuration**:
  ```yaml
  deployment:
    model: gpt-4
    version: "0613"
    capacity: 120K TPM
    content_filter: enabled
    max_tokens: 4096
  ```
- **Responsible AI**:
  - Content filtering enabled
  - Abuse monitoring
  - Rate limiting per user

#### Azure AI Foundry
- **Purpose**: Model registry, governance, and lifecycle management
- **Responsibilities**:
  1. **Model Registration**: Catalog of all deployed models
  2. **Model Evaluation**: Performance and safety metrics
  3. **A/B Testing**: Controlled rollout of new models
  4. **Governance**: Approval workflows for model promotion
- **Integration**: Orchestrator queries Foundry for model selection and routing

#### Azure AI Search (RAG)
- **Purpose**: Retrieval-Augmented Generation for product catalog
- **Index Schema**:
  ```json
  {
    "productId": "string",
    "name": "string",
    "description": "searchable",
    "category": "filterable",
    "price": "number",
    "embedding": "vector (1536 dimensions)"
  }
  ```
- **Search Strategy**:
  - Hybrid search (keyword + semantic)
  - Re-ranking with semantic models
  - Top K=5 results for context injection
- **Performance**: < 100ms p95 latency

---

### Layer 6: Data

#### Microsoft Fabric Lakehouse
- **Purpose**: Centralized data warehouse for analytics
- **Access Pattern**: **READ-ONLY**
- **Data Constraints**:
  - ✅ Aggregated metrics only
  - ✅ PII masked via Policy Engine
  - ✅ Row-level security enforced
  - ❌ No raw sales data exposure
  - ❌ No direct table access
- **Example Queries**:
  ```sql
  -- ALLOWED: Aggregated, masked data
  SELECT 
    DATE_TRUNC('month', order_date) as month,
    COUNT(*) as order_count,
    AVG(order_value) as avg_value
  FROM sales
  WHERE customer_id IS NOT NULL
  GROUP BY month
  HAVING COUNT(*) >= 10;  -- Minimum aggregation threshold
  
  -- BLOCKED: Direct customer access
  SELECT * FROM sales WHERE customer_id = '12345';  -- DENIED
  ```

#### Product DB (Catalog Index)
- **Purpose**: Source of truth for product catalog
- **Technology**: Azure SQL Database or Cosmos DB
- **Data Flow**: Product DB → AI Search Index (nightly sync)
- **Schema**: Products, Categories, Pricing, Inventory

---

### Layer 7: DevOps & Ops

#### GitHub Enterprise
- **Purpose**: Source code management and collaboration
- **Repositories**:
  - `/agents` - Agent implementations
  - `/orchestrator` - Orchestration service
  - `/infra` - Terraform/Bicep IaC
  - `/docs` - Documentation

#### GitHub Actions
- **Purpose**: CI/CD automation
- **Workflows**:
  1. **Build & Test**: On PR to main
  2. **Security Scan**: Container scanning, dependency checks
  3. **Deploy to Staging**: On merge to main
  4. **Deploy to Production**: On release tag
- **Deployment Targets**: Azure Kubernetes Service (AKS), App Services, or VMs

#### GitHub Copilot
- **Purpose**: AI-assisted development
- **Integration**: Embedded in developer IDEs
- **Capabilities**: Code completion, test generation, documentation

#### Deployment Host (AKS / App Services / VMs)
- **Purpose**: Runtime environment for services
- **Recommended**: Azure Kubernetes Service (AKS) for microservices
- **Configuration**:
  - Auto-scaling enabled
  - Managed Identity for Azure service access
  - Private endpoints for data services

#### Azure Monitor & Log Analytics
- **Purpose**: Observability and alerting
- **Metrics Collected**:
  - Request latency, throughput, error rates
  - Agent performance and accuracy
  - Model inference metrics
  - Cost tracking (OpenAI token usage)
- **Alerts**:
  - Error rate > 5%
  - Latency > 3 seconds
  - Policy violations detected
- **Integration**: All services send logs and metrics to centralized workspace

---

## Data Flow and Interaction Patterns

### Pattern 1: Customer Design Request Flow

```
1. Customer → Web App → API Management
2. API Management → Entra ID (Auth) → Policy Engine (Safety Check)
3. Policy Engine → Orchestrator Service (Intent: "product_recommendation")
4. Orchestrator → Agent Manager → Customer Design Agent
5. Customer Design Agent → Azure AI Search (RAG: product catalog)
6. Customer Design Agent → Azure OpenAI (LLM reasoning)
7. Azure OpenAI ← AI Foundry (model governance check)
8. Customer Design Agent → Orchestrator (response assembly)
9. Orchestrator → Policy Engine (data masking, discount validation)
10. Policy Engine → API Management → Customer Web App
11. All layers → Azure Monitor (logs, metrics, audit trail)
```

**Latency Budget**: < 3 seconds end-to-end

### Pattern 2: Marketing Analytics Request Flow

```
1. Marketing Staff → Web App → API Management
2. API Management → Entra ID (Auth + RBAC) → Policy Engine
3. Policy Engine → Orchestrator Service (Intent: "analytics_query")
4. Orchestrator → Agent Manager → Marketing Analytics Agent
5. Marketing Analytics Agent → Fabric Lakehouse (read-only, aggregated query)
6. Fabric Lakehouse → Policy Engine (enforce masking + aggregation)
7. Marketing Analytics Agent → Azure OpenAI (insight generation)
8. Marketing Analytics Agent → Orchestrator (response)
9. Orchestrator → Policy Engine (final validation)
10. Policy Engine → API Management → Marketing Staff
11. All layers → Azure Monitor (audit trail for data access)
```

**Critical Controls**:
- Fabric access always through Policy Engine
- Minimum aggregation: 10 records
- PII automatically masked

### Pattern 3: DevOps Agent Workflow

```
1. Developer → GitHub Enterprise (push code)
2. GitHub Enterprise → GitHub Actions (trigger build)
3. GitHub Actions → DevOps Agent (IaC generation, PR review)
4. DevOps Agent ← GitHub Copilot (code suggestions)
5. GitHub Actions → Deployment Host (deploy to staging/prod)
6. Deployment Host → Azure Monitor (deployment metrics)
```

### Pattern 4: Fallback / Escalation Flow

```
1. Agent detects low confidence (< 0.6) or sensitive topic
2. Agent → Orchestrator (escalation_required flag)
3. Orchestrator → Fallback Service (queue message)
4. Fallback Service → Marketing Staff (notification)
5. Marketing Staff → Web App (manual response)
```

---

## Security and Compliance

### Security Controls by Layer

#### Layer 2: API & Security
- ✅ WAF (Web Application Firewall) enabled
- ✅ DDoS protection via Azure Front Door
- ✅ TLS 1.3 encryption in transit
- ✅ OAuth 2.0 + JWT token validation
- ✅ API rate limiting and throttling
- ✅ Content Security Policy (CSP) headers

#### Layer 3: Orchestration
- ✅ Managed Identity for Azure service access
- ✅ No hardcoded secrets (Key Vault integration)
- ✅ Network isolation via Virtual Network
- ✅ Private endpoints for internal services

#### Layer 4: Agents
- ✅ Prompt injection detection
- ✅ Input sanitization and validation
- ✅ Output content filtering
- ✅ Agent isolation (separate containers)

#### Layer 5: Model & Retrieval
- ✅ Azure OpenAI content filters (hate, violence, sexual, self-harm)
- ✅ Model access via Managed Identity
- ✅ Abuse monitoring and alerts

#### Layer 6: Data
- ✅ Fabric Lakehouse: read-only access enforced
- ✅ Row-level security (RLS) via Entra ID groups
- ✅ Data at rest encryption (AES-256)
- ✅ PII masking via Policy Engine
- ✅ Aggregation thresholds enforced

### Compliance Framework

#### GDPR (General Data Protection Regulation)
- **Right to be Forgotten**: Customer data deletion workflows
- **Data Minimization**: Only collect necessary data
- **Consent Management**: Explicit user consent for data processing
- **Data Portability**: Export user data on request

#### Responsible AI Principles
1. **Fairness**: Regular bias testing in model outputs
2. **Reliability & Safety**: Content filtering, fallback mechanisms
3. **Privacy & Security**: Data masking, encryption, access controls
4. **Inclusiveness**: Multi-language support, accessibility standards
5. **Transparency**: Model cards, explainability features
6. **Accountability**: Audit logs, human oversight

#### Audit and Logging
- **Retention**: 90 days in hot storage, 2 years in cold storage
- **Logged Events**:
  - All API requests with user context
  - Policy Engine decisions (masking, validation)
  - Agent invocations and responses
  - Data access patterns (Fabric queries)
  - Authentication and authorization events
- **Immutable Logs**: Write-once, tamper-proof storage

---

## Implementation Guidance

### Getting Started

#### Prerequisites
1. Azure subscription with required services:
   - Azure API Management (Developer or Premium tier)
   - Microsoft Entra ID (P1 or P2)
   - Azure OpenAI (approved access)
   - Azure AI Foundry (preview enrollment)
   - Azure AI Search (Standard tier)
   - Microsoft Fabric (F64 or higher)
   - Azure Kubernetes Service (AKS) or App Services
   - Azure Monitor & Log Analytics
2. GitHub Enterprise account
3. Development tools:
   - VS Code with GitHub Copilot
   - Azure CLI
   - Terraform or Bicep
   - Docker

#### Phase 1: Infrastructure Setup (Week 1-2)

1. **Network Foundation**
   ```bash
   # Create resource group
   az group create --name rg-zava-prod --location eastus2
   
   # Deploy Virtual Network
   az network vnet create \
     --resource-group rg-zava-prod \
     --name vnet-zava \
     --address-prefix 10.0.0.0/16
   
   # Create subnets for each layer
   az network vnet subnet create \
     --resource-group rg-zava-prod \
     --vnet-name vnet-zava \
     --name snet-apim \
     --address-prefix 10.0.1.0/24
   ```

2. **Deploy Azure API Management**
   ```bash
   az apim create \
     --resource-group rg-zava-prod \
     --name apim-zava-prod \
     --publisher-email admin@zava.com \
     --publisher-name "Zava Inc" \
     --sku-name Developer \
     --virtual-network External
   ```

3. **Configure Key Vault**
   ```bash
   az keyvault create \
     --resource-group rg-zava-prod \
     --name kv-zava-prod \
     --enable-rbac-authorization true
   ```

4. **Deploy Azure OpenAI**
   ```bash
   az cognitiveservices account create \
     --name openai-zava-prod \
     --resource-group rg-zava-prod \
     --kind OpenAI \
     --sku S0 \
     --location eastus2
   
   # Deploy GPT-4 model
   az cognitiveservices account deployment create \
     --name openai-zava-prod \
     --resource-group rg-zava-prod \
     --deployment-name gpt-4 \
     --model-name gpt-4 \
     --model-version "0613" \
     --model-format OpenAI \
     --sku-capacity 120 \
     --sku-name Standard
   ```

5. **Deploy Azure AI Search**
   ```bash
   az search service create \
     --resource-group rg-zava-prod \
     --name search-zava-prod \
     --sku standard
   ```

#### Phase 2: Policy Engine (Week 2-3)

**Implementation**: Azure Functions (Python)

```python
# policy_engine/discount_validator.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class DiscountPolicy:
    max_percentage: float = 30.0
    max_amount: float = 1000.0
    min_order_value: float = 100.0

def validate_discount(
    discount_percentage: float,
    discount_amount: float,
    order_value: float
) -> tuple[bool, Optional[str]]:
    """
    Validate discount against business rules.
    Returns (is_valid, error_message)
    """
    policy = DiscountPolicy()
    
    if discount_percentage > policy.max_percentage:
        return False, f"Discount exceeds maximum {policy.max_percentage}%"
    
    if discount_amount > policy.max_amount:
        return False, f"Discount exceeds maximum ${policy.max_amount}"
    
    if order_value < policy.min_order_value:
        return False, f"Order value must be at least ${policy.min_order_value}"
    
    return True, None
```

```python
# policy_engine/data_masking.py
import re
from typing import Any, Dict

def mask_pii(data: Dict[str, Any]) -> Dict[str, Any]:
    """
    Mask personally identifiable information.
    """
    masked_data = data.copy()
    
    # Email masking
    if 'email' in masked_data:
        email = masked_data['email']
        masked_data['email'] = re.sub(
            r'(.{2}).*(@.*)',
            r'\1***\2',
            email
        )
    
    # Phone masking
    if 'phone' in masked_data:
        phone = masked_data['phone']
        masked_data['phone'] = f"***-***-{phone[-4:]}"
    
    # Name masking
    if 'name' in masked_data:
        name = masked_data['name']
        masked_data['name'] = f"{name[0]}***"
    
    return masked_data
```

#### Phase 3: Orchestrator Service (Week 3-4)

**Implementation**: FastAPI (Python)

```python
# orchestrator/main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional
import httpx

app = FastAPI(title="Zava Orchestrator")

class OrchestratorRequest(BaseModel):
    user_message: str
    user_id: str
    session_id: str
    context: Optional[dict] = None

class OrchestratorResponse(BaseModel):
    agent_response: str
    confidence: float
    agent_used: str
    escalated: bool = False

@app.post("/orchestrate", response_model=OrchestratorResponse)
async def orchestrate(request: OrchestratorRequest):
    # Step 1: Classify intent
    intent = await classify_intent(request.user_message)
    
    # Step 2: Route to appropriate agent
    agent = select_agent(intent)
    
    # Step 3: Gather grounding data
    context = await gather_context(intent, request.user_id)
    
    # Step 4: Execute agent
    response = await agent.execute(request.user_message, context)
    
    # Step 5: Check if escalation needed
    if response.confidence < 0.6:
        await escalate(request, response)
        return OrchestratorResponse(
            agent_response="I've escalated your request to a team member.",
            confidence=response.confidence,
            agent_used=agent.name,
            escalated=True
        )
    
    # Step 6: Policy validation
    validated_response = await policy_check(response)
    
    return OrchestratorResponse(
        agent_response=validated_response.text,
        confidence=response.confidence,
        agent_used=agent.name
    )

async def classify_intent(message: str) -> str:
    """Use Azure OpenAI to classify user intent."""
    # Implementation here
    pass

def select_agent(intent: str):
    """Route to appropriate agent based on intent."""
    # Implementation here
    pass
```

#### Phase 4: Agents (Week 4-6)

**Customer Design Agent**:

```python
# agents/customer_design_agent.py
from azure.search.documents import SearchClient
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI

class CustomerDesignAgent:
    def __init__(self):
        self.search_client = SearchClient(
            endpoint=os.getenv("SEARCH_ENDPOINT"),
            index_name="product-catalog",
            credential=DefaultAzureCredential()
        )
        self.openai_client = AzureOpenAI(
            azure_endpoint=os.getenv("OPENAI_ENDPOINT"),
            api_version="2024-02-01",
            azure_ad_token_provider=get_bearer_token_provider(
                DefaultAzureCredential(),
                "https://cognitiveservices.azure.com/.default"
            )
        )
    
    async def execute(self, user_message: str, context: dict):
        # Step 1: RAG - Search product catalog
        search_results = await self.search_products(user_message)
        
        # Step 2: Build grounded prompt
        prompt = self.build_prompt(user_message, search_results, context)
        
        # Step 3: LLM reasoning
        response = self.openai_client.chat.completions.create(
            model="gpt-4",
            messages=[
                {"role": "system", "content": "You are a helpful product design advisor."},
                {"role": "user", "content": prompt}
            ],
            max_tokens=1000,
            temperature=0.7
        )
        
        return {
            "text": response.choices[0].message.content,
            "confidence": self.calculate_confidence(response),
            "products_referenced": [r["productId"] for r in search_results]
        }
    
    async def search_products(self, query: str):
        results = self.search_client.search(
            search_text=query,
            select=["productId", "name", "description", "price"],
            top=5
        )
        return [doc for doc in results]
```

**Marketing Analytics Agent**:

```python
# agents/marketing_analytics_agent.py
from azure.identity import DefaultAzureCredential
import pyodbc

class MarketingAnalyticsAgent:
    def __init__(self):
        self.credential = DefaultAzureCredential()
        # Read-only connection to Fabric
        self.connection_string = self._get_fabric_connection()
    
    async def execute(self, user_message: str, context: dict):
        # Step 1: Generate SQL query (with aggregation enforced)
        sql_query = await self.generate_query(user_message)
        
        # Step 2: Validate query enforces aggregation
        if not self.is_aggregated_query(sql_query):
            raise ValueError("Query must use aggregation (COUNT, AVG, SUM)")
        
        # Step 3: Execute against Fabric (read-only)
        data = await self.execute_query(sql_query)
        
        # Step 4: Mask PII via Policy Engine
        masked_data = await self.mask_data(data)
        
        # Step 5: Generate insights with LLM
        insights = await self.generate_insights(masked_data, user_message)
        
        return {
            "text": insights,
            "confidence": 0.9,
            "data_points": len(masked_data)
        }
    
    def is_aggregated_query(self, sql: str) -> bool:
        """Ensure query uses aggregation functions."""
        aggregation_keywords = ["COUNT", "AVG", "SUM", "MIN", "MAX", "GROUP BY"]
        return any(keyword in sql.upper() for keyword in aggregation_keywords)
```

#### Phase 5: Monitoring & Observability (Week 6-7)

**Azure Monitor Configuration**:

```python
# monitoring/telemetry.py
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace, metrics
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# Configure Azure Monitor
configure_azure_monitor(
    connection_string=os.getenv("APPLICATIONINSIGHTS_CONNECTION_STRING")
)

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Custom metrics
request_counter = meter.create_counter(
    "orchestrator.requests",
    description="Number of orchestration requests"
)

latency_histogram = meter.create_histogram(
    "orchestrator.latency",
    description="Request latency in milliseconds"
)

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)

@app.middleware("http")
async def add_telemetry(request, call_next):
    with tracer.start_as_current_span("orchestrator_request") as span:
        start_time = time.time()
        
        response = await call_next(request)
        
        latency = (time.time() - start_time) * 1000
        latency_histogram.record(latency)
        request_counter.add(1)
        
        span.set_attribute("http.status_code", response.status_code)
        span.set_attribute("http.latency_ms", latency)
        
        return response
```

### Testing Strategy

#### Unit Tests
```python
# tests/test_policy_engine.py
import pytest
from policy_engine.discount_validator import validate_discount

def test_discount_within_limits():
    is_valid, error = validate_discount(
        discount_percentage=20.0,
        discount_amount=50.0,
        order_value=250.0
    )
    assert is_valid is True
    assert error is None

def test_discount_exceeds_percentage():
    is_valid, error = validate_discount(
        discount_percentage=35.0,
        discount_amount=50.0,
        order_value=250.0
    )
    assert is_valid is False
    assert "exceeds maximum" in error
```

#### Integration Tests
```python
# tests/test_orchestrator_integration.py
import pytest
from httpx import AsyncClient
from orchestrator.main import app

@pytest.mark.asyncio
async def test_customer_design_flow():
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.post("/orchestrate", json={
            "user_message": "I need a blue t-shirt",
            "user_id": "test-user-123",
            "session_id": "test-session-456"
        })
    
    assert response.status_code == 200
    data = response.json()
    assert data["agent_used"] == "CustomerDesignAgent"
    assert data["confidence"] > 0.6
```

#### Load Tests
```python
# tests/load_test.py
from locust import HttpUser, task, between

class ZavaUser(HttpUser):
    wait_time = between(1, 3)
    
    @task
    def orchestrate_request(self):
        self.client.post("/orchestrate", json={
            "user_message": "Show me trending products",
            "user_id": f"user-{self.user_id}",
            "session_id": f"session-{self.session_id}"
        })
```

**Run load test**:
```bash
locust -f tests/load_test.py --host=https://apim-zava-prod.azure-api.net
```

---

## Deployment and Operations

### CI/CD Pipeline

#### GitHub Actions Workflow

```yaml
# .github/workflows/deploy-orchestrator.yml
name: Deploy Orchestrator Service

on:
  push:
    branches: [main]
    paths:
      - 'orchestrator/**'
  pull_request:
    branches: [main]

env:
  AZURE_RESOURCE_GROUP: rg-zava-prod
  AKS_CLUSTER_NAME: aks-zava-prod
  ACR_NAME: acrzavaprod

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          cd orchestrator
          pip install -r requirements.txt
          pip install pytest pytest-asyncio
      
      - name: Run unit tests
        run: |
          cd orchestrator
          pytest tests/ -v
      
      - name: Run security scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: 'orchestrator/'
  
  build-and-push:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Build and push Docker image
        run: |
          az acr login --name ${{ env.ACR_NAME }}
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/orchestrator:${{ github.sha }} orchestrator/
          docker push ${{ env.ACR_NAME }}.azurecr.io/orchestrator:${{ github.sha }}
  
  deploy-to-aks:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}
      
      - name: Get AKS credentials
        run: |
          az aks get-credentials \
            --resource-group ${{ env.AZURE_RESOURCE_GROUP }} \
            --name ${{ env.AKS_CLUSTER_NAME }}
      
      - name: Deploy to AKS
        run: |
          kubectl set image deployment/orchestrator \
            orchestrator=${{ env.ACR_NAME }}.azurecr.io/orchestrator:${{ github.sha }} \
            -n zava-prod
          kubectl rollout status deployment/orchestrator -n zava-prod
```

### Kubernetes Deployment

```yaml
# k8s/orchestrator-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orchestrator
  namespace: zava-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orchestrator
  template:
    metadata:
      labels:
        app: orchestrator
    spec:
      serviceAccountName: orchestrator-sa
      containers:
      - name: orchestrator
        image: acrzavaprod.azurecr.io/orchestrator:latest
        ports:
        - containerPort: 8000
        env:
        - name: APPLICATIONINSIGHTS_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: app-insights-secret
              key: connection-string
        - name: AZURE_CLIENT_ID
          value: "managed-identity-client-id"
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: orchestrator-service
  namespace: zava-prod
spec:
  selector:
    app: orchestrator
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orchestrator-hpa
  namespace: zava-prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orchestrator
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

### Operational Runbooks

#### Incident Response

**High Error Rate Alert**

```markdown
## Alert: Error Rate > 5%

### Immediate Actions
1. Check Azure Monitor dashboard for affected services
2. Review recent deployments (last 2 hours)
3. Check OpenAI service health
4. Verify API Management policies

### Investigation Steps
1. Query Log Analytics:
   ```kusto
   traces
   | where timestamp > ago(1h)
   | where severityLevel >= 3
   | summarize count() by cloud_RoleName, operation_Name
   | order by count_ desc
   ```

2. Check agent performance:
   ```kusto
   customMetrics
   | where name == "agent.latency"
   | summarize avg(value), max(value) by bin(timestamp, 5m)
   ```

### Remediation
- If OpenAI throttling: Enable fallback to GPT-3.5-Turbo
- If agent errors: Roll back to previous version
- If database issues: Check connection pool exhaustion

### Escalation
- On-call: @platform-team
- Severity 1: Page VP Engineering after 30 minutes
```

#### Capacity Planning

**Monthly Review Checklist**:
- [ ] Review Azure OpenAI quota usage
- [ ] Analyze AKS cluster resource utilization
- [ ] Check API Management rate limit hits
- [ ] Review Fabric Lakehouse query patterns
- [ ] Analyze cost trends and optimize

### Cost Optimization

**Current Cost Breakdown** (Estimated monthly for 10K MAU):

| Service | Cost | Optimization |
|---------|------|--------------|
| Azure OpenAI (GPT-4) | $5,000 | Use GPT-3.5 for simple queries |
| Azure AI Search | $1,200 | Right-size to Basic tier for dev |
| AKS Cluster | $2,500 | Use spot instances for non-prod |
| API Management | $800 | Developer tier sufficient for <1M calls |
| Fabric Lakehouse | $3,000 | Optimize query patterns, cache results |
| Monitoring & Logs | $500 | 30-day retention, archive to cold storage |
| **Total** | **$13,000** | |

**Optimization Tips**:
1. Implement response caching for common queries (save 30% on OpenAI costs)
2. Use prompt optimization to reduce token usage
3. Batch AI Search queries where possible
4. Use reserved instances for production AKS nodes

---

## Appendix

### Glossary

- **RAG**: Retrieval-Augmented Generation - combining retrieval from knowledge bases with LLM generation
- **Managed Identity**: Azure AD identity for resources to access other Azure services without credentials
- **Policy Engine**: Centralized service enforcing business rules, safety checks, and data masking
- **Orchestrator**: Central service coordinating agent workflows and routing requests
- **Grounding**: Providing factual context to LLM prompts to improve accuracy

### References

- [Azure AI Foundry Documentation](https://learn.microsoft.com/azure/ai-studio/)
- [Azure OpenAI Best Practices](https://learn.microsoft.com/azure/ai-services/openai/)
- [Microsoft Responsible AI Guidelines](https://www.microsoft.com/ai/responsible-ai)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)

### Contact

- **Architecture Team**: architecture@zava.com
- **Platform Team**: platform@zava.com
- **Security Team**: security@zava.com

---

**Document Version**: 1.0  
**Last Updated**: 2025-12-10  
**Maintained By**: Platform Architecture Team
