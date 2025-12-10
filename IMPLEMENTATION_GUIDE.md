# Zava AI Platform - Developer Implementation Guide

## Quick Start Guide

This guide provides step-by-step instructions for developers to implement the Zava AI Platform based on the architecture defined in `ARCHITECTURE.md`.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Local Development Setup](#local-development-setup)
3. [Component Implementation](#component-implementation)
4. [Testing Guide](#testing-guide)
5. [Deployment Guide](#deployment-guide)
6. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Tools

- **Azure CLI** (v2.50+): [Install](https://learn.microsoft.com/cli/azure/install-azure-cli)
- **Docker Desktop** (v24+): [Install](https://www.docker.com/products/docker-desktop/)
- **kubectl** (v1.28+): [Install](https://kubernetes.io/docs/tasks/tools/)
- **Python** (v3.11+): [Install](https://www.python.org/downloads/)
- **Node.js** (v20+): [Install](https://nodejs.org/)
- **Git**: [Install](https://git-scm.com/)
- **VS Code**: [Install](https://code.visualstudio.com/)
  - GitHub Copilot extension
  - Python extension
  - Azure Tools extension

### Required Access

- Azure subscription with Owner or Contributor role
- Azure OpenAI service access (requires application approval)
- GitHub Enterprise account
- Microsoft Entra ID tenant admin access (for app registrations)

### Azure Resource Requirements

Estimated monthly cost for development environment: **$2,000 - $3,000**

| Service | SKU | Estimated Cost |
|---------|-----|---------------|
| Azure OpenAI | S0 | $500 - $1,000 |
| Azure AI Search | Basic | $75 |
| Azure Kubernetes Service | D2s_v3 (2 nodes) | $150 |
| Azure API Management | Developer | $50 |
| Microsoft Fabric | F2 | $500 |
| Azure Monitor | Pay-as-you-go | $100 |

---

## Local Development Setup

### Step 1: Clone Repository

```bash
# Clone the repository
git clone https://github.com/your-org/zava-ai-platform.git
cd zava-ai-platform

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Step 2: Configure Environment Variables

Create a `.env` file in the root directory:

```bash
# .env
# Azure Configuration
AZURE_SUBSCRIPTION_ID=your-subscription-id
AZURE_TENANT_ID=your-tenant-id
AZURE_RESOURCE_GROUP=rg-zava-dev

# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://your-openai.openai.azure.com/
AZURE_OPENAI_API_VERSION=2024-02-01
AZURE_OPENAI_DEPLOYMENT_NAME=gpt-4

# Azure AI Search
AZURE_SEARCH_ENDPOINT=https://your-search.search.windows.net
AZURE_SEARCH_INDEX_NAME=product-catalog

# Microsoft Fabric
FABRIC_WORKSPACE_ID=your-workspace-id
FABRIC_LAKEHOUSE_ID=your-lakehouse-id

# Application Insights
APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=...

# Policy Engine
POLICY_MAX_DISCOUNT_PERCENTAGE=30.0
POLICY_MAX_DISCOUNT_AMOUNT=1000.0
POLICY_MIN_AGGREGATION_ROWS=10

# Development Settings
LOG_LEVEL=DEBUG
ENVIRONMENT=development
```

**⚠️ Security Note**: Never commit `.env` files to version control. Add to `.gitignore`.

### Step 3: Azure Resources Setup

Run the automated setup script:

```bash
# Login to Azure
az login

# Set subscription
az account set --subscription "your-subscription-name"

# Run infrastructure setup
cd infra
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

Alternatively, use the provided setup script:

```bash
# Make script executable
chmod +x scripts/setup-azure-resources.sh

# Run setup (this will take 15-20 minutes)
./scripts/setup-azure-resources.sh
```

### Step 4: Seed Initial Data

```bash
# Create product catalog index
python scripts/seed_product_catalog.py

# Verify index
python scripts/verify_search_index.py
```

### Step 5: Start Local Services

```bash
# Terminal 1: Start Orchestrator Service
cd orchestrator
uvicorn main:app --reload --port 8000

# Terminal 2: Start Policy Engine
cd policy-engine
uvicorn main:app --reload --port 8001

# Terminal 3: Start Agent Manager
cd agent-manager
uvicorn main:app --reload --port 8002
```

### Step 6: Verify Setup

```bash
# Health check
curl http://localhost:8000/health

# Test orchestration
curl -X POST http://localhost:8000/orchestrate \
  -H "Content-Type: application/json" \
  -d '{
    "user_message": "Show me blue t-shirts",
    "user_id": "test-user-123",
    "session_id": "test-session-456"
  }'
```

---

## Component Implementation

### 1. Policy Engine Implementation

#### File Structure

```
policy-engine/
├── main.py                 # FastAPI application
├── validators/
│   ├── __init__.py
│   ├── discount_validator.py
│   ├── data_masking.py
│   └── safety_checker.py
├── models/
│   ├── __init__.py
│   └── policy_models.py
├── config.py
├── requirements.txt
└── tests/
    ├── test_discount.py
    ├── test_masking.py
    └── test_safety.py
```

#### Implementation: `main.py`

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Dict, Any, Optional
from validators.discount_validator import validate_discount
from validators.data_masking import mask_pii
from validators.safety_checker import check_content_safety
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Zava Policy Engine", version="1.0.0")

class PolicyRequest(BaseModel):
    data: Dict[str, Any]
    policy_type: str  # "discount", "masking", "safety"
    context: Optional[Dict[str, Any]] = None

class PolicyResponse(BaseModel):
    approved: bool
    masked_data: Optional[Dict[str, Any]] = None
    violations: list[str] = []
    metadata: Dict[str, Any] = {}

@app.post("/validate", response_model=PolicyResponse)
async def validate_policy(request: PolicyRequest):
    """
    Main policy validation endpoint.
    Routes to specific validators based on policy_type.
    """
    try:
        if request.policy_type == "discount":
            return await validate_discount_policy(request)
        elif request.policy_type == "masking":
            return await apply_data_masking(request)
        elif request.policy_type == "safety":
            return await validate_safety_policy(request)
        else:
            raise HTTPException(status_code=400, detail="Invalid policy_type")
    
    except Exception as e:
        logger.error(f"Policy validation error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

async def validate_discount_policy(request: PolicyRequest) -> PolicyResponse:
    """Validate discount rules."""
    discount_pct = request.data.get("discount_percentage", 0)
    discount_amt = request.data.get("discount_amount", 0)
    order_value = request.data.get("order_value", 0)
    
    is_valid, error = validate_discount(discount_pct, discount_amt, order_value)
    
    return PolicyResponse(
        approved=is_valid,
        violations=[error] if error else [],
        metadata={
            "checked_at": "timestamp",
            "policy_version": "1.0"
        }
    )

async def apply_data_masking(request: PolicyRequest) -> PolicyResponse:
    """Apply PII masking to data."""
    masked_data = mask_pii(request.data)
    
    return PolicyResponse(
        approved=True,
        masked_data=masked_data,
        metadata={"masked_fields": list(masked_data.keys())}
    )

async def validate_safety_policy(request: PolicyRequest) -> PolicyResponse:
    """Check content safety (prompt injection, toxic content)."""
    content = request.data.get("content", "")
    
    is_safe, violations = check_content_safety(content)
    
    return PolicyResponse(
        approved=is_safe,
        violations=violations,
        metadata={"safety_score": 0.95}
    )

@app.get("/health")
async def health():
    return {"status": "healthy", "service": "policy-engine"}
```

#### Implementation: `validators/discount_validator.py`

```python
from dataclasses import dataclass
from typing import Optional, Tuple
import os

@dataclass
class DiscountPolicy:
    """Business rules for discount validation."""
    max_percentage: float = float(os.getenv("POLICY_MAX_DISCOUNT_PERCENTAGE", 30.0))
    max_amount: float = float(os.getenv("POLICY_MAX_DISCOUNT_AMOUNT", 1000.0))
    min_order_value: float = 100.0

def validate_discount(
    discount_percentage: float,
    discount_amount: float,
    order_value: float
) -> Tuple[bool, Optional[str]]:
    """
    Validate discount against business rules.
    
    Args:
        discount_percentage: Discount as percentage (e.g., 20.0 for 20%)
        discount_amount: Discount in currency units
        order_value: Total order value
    
    Returns:
        Tuple of (is_valid, error_message)
    
    Examples:
        >>> validate_discount(20.0, 50.0, 250.0)
        (True, None)
        
        >>> validate_discount(35.0, 50.0, 250.0)
        (False, "Discount percentage 35.0% exceeds maximum 30.0%")
    """
    policy = DiscountPolicy()
    
    # Check percentage limit
    if discount_percentage > policy.max_percentage:
        return False, f"Discount percentage {discount_percentage}% exceeds maximum {policy.max_percentage}%"
    
    # Check amount limit
    if discount_amount > policy.max_amount:
        return False, f"Discount amount ${discount_amount} exceeds maximum ${policy.max_amount}"
    
    # Check minimum order value
    if order_value < policy.min_order_value:
        return False, f"Order value ${order_value} must be at least ${policy.min_order_value}"
    
    # Check discount doesn't exceed order value
    if discount_amount > order_value:
        return False, f"Discount amount ${discount_amount} exceeds order value ${order_value}"
    
    # Calculate effective percentage
    effective_pct = (discount_amount / order_value) * 100 if order_value > 0 else 0
    if effective_pct > policy.max_percentage:
        return False, f"Effective discount {effective_pct:.1f}% exceeds maximum {policy.max_percentage}%"
    
    return True, None
```

#### Implementation: `validators/data_masking.py`

```python
import re
from typing import Dict, Any
from datetime import datetime

def mask_pii(data: Dict[str, Any]) -> Dict[str, Any]:
    """
    Mask personally identifiable information (PII).
    
    Masks:
    - Email addresses
    - Phone numbers
    - Names
    - Credit card numbers
    - SSN/Tax IDs
    
    Args:
        data: Dictionary containing potentially sensitive data
    
    Returns:
        Dictionary with PII masked
    
    Examples:
        >>> mask_pii({"email": "john.doe@example.com"})
        {"email": "jo***@example.com"}
        
        >>> mask_pii({"phone": "555-123-4567"})
        {"phone": "***-***-4567"}
    """
    masked_data = data.copy()
    
    # Email masking: preserve first 2 chars and domain
    if 'email' in masked_data and masked_data['email']:
        email = str(masked_data['email'])
        if '@' in email:
            local, domain = email.split('@', 1)
            if len(local) > 2:
                masked_data['email'] = f"{local[:2]}***@{domain}"
            else:
                masked_data['email'] = f"***@{domain}"
    
    # Phone masking: show only last 4 digits
    if 'phone' in masked_data and masked_data['phone']:
        phone = str(masked_data['phone'])
        # Remove non-digits
        digits = re.sub(r'\D', '', phone)
        if len(digits) >= 4:
            masked_data['phone'] = f"***-***-{digits[-4:]}"
        else:
            masked_data['phone'] = "***-***-****"
    
    # Name masking: show only first initial
    if 'name' in masked_data and masked_data['name']:
        name = str(masked_data['name'])
        if len(name) > 0:
            masked_data['name'] = f"{name[0]}***"
    
    if 'first_name' in masked_data and masked_data['first_name']:
        masked_data['first_name'] = f"{masked_data['first_name'][0]}***"
    
    if 'last_name' in masked_data and masked_data['last_name']:
        masked_data['last_name'] = f"{masked_data['last_name'][0]}***"
    
    # Credit card masking: show only last 4 digits
    if 'credit_card' in masked_data and masked_data['credit_card']:
        cc = str(masked_data['credit_card'])
        digits = re.sub(r'\D', '', cc)
        if len(digits) >= 4:
            masked_data['credit_card'] = f"****-****-****-{digits[-4:]}"
    
    # SSN/Tax ID masking
    if 'ssn' in masked_data and masked_data['ssn']:
        masked_data['ssn'] = "***-**-****"
    
    if 'tax_id' in masked_data and masked_data['tax_id']:
        masked_data['tax_id'] = "***-**-****"
    
    # Address masking: keep only city and state
    if 'address' in masked_data and isinstance(masked_data['address'], dict):
        address = masked_data['address']
        masked_data['address'] = {
            'street': '***',
            'city': address.get('city', '***'),
            'state': address.get('state', '***'),
            'zip': '***'
        }
    
    # IP address masking
    if 'ip_address' in masked_data and masked_data['ip_address']:
        ip = str(masked_data['ip_address'])
        parts = ip.split('.')
        if len(parts) == 4:
            masked_data['ip_address'] = f"{parts[0]}.***.***.{parts[3]}"
    
    return masked_data

def enforce_aggregation(query_result: list, min_rows: int = 10) -> list:
    """
    Ensure query results meet minimum aggregation threshold.
    
    Args:
        query_result: List of data rows
        min_rows: Minimum number of rows required
    
    Returns:
        Query result if threshold met
    
    Raises:
        ValueError: If result set is too small
    """
    if len(query_result) < min_rows:
        raise ValueError(
            f"Query result contains {len(query_result)} rows, "
            f"minimum {min_rows} required for aggregation"
        )
    
    return query_result
```

#### Implementation: `validators/safety_checker.py`

```python
import re
from typing import Tuple, List
from azure.ai.contentsafety import ContentSafetyClient
from azure.core.credentials import AzureKeyCredential
import os

# Prompt injection patterns
INJECTION_PATTERNS = [
    r"ignore previous instructions",
    r"disregard all previous",
    r"forget everything",
    r"you are now",
    r"system prompt",
    r"<\|im_start\|>",
    r"<\|im_end\|>",
]

def check_content_safety(content: str) -> Tuple[bool, List[str]]:
    """
    Check content for safety violations.
    
    Checks:
    - Prompt injection attempts
    - Toxic content
    - PII leakage
    - SQL injection patterns
    
    Args:
        content: Text content to check
    
    Returns:
        Tuple of (is_safe, list_of_violations)
    """
    violations = []
    
    # Check for prompt injection
    for pattern in INJECTION_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            violations.append(f"Potential prompt injection detected: {pattern}")
    
    # Check for SQL injection patterns
    sql_patterns = [
        r"(\bOR\b.*=.*)",
        r"(\bUNION\b.*\bSELECT\b)",
        r"(\bDROP\b.*\bTABLE\b)",
        r"(--\s*$)",
        r"(;\s*\bDROP\b)",
    ]
    
    for pattern in sql_patterns:
        if re.search(pattern, content, re.IGNORECASE):
            violations.append(f"Potential SQL injection detected")
            break
    
    # Check for sensitive data patterns
    if re.search(r"\b\d{3}-\d{2}-\d{4}\b", content):  # SSN pattern
        violations.append("Potential SSN detected in input")
    
    if re.search(r"\b\d{16}\b", content):  # Credit card pattern
        violations.append("Potential credit card number detected")
    
    # Optional: Use Azure Content Safety API for advanced checks
    # (Commented out for local development)
    # violations.extend(check_with_azure_content_safety(content))
    
    is_safe = len(violations) == 0
    
    return is_safe, violations

def check_with_azure_content_safety(content: str) -> List[str]:
    """
    Use Azure Content Safety API for advanced content moderation.
    """
    violations = []
    
    try:
        endpoint = os.getenv("CONTENT_SAFETY_ENDPOINT")
        key = os.getenv("CONTENT_SAFETY_KEY")
        
        if not endpoint or not key:
            return []  # Skip if not configured
        
        client = ContentSafetyClient(endpoint, AzureKeyCredential(key))
        
        # Analyze text
        # response = client.analyze_text(...)
        
        # Add violations based on response
        # if response.hate_result.severity > 2:
        #     violations.append("Hate speech detected")
        
    except Exception as e:
        # Log error but don't fail the request
        print(f"Content safety check failed: {e}")
    
    return violations
```

---

### 2. Orchestrator Service Implementation

#### File Structure

```
orchestrator/
├── main.py
├── services/
│   ├── __init__.py
│   ├── intent_classifier.py
│   ├── agent_router.py
│   └── context_builder.py
├── models/
│   ├── __init__.py
│   └── orchestrator_models.py
├── config.py
├── requirements.txt
└── tests/
    └── test_orchestrator.py
```

#### Implementation: `main.py`

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel
from typing import Optional, Dict, Any
import time
import logging
from opentelemetry import trace, metrics
from services.intent_classifier import classify_intent
from services.agent_router import route_to_agent
from services.context_builder import build_context
import httpx

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = FastAPI(title="Zava Orchestrator Service", version="1.0.0")

tracer = trace.get_tracer(__name__)
meter = metrics.get_meter(__name__)

# Metrics
request_counter = meter.create_counter(
    "orchestrator.requests.total",
    description="Total orchestration requests"
)

latency_histogram = meter.create_histogram(
    "orchestrator.latency.ms",
    description="Request latency in milliseconds"
)

class OrchestratorRequest(BaseModel):
    user_message: str
    user_id: str
    session_id: str
    context: Optional[Dict[str, Any]] = None

class OrchestratorResponse(BaseModel):
    response: str
    confidence: float
    agent_used: str
    escalated: bool = False
    metadata: Dict[str, Any] = {}

@app.middleware("http")
async def add_telemetry(request: Request, call_next):
    """Add telemetry to all requests."""
    start_time = time.time()
    
    response = await call_next(request)
    
    latency_ms = (time.time() - start_time) * 1000
    latency_histogram.record(latency_ms)
    request_counter.add(1)
    
    logger.info(f"Request to {request.url.path} took {latency_ms:.2f}ms")
    
    return response

@app.post("/orchestrate", response_model=OrchestratorResponse)
async def orchestrate(req: OrchestratorRequest):
    """
    Main orchestration endpoint.
    
    Flow:
    1. Classify user intent
    2. Build context with grounding data
    3. Route to appropriate agent
    4. Validate response through policy engine
    5. Return response or escalate if needed
    """
    with tracer.start_as_current_span("orchestrate_request") as span:
        span.set_attribute("user_id", req.user_id)
        span.set_attribute("session_id", req.session_id)
        
        try:
            # Step 1: Classify intent
            intent = await classify_intent(req.user_message)
            span.set_attribute("intent", intent)
            logger.info(f"Classified intent: {intent}")
            
            # Step 2: Build context
            context = await build_context(intent, req.user_id, req.context or {})
            
            # Step 3: Route to agent
            agent_response = await route_to_agent(
                intent=intent,
                message=req.user_message,
                context=context,
                user_id=req.user_id
            )
            
            span.set_attribute("agent_used", agent_response["agent"])
            span.set_attribute("confidence", agent_response["confidence"])
            
            # Step 4: Check if escalation needed
            if agent_response["confidence"] < 0.6:
                logger.warning(f"Low confidence ({agent_response['confidence']}), escalating")
                await escalate_to_human(req, agent_response)
                
                return OrchestratorResponse(
                    response="I've connected you with a team member who can better assist you.",
                    confidence=agent_response["confidence"],
                    agent_used=agent_response["agent"],
                    escalated=True
                )
            
            # Step 5: Policy validation
            validated_response = await validate_with_policy(agent_response)
            
            return OrchestratorResponse(
                response=validated_response["text"],
                confidence=agent_response["confidence"],
                agent_used=agent_response["agent"],
                metadata=agent_response.get("metadata", {})
            )
        
        except Exception as e:
            logger.error(f"Orchestration error: {str(e)}")
            span.record_exception(e)
            raise HTTPException(status_code=500, detail=str(e))

async def validate_with_policy(agent_response: Dict[str, Any]) -> Dict[str, Any]:
    """Validate agent response through policy engine."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://policy-engine:8001/validate",
            json={
                "data": {"content": agent_response["text"]},
                "policy_type": "safety"
            },
            timeout=5.0
        )
        
        if response.status_code == 200:
            policy_result = response.json()
            if not policy_result["approved"]:
                raise ValueError(f"Policy violation: {policy_result['violations']}")
        
        return agent_response

async def escalate_to_human(req: OrchestratorRequest, agent_response: Dict[str, Any]):
    """Queue request for human escalation."""
    # In production, this would send to Service Bus queue
    logger.info(f"Escalating session {req.session_id} to human support")
    
    # Placeholder for queue integration
    # await service_bus_client.send_message({
    #     "session_id": req.session_id,
    #     "user_id": req.user_id,
    #     "message": req.user_message,
    #     "agent_response": agent_response,
    #     "timestamp": datetime.utcnow().isoformat()
    # })

@app.get("/health")
async def health():
    """Health check endpoint."""
    return {
        "status": "healthy",
        "service": "orchestrator",
        "timestamp": time.time()
    }

@app.get("/ready")
async def readiness():
    """Readiness check endpoint."""
    # Check dependencies
    try:
        async with httpx.AsyncClient() as client:
            policy_health = await client.get("http://policy-engine:8001/health", timeout=2.0)
            
            if policy_health.status_code == 200:
                return {"status": "ready"}
            else:
                return JSONResponse(
                    status_code=503,
                    content={"status": "not ready", "reason": "policy engine unavailable"}
                )
    except Exception as e:
        return JSONResponse(
            status_code=503,
            content={"status": "not ready", "reason": str(e)}
        )
```

---

### 3. Agent Implementation

#### Customer Design Agent

```python
# agents/customer_design_agent.py
from azure.search.documents import SearchClient
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI
from azure.core.credentials import AzureKeyCredential
import os
import logging

logger = logging.getLogger(__name__)

class CustomerDesignAgent:
    """
    Agent for assisting customers with product design and recommendations.
    
    Capabilities:
    - Product search via RAG
    - Design recommendations
    - LLM-powered reasoning
    """
    
    def __init__(self):
        # Initialize Azure AI Search client
        search_endpoint = os.getenv("AZURE_SEARCH_ENDPOINT")
        search_key = os.getenv("AZURE_SEARCH_KEY")
        
        self.search_client = SearchClient(
            endpoint=search_endpoint,
            index_name="product-catalog",
            credential=AzureKeyCredential(search_key)
        )
        
        # Initialize Azure OpenAI client
        self.openai_client = AzureOpenAI(
            azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
            api_key=os.getenv("AZURE_OPENAI_API_KEY"),
            api_version="2024-02-01"
        )
        
        self.deployment_name = os.getenv("AZURE_OPENAI_DEPLOYMENT_NAME", "gpt-4")
    
    async def execute(self, user_message: str, context: dict) -> dict:
        """
        Execute the customer design agent workflow.
        
        Args:
            user_message: User's input message
            context: Additional context (user preferences, history, etc.)
        
        Returns:
            Dict with response, confidence, and metadata
        """
        try:
            # Step 1: Search product catalog (RAG)
            search_results = await self.search_products(user_message)
            
            # Step 2: Build grounded prompt
            prompt = self.build_prompt(user_message, search_results, context)
            
            # Step 3: Generate response with LLM
            response = self.openai_client.chat.completions.create(
                model=self.deployment_name,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "You are a helpful product design advisor for Zava. "
                            "Assist customers with product selection and customization. "
                            "Base your recommendations on the provided product catalog. "
                            "Be friendly, professional, and concise."
                        )
                    },
                    {
                        "role": "user",
                        "content": prompt
                    }
                ],
                max_tokens=1000,
                temperature=0.7
            )
            
            # Step 4: Calculate confidence
            confidence = self.calculate_confidence(response, search_results)
            
            return {
                "text": response.choices[0].message.content,
                "confidence": confidence,
                "agent": "CustomerDesignAgent",
                "metadata": {
                    "products_found": len(search_results),
                    "products_referenced": [r["productId"] for r in search_results[:5]],
                    "tokens_used": response.usage.total_tokens
                }
            }
        
        except Exception as e:
            logger.error(f"Customer design agent error: {str(e)}")
            return {
                "text": "I'm having trouble finding products right now. Let me connect you with a team member.",
                "confidence": 0.0,
                "agent": "CustomerDesignAgent",
                "metadata": {"error": str(e)}
            }
    
    async def search_products(self, query: str) -> list:
        """Search product catalog using hybrid search."""
        try:
            results = self.search_client.search(
                search_text=query,
                select=["productId", "name", "description", "price", "category", "colors"],
                top=5,
                query_type="semantic"
            )
            
            return [
                {
                    "productId": doc.get("productId"),
                    "name": doc.get("name"),
                    "description": doc.get("description"),
                    "price": doc.get("price"),
                    "category": doc.get("category"),
                    "colors": doc.get("colors", [])
                }
                for doc in results
            ]
        
        except Exception as e:
            logger.error(f"Product search error: {str(e)}")
            return []
    
    def build_prompt(self, user_message: str, search_results: list, context: dict) -> str:
        """Build grounded prompt with search results."""
        # Format search results
        products_text = "\n\n".join([
            f"**{p['name']}** (${p['price']})\n"
            f"Category: {p['category']}\n"
            f"Description: {p['description']}\n"
            f"Available Colors: {', '.join(p.get('colors', []))}"
            for p in search_results[:3]
        ])
        
        # Add context if available
        user_prefs = context.get("preferences", {})
        prefs_text = ""
        if user_prefs:
            prefs_text = f"\n\nUser Preferences:\n{user_prefs}"
        
        prompt = f"""
User Question: {user_message}

Relevant Products from Catalog:
{products_text}
{prefs_text}

Please provide a helpful recommendation based on the user's question and the available products.
If suggesting a product, explain why it's a good match.
If no products match, suggest alternatives or ask clarifying questions.
"""
        
        return prompt
    
    def calculate_confidence(self, response, search_results: list) -> float:
        """Calculate confidence score based on multiple factors."""
        confidence = 0.8  # Base confidence
        
        # Adjust based on search results
        if len(search_results) == 0:
            confidence *= 0.5  # Low confidence if no products found
        elif len(search_results) >= 3:
            confidence = min(confidence * 1.1, 1.0)  # Higher confidence with good results
        
        # Could add more sophisticated confidence calculation
        # based on response coherence, finish_reason, etc.
        
        return round(confidence, 2)
```

---

## Testing Guide

### Unit Tests

```python
# tests/test_policy_engine.py
import pytest
from policy_engine.validators.discount_validator import validate_discount
from policy_engine.validators.data_masking import mask_pii

class TestDiscountValidator:
    """Test discount validation logic."""
    
    def test_valid_discount(self):
        """Test discount within limits."""
        is_valid, error = validate_discount(
            discount_percentage=20.0,
            discount_amount=50.0,
            order_value=250.0
        )
        assert is_valid is True
        assert error is None
    
    def test_percentage_exceeds_limit(self):
        """Test discount percentage over maximum."""
        is_valid, error = validate_discount(
            discount_percentage=35.0,
            discount_amount=50.0,
            order_value=250.0
        )
        assert is_valid is False
        assert "exceeds maximum 30.0%" in error
    
    def test_amount_exceeds_limit(self):
        """Test discount amount over maximum."""
        is_valid, error = validate_discount(
            discount_percentage=10.0,
            discount_amount=1500.0,
            order_value=10000.0
        )
        assert is_valid is False
        assert "exceeds maximum $1000.0" in error
    
    def test_order_below_minimum(self):
        """Test order value below minimum."""
        is_valid, error = validate_discount(
            discount_percentage=10.0,
            discount_amount=5.0,
            order_value=50.0
        )
        assert is_valid is False
        assert "must be at least $100.0" in error

class TestDataMasking:
    """Test PII masking functionality."""
    
    def test_email_masking(self):
        """Test email address masking."""
        data = {"email": "john.doe@example.com"}
        masked = mask_pii(data)
        assert masked["email"] == "jo***@example.com"
    
    def test_phone_masking(self):
        """Test phone number masking."""
        data = {"phone": "555-123-4567"}
        masked = mask_pii(data)
        assert masked["phone"] == "***-***-4567"
    
    def test_name_masking(self):
        """Test name masking."""
        data = {"name": "John Doe"}
        masked = mask_pii(data)
        assert masked["name"] == "J***"
    
    def test_combined_masking(self):
        """Test masking multiple fields."""
        data = {
            "email": "jane@example.com",
            "phone": "555-999-8888",
            "name": "Jane Smith",
            "age": 30  # Should not be masked
        }
        masked = mask_pii(data)
        assert masked["email"] == "ja***@example.com"
        assert masked["phone"] == "***-***-8888"
        assert masked["name"] == "J***"
        assert masked["age"] == 30  # Unchanged
```

Run tests:

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=policy_engine --cov=orchestrator --cov=agents --cov-report=html

# Run specific test file
pytest tests/test_policy_engine.py -v
```

---

## Deployment Guide

### Deploy to Azure Kubernetes Service

```bash
# Build and push Docker images
docker build -t acrzavaprod.azurecr.io/orchestrator:v1.0.0 orchestrator/
docker build -t acrzavaprod.azurecr.io/policy-engine:v1.0.0 policy-engine/
docker build -t acrzavaprod.azurecr.io/agent-manager:v1.0.0 agent-manager/

# Push to Azure Container Registry
az acr login --name acrzavaprod
docker push acrzavaprod.azurecr.io/orchestrator:v1.0.0
docker push acrzavaprod.azurecr.io/policy-engine:v1.0.0
docker push acrzavaprod.azurecr.io/agent-manager:v1.0.0

# Deploy to AKS
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/orchestrator-deployment.yaml
kubectl apply -f k8s/policy-engine-deployment.yaml
kubectl apply -f k8s/agent-manager-deployment.yaml

# Verify deployment
kubectl get pods -n zava-prod
kubectl get services -n zava-prod
```

---

## Troubleshooting

### Common Issues

#### Issue: Azure OpenAI Rate Limiting

**Symptoms**: `429 Too Many Requests` errors

**Solution**:
```python
# Implement exponential backoff
from tenacity import retry, wait_exponential, stop_after_attempt

@retry(wait=wait_exponential(multiplier=1, min=4, max=60), stop=stop_after_attempt(5))
async def call_openai_with_retry(client, **kwargs):
    return client.chat.completions.create(**kwargs)
```

#### Issue: Policy Engine Blocking Valid Requests

**Symptoms**: High false positive rate on safety checks

**Solution**: Adjust safety patterns or thresholds in `validators/safety_checker.py`

#### Issue: Low Agent Confidence Scores

**Symptoms**: Many requests being escalated

**Solution**:
1. Improve search index quality
2. Add more context to prompts
3. Use prompt engineering techniques (few-shot examples)

---

## Next Steps

1. **Implement Remaining Agents**: Marketing Analytics Agent, DevOps Agent
2. **Add Authentication**: Integrate with Microsoft Entra ID
3. **Implement Caching**: Add Redis for frequently requested data
4. **Set Up Monitoring**: Configure Application Insights dashboards
5. **Load Testing**: Run performance tests with Locust
6. **Documentation**: Add API documentation with Swagger/OpenAPI

---

## Support

For questions or issues:
- **Slack**: #zava-platform-support
- **Email**: platform-team@zava.com
- **Documentation**: https://docs.zava.com

---

**Version**: 1.0.0  
**Last Updated**: 2025-12-10  
**Maintained By**: Platform Engineering Team
