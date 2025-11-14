# Front Door (FD2) Video Tutorial Scripts

## Episode FD-01: Introduction & Core Concepts (8 minutes)

### Opening [00:00-00:20]
**Visual**: Front Door logo animation with gateway visualization
**Script**: "Welcome to the Front Door tutorial series! Front Door is your intelligent API gateway - the single entry point that dynamically routes, secures, and manages all your AI services. Let's explore how it revolutionizes service management."

### What is Front Door? [00:20-01:30]
**Visual**: Architecture diagram showing Front Door as central gateway
**Script**: "Front Door v2, or FD2, is an enterprise-grade modular API gateway. It serves as the intelligent front door for dynamic service discovery and routing. Think of it as a smart traffic controller that knows exactly where to send each request."

**Key Features**:
- Dynamic module loading
- Intelligent request routing
- Security enforcement
- Service discovery
- Observability & monitoring
- Auto-scaling support

### The Problem It Solves [01:30-02:30]
**Visual**: Comparison of architectures with/without Front Door
**Script**: "Without Front Door, you face:
- Multiple endpoints to manage
- Inconsistent security policies
- No centralized monitoring
- Manual service discovery
- Complex client configurations

Front Door provides:
- Single entry point
- Unified security layer
- Centralized observability
- Automatic service discovery
- Simple client integration"

### Core Architecture [02:30-04:30]
**Visual**: Detailed component diagram
**Script**: "Let's explore Front Door's architecture:

1. **Request Handler**: Receives and validates incoming requests
2. **Module Manager**: Dynamically loads and manages modules
3. **Control Tower Client**: Fetches configurations and manifests
4. **Security Layer**: JWT validation and authorization
5. **Routing Engine**: Intelligent request routing
6. **Module Pool**: Cached module instances
7. **Observability Layer**: Metrics, logging, tracing"

### Request Flow [04:30-06:00]
**Visual**: Animated request flow through Front Door
**Script**: "Here's how a request flows through Front Door:

1. Client sends request to Front Door
2. Security layer validates JWT token
3. Router identifies target module from path
4. Module Manager loads/retrieves module
5. Module processes the request
6. Response flows back to client

All this happens in milliseconds with caching and optimization."

### Module Interface [06:00-07:00]
**Visual**: Code structure of BaseModule
**Script**: "Every module implements the BaseModule interface:"

```python
class BaseModule:
    async def initialize(self, config: ModuleConfig)
    async def handle_request(self, request: ModuleRequest)
    async def health_check()
    async def shutdown()
```

"This standardization enables dynamic loading and consistent behavior."

### Quick Demo [07:00-07:40]
**Visual**: Terminal showing Front Door in action
```bash
# Start Front Door
python -m src.main

# Send request through Front Door
curl http://localhost:8080/my-project/inference/v1/chat/completions \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"model": "gpt-4", "messages": [...]}'
```

### Closing [07:40-08:00]
**Visual**: Next episode preview
**Script**: "You've learned how Front Door acts as an intelligent gateway for your AI services. Next episode: deep dive into the module system and dynamic loading. See you there!"

---

## Episode FD-02: Module System & Dynamic Loading (12 minutes)

### Opening [00:00-00:20]
**Visual**: Module ecosystem diagram
**Script**: "Welcome back! Today we're diving deep into Front Door's module system - the magic that enables dynamic, pluggable functionality."

### Understanding Modules [00:20-01:30]
**Visual**: Module concept visualization
**Script**: "Modules are self-contained units of functionality. Each module handles specific types of requests - inference, RAG, data processing, etc. They're loaded on-demand and cached for performance."

**Module Types**:
- Inference modules (OpenAI, custom LLMs)
- RAG modules (retrieval & generation)
- Data processing modules
- Security modules
- Custom business logic

### Module Lifecycle [01:30-03:00]
**Visual**: Lifecycle state diagram
**Script**: "Let's understand the module lifecycle:

1. **Discovery**: Module identified from manifest
2. **Loading**: Module class imported dynamically
3. **Initialization**: Configuration applied
4. **Active**: Handling requests
5. **Idle**: Cached but not active
6. **Shutdown**: Cleanup and removal"

### Creating a Custom Module [03:00-06:00]
**Visual**: Live coding session
**Script**: "Let's build a custom module from scratch:"

```python
from src.core.module_interface import BaseModule, ModuleConfig, ModuleRequest, ModuleResponse

class SentimentAnalysisModule(BaseModule):
    async def initialize(self, config: ModuleConfig) -> None:
        await super().initialize(config)
        # Load ML model
        self.model = await self.load_model(config.model_path)
        self.logger.info("Sentiment module initialized")
    
    async def handle_request(self, request: ModuleRequest) -> ModuleResponse:
        try:
            # Extract text from request
            text = request.body.get("text")
            
            # Perform sentiment analysis
            sentiment = await self.analyze_sentiment(text)
            
            return ModuleResponse(
                status_code=200,
                body={"sentiment": sentiment, "confidence": 0.95}
            )
        except Exception as e:
            return ModuleResponse(
                status_code=500,
                body={"error": str(e)}
            )
    
    async def health_check(self) -> Dict[str, Any]:
        return {
            "status": "healthy",
            "model_loaded": self.model is not None
        }
```

### Module Configuration [06:00-08:00]
**Visual**: Configuration examples
**Script**: "Modules are configured through manifests in Control Tower:"

```json
{
  "module_type": "sentiment_analysis",
  "runtime": {
    "type": "python:3.11",
    "implementation": "src.modules.sentiment.SentimentAnalysisModule"
  },
  "endpoints": {
    "dev": {"primary": "http://dev-sentiment:8080"},
    "prod": {"primary": "https://prod-sentiment"}
  },
  "configuration_references": [
    {
      "name": "model_path",
      "source": "vault://models/sentiment-v2",
      "required": true
    }
  ]
}
```

### Dynamic Loading Mechanism [08:00-10:00]
**Visual**: Loading flow diagram
**Script**: "Front Door's dynamic loading is powerful:"

```python
# Module Manager loads modules dynamically
async def load_module(self, module_config: ModuleConfig):
    # Import module class
    module_class = self.import_module_class(
        module_config.runtime.implementation
    )
    
    # Create instance
    module_instance = module_class()
    
    # Initialize with config
    await module_instance.initialize(module_config)
    
    # Add to pool
    self.module_pool[module_config.name] = module_instance
    
    return module_instance
```

**Features**:
- Lazy loading on first request
- Module pooling for performance
- Hot-reload capability
- Graceful module updates

### Module Pool Management [10:00-11:30]
**Visual**: Pool management visualization
**Script**: "The module pool optimizes performance:

- **Pool Size**: Configurable max modules in memory
- **LRU Eviction**: Least recently used modules removed
- **Preloading**: Critical modules loaded at startup
- **Health Monitoring**: Unhealthy modules auto-reloaded"

### Closing [11:30-12:00]
**Visual**: Module development checklist
**Script**: "You've mastered Front Door's module system! You can now create custom modules, configure them via manifests, and leverage dynamic loading. Next: Request routing and service discovery!"

---

## Episode FD-03: Request Routing & Service Discovery (10 minutes)

### Opening [00:00-00:20]
**Visual**: Network routing visualization
**Script**: "Welcome to episode 3! Today we'll explore how Front Door intelligently routes requests and discovers services - the core of its gateway functionality."

### Routing Patterns [00:20-02:00]
**Visual**: Three routing pattern examples
**Script**: "Front Door supports three routing patterns:

1. **Path-based Routing** (Recommended):
   ```
   /{project}/{module}/endpoint
   Example: /ai-platform/inference/v1/chat
   ```

2. **Header-based Routing**:
   ```
   X-Project-Module: ai-platform/inference
   Path: /v1/chat
   ```

3. **Subdomain-based Routing**:
   ```
   ai-platform-inference.api.company.com/v1/chat
   ```"

### Service Discovery [02:00-03:30]
**Visual**: Service discovery flow
**Script**: "Front Door discovers services through Control Tower:

1. Fetches project manifest
2. Identifies available modules
3. Resolves endpoint URLs
4. Caches service mappings
5. Updates on manifest changes

This enables zero-downtime updates and dynamic scaling."

### Routing Engine Deep Dive [03:30-05:30]
**Visual**: Code walkthrough
**Script**: "Let's examine the routing engine:"

```python
class RoutingEngine:
    async def route_request(self, request: Request):
        # Extract routing info
        project, module = self.extract_route_info(request)
        
        # Get module config from Control Tower
        module_config = await self.control_tower.get_module(
            project, module
        )
        
        # Select endpoint based on environment
        endpoint = self.select_endpoint(
            module_config, 
            request.headers.get("X-Environment")
        )
        
        # Load balancing if multiple endpoints
        if isinstance(endpoint, list):
            endpoint = self.load_balancer.select(endpoint)
        
        return endpoint
```

### Load Balancing Strategies [05:30-07:00]
**Visual**: Load balancing algorithms
**Script**: "Front Door supports multiple load balancing strategies:

- **Round Robin**: Requests distributed evenly
- **Least Connections**: Route to least busy server
- **Weighted**: Based on server capacity
- **Consistent Hashing**: Same client to same server
- **Health-based**: Only to healthy endpoints"

### Failover & Resilience [07:00-08:30]
**Visual**: Failover scenario animation
**Script**: "Front Door ensures high availability:

1. **Health Checks**: Regular endpoint monitoring
2. **Circuit Breakers**: Prevent cascade failures
3. **Retry Logic**: Automatic retry with backoff
4. **Fallback Routes**: Secondary endpoints
5. **Graceful Degradation**: Partial service availability"

```python
# Circuit breaker example
@circuit_breaker(failure_threshold=5, timeout=30)
async def call_service(self, endpoint, request):
    try:
        response = await self.http_client.post(endpoint, data=request)
        return response
    except Exception as e:
        # Circuit opens after 5 failures
        return self.fallback_response()
```

### Practical Examples [08:30-09:30]
**Visual**: Real-world routing scenarios
**Script**: "Let's see routing in action:

```bash
# Inference request
curl http://localhost:8080/customer-service/llm/v1/completions

# RAG query
curl http://localhost:8080/knowledge-base/rag/search

# Custom module
curl http://localhost:8080/analytics/sentiment/analyze
```

Each request is intelligently routed based on project and module."

### Closing [09:30-10:00]
**Visual**: Routing best practices
**Script**: "You've mastered request routing and service discovery! You understand patterns, load balancing, and resilience. Next episode: APISIX integration for advanced gateway features!"

---

## Episode FD-04: APISIX Integration & Observability (10 minutes)

### Opening [00:00-00:20]
**Visual**: APISIX and observability logos
**Script**: "Welcome to our final Front Door episode! Today we'll explore APISIX integration for enterprise features and comprehensive observability."

### Why APISIX? [00:20-01:30]
**Visual**: APISIX features overview
**Script**: "APISIX supercharges Front Door with:

- **Advanced Rate Limiting**: Per-user, per-IP, custom rules
- **Authentication Plugins**: JWT, OAuth2, API Key, mTLS
- **Traffic Management**: Canary deployments, A/B testing
- **Observability**: Prometheus, Zipkin, SkyWalking
- **Security**: WAF, IP restrictions, CORS
- **Performance**: Response caching, compression"

### Integration Architecture [01:30-03:00]
**Visual**: Integration diagram
**Script**: "Front Door integrates with APISIX:

1. Front Door reads manifests from Control Tower
2. Automatically configures APISIX routes
3. APISIX handles edge concerns
4. Front Door focuses on business logic
5. Unified monitoring and logging"

### APISIX Configuration [03:00-05:00]
**Visual**: Configuration examples
**Script**: "Let's configure APISIX with Front Door:"

```python
# Front Door APISIX client
class APISIXClient:
    async def sync_routes(self, manifest):
        for module in manifest.modules:
            route = {
                "uri": f"/{manifest.project_id}/{module.name}/*",
                "upstream": {
                    "type": "roundrobin",
                    "nodes": module.endpoints
                },
                "plugins": {
                    "jwt-auth": {"key": "user-key"},
                    "limit-req": {
                        "rate": 100,
                        "burst": 50
                    },
                    "prometheus": {}
                }
            }
            await self.create_route(route)
```

### Observability Stack [05:00-07:00]
**Visual**: Monitoring dashboard
**Script**: "Front Door provides comprehensive observability:

**Metrics** (Prometheus):
- Request rate and latency
- Error rates and status codes
- Module performance
- Resource utilization

**Tracing** (Jaeger/Zipkin):
- End-to-end request flow
- Service dependencies
- Latency breakdown
- Error propagation

**Logging** (ELK Stack):
- Structured JSON logs
- Correlation IDs
- Request/response bodies
- Error stack traces"

### Langfuse Integration [07:00-08:30]
**Visual**: Langfuse dashboard
**Script**: "For LLM observability, we integrate Langfuse:

```python
# Langfuse tracking
from langfuse import Langfuse

class LLMModule(BaseModule):
    def __init__(self):
        self.langfuse = Langfuse()
    
    async def handle_request(self, request):
        # Start trace
        trace = self.langfuse.trace(
            name="llm-inference",
            metadata={"model": request.model}
        )
        
        # Track generation
        generation = trace.generation(
            name="completion",
            input=request.messages,
            model=request.model
        )
        
        # Get LLM response
        response = await self.llm.complete(request)
        
        # End tracking
        generation.end(output=response)
        
        return response
```

This provides:
- Token usage tracking
- Cost monitoring
- Quality metrics
- Performance analysis"

### Production Best Practices [08:30-09:30]
**Visual**: Best practices checklist
**Script**: "For production deployments:

1. **Security**:
   - Enable mTLS for service communication
   - Rotate JWT secrets regularly
   - Implement rate limiting
   - Use WAF rules

2. **Performance**:
   - Enable response caching
   - Use connection pooling
   - Implement request compression
   - Optimize module pool size

3. **Reliability**:
   - Configure health checks
   - Set up alerts
   - Implement circuit breakers
   - Plan disaster recovery"

### Closing [09:30-10:00]
**Visual**: Front Door series summary
**Script**: "Congratulations! You've mastered Front Door from basics to advanced integrations. You can now deploy a production-ready API gateway with enterprise features. Next series: JWT Service for authentication!"

## Quick Reference

### Docker Compose Setup
```yaml
version: '3.8'
services:
  front-door:
    image: dsp-fd2:latest
    ports:
      - "8080:8080"
    environment:
      - CONTROL_TOWER_URL=http://control-tower:8000
      - JWT_SERVICE_URL=http://jwt-service:5000
    depends_on:
      - control-tower
      - jwt-service
      
  apisix:
    image: apache/apisix:latest
    ports:
      - "9080:9080"
    volumes:
      - ./apisix.yaml:/usr/local/apisix/conf/apisix.yaml
```

### Key Commands
```bash
# Start Front Door
docker-compose up -d

# Check health
curl http://localhost:8080/health

# View metrics
curl http://localhost:8080/metrics

# Test routing
curl http://localhost:8080/{project}/{module}/endpoint
```

### Configuration Files
- `config.yaml` - Front Door configuration
- `apisix.yaml` - APISIX configuration
- `.env` - Environment variables
- `docker-compose.yml` - Container orchestration
