# Lab 04: Advanced RAG Agent with Strands SDK

This lab demonstrates how to build an advanced RAG (Retrieval-Augmented Generation) agent using the **Strands Agents SDK** - a modern, simplified approach to building AI agents on Amazon Bedrock.

## Overview

This lab provides two implementations of the same RAG agent:

1. **Traditional Approach** (`bedrock-swagger-agent-w-action-group.ipynb`) - Uses boto3 Bedrock Agent API with Lambda functions
2. **Strands SDK Approach** (`bedrock-strands-agent-w-knowledge-base.ipynb`) - Uses Strands Agents SDK (RECOMMENDED)

Both implementations create an agent that can:
- Search Swagger API documentation stored in a Knowledge Base
- Generate UML flow diagrams from OpenAPI specifications
- Generate functional test code in various programming languages

## Why Use Strands SDK?

The Strands SDK offers significant advantages over the traditional boto3 approach:

### Comparison

| Feature | Traditional Boto3 | Strands SDK |
|---------|------------------|-------------|
| **Lines of Code** | ~500 lines | ~100 lines |
| **Infrastructure** | Lambda, IAM, Action Groups | None required |
| **Agent Creation** | Multi-step API calls | `Agent(tools=[...])` |
| **Tool Definition** | Lambda functions + schemas | Python functions with `@tool` |
| **Invocation** | Event stream parsing | `agent(query)` |
| **Testing** | Deploy to Lambda | Test locally |
| **Iteration Speed** | Deploy + wait | Immediate |

### Key Benefits

1. **90% Less Code**: Dramatically simplified implementation
2. **No Infrastructure Management**: No Lambda functions, IAM roles, or permissions
3. **Pythonic**: Tools are just decorated Python functions
4. **Fast Iteration**: Test changes immediately without deployment
5. **Built-in Features**: Conversation memory, streaming, and tracing included
6. **Flexibility**: Easy to switch between model providers

## Files

### Strands SDK Implementation (NEW)

- **`bedrock-strands-agent-w-knowledge-base.ipynb`** - Main notebook using Strands SDK
- **`strands_agent_tools.py`** - Strands tools implementation (replaces Lambda)
- **`utility.py`** - Shared utility functions for IAM and OpenSearch setup

### Traditional Implementation

- **`bedrock-swagger-agent-w-action-group.ipynb`** - Original boto3 implementation
- Uses Lambda function from `../lab00-setup/lambda_function.py`

## Prerequisites

Before running this lab, you must complete:

1. **Lab 00: Setup** (`lab00-setup/workshop_setup.ipynb`)
   - Sets up S3 bucket, uploads data, creates IAM roles
   - Creates OpenSearch Serverless collection
   - Deploys Lambda function (for traditional approach only)

## Getting Started with Strands SDK

### 1. Install Dependencies

```bash
pip install strands-agents strands-agents-tools
```

### 2. Run the Notebook

Open and run `bedrock-strands-agent-w-knowledge-base.ipynb`:

```bash
jupyter lab bedrock-strands-agent-w-knowledge-base.ipynb
```

### 3. Key Steps in the Notebook

The notebook follows these steps:

1. **Install Strands SDK**
   ```python
   !pip install -q strands-agents strands-agents-tools
   ```

2. **Load Parameters** from previous labs (bucket, KB config, etc.)

3. **Create Knowledge Base** (same as traditional approach)
   - Creates OpenSearch Serverless index
   - Creates Bedrock Knowledge Base
   - Ingests Swagger API documentation

4. **Create Strands Agent** (simplified!)
   ```python
   from strands import Agent
   from strands_agent_tools import get_uml_diagram, get_unit_test_code, search_knowledge_base

   agent = Agent(
       tools=[search_swagger_docs, get_uml_diagram, get_unit_test_code],
       model="bedrock:anthropic.claude-3-sonnet-20240229-v1:0",
       instructions="..."
   )
   ```

5. **Invoke Agent** (one line!)
   ```python
   response = agent("Generate a UML diagram for the petstore API")
   ```

## Tool Implementations

The `strands_agent_tools.py` file provides three tools:

### 1. `search_knowledge_base`
Searches the Swagger API documentation Knowledge Base to retrieve relevant OpenAPI YAML content.

```python
@tool
def search_knowledge_base(query: str, kb_id: str, max_results: int = 5) -> str:
    """Search the KB and return relevant documents"""
    ...
```

### 2. `get_uml_diagram`
Generates UML sequence diagrams from OpenAPI specifications using Claude and PlantUML.

```python
@tool
def get_uml_diagram(yml_body: str, output_format: str = "png") -> str:
    """Generate UML diagram and upload to S3"""
    ...
```

### 3. `get_unit_test_code`
Generates functional test code from OpenAPI specifications.

```python
@tool
def get_unit_test_code(yml_body: str, user_query: str) -> str:
    """Generate test code based on API spec"""
    ...
```

## Usage Examples

### Example 1: Search Documentation
```python
query = "How do I add a new pet to the petstore API?"
response = agent(query)
```

### Example 2: Generate UML Diagram
```python
query = "Can you generate a UML diagram for the bookstore API?"
response = agent(query)
# Returns PlantUML code and S3 URI of the generated diagram
```

### Example 3: Generate Test Code
```python
query = "Generate Python code to test adding a pet to the petstore"
response = agent(query)
# Returns Python test code using requests library
```

### Example 4: Multi-turn Conversation
```python
# First turn
response = agent("What endpoints are available in the bookstore API?")

# Follow-up (agent remembers context)
response = agent("Can you show me how to use the first one in Python?")
```

## Architecture

### Strands SDK Architecture

```
┌─────────────────┐
│   User Query    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│    Strands Agent            │
│  - Claude 3 Sonnet          │
│  - Tool orchestration       │
│  - Conversation memory      │
└────────┬────────────────────┘
         │
         ├──────────────────┬──────────────────┬────────────────────┐
         ▼                  ▼                  ▼                    ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────────────┐
│ search_kb        │ │ get_uml      │ │ get_test     │ │ Amazon Bedrock     │
│ (Python func)    │ │ (Python func)│ │ (Python func)│ │ - Claude Sonnet    │
└─────────┬────────┘ └──────┬───────┘ └──────┬───────┘ │ - Titan Embeddings │
          │                 │                │         └────────────────────┘
          ▼                 ▼                ▼
┌─────────────────────────────────────────────────┐
│         Amazon Bedrock Knowledge Base            │
│  - OpenSearch Serverless                        │
│  - Swagger API Documentation                    │
└─────────────────────────────────────────────────┘
```

### Traditional Boto3 Architecture (for comparison)

```
┌─────────────────┐
│   User Query    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│ bedrock_agent_runtime.      │
│ invoke_agent()              │
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│    Bedrock Agent            │
│  - Agent ID + Alias         │
│  - Action Groups            │
└────────┬────────────────────┘
         │
         ├──────────────────┬──────────────────┐
         ▼                  ▼                  ▼
┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐
│ Lambda Function  │ │ Lambda       │ │ Knowledge Base   │
│ get_uml_diagram  │ │ get_test_code│ │ Association      │
└──────────────────┘ └──────────────┘ └──────────────────┘
```

## Cost Considerations

### Strands SDK Approach
- **Bedrock Model Invocation**: Pay per token (Claude Sonnet)
- **Knowledge Base**: Storage + query costs
- **S3**: Diagram storage (minimal)
- **No Lambda costs**: Tools run in-process

### Traditional Approach
- Same Bedrock and KB costs
- **Additional Lambda costs**: Invocation + execution time
- **Additional complexity**: IAM, VPC, monitoring

**Estimated savings**: 20-30% lower costs with Strands SDK

## Troubleshooting

### Issue: "Module 'strands' not found"
**Solution**: Install the SDK:
```bash
pip install strands-agents strands-agents-tools
```

### Issue: "Knowledge Base not found"
**Solution**: Ensure you've run `lab00-setup/workshop_setup.ipynb` first

### Issue: Tools not being called
**Solution**: Check tool descriptions are clear and specific. The LLM uses these to decide when to call tools.

### Issue: PlantUML diagrams not generating
**Solution**: Ensure internet access to `http://www.plantuml.com/plantuml` or deploy a local PlantUML server

## Migration Guide

### Converting from Traditional to Strands

If you have an existing boto3 Bedrock Agent implementation:

1. **Extract Lambda function logic** into Python functions
2. **Add `@tool` decorator** to each function
3. **Remove Lambda deployment** code
4. **Replace agent creation** with `Agent(tools=[...])`
5. **Simplify invocation** from event stream parsing to `agent(query)`

See the two notebooks in this directory for side-by-side comparison.

## Additional Resources

- [Strands Agents Documentation](https://strandsagents.com/latest/documentation/docs/)
- [Strands SDK GitHub](https://github.com/strands-agents/sdk-python)
- [AWS Bedrock AgentCore](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/)
- [AWS Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)

## Next Steps

1. **Experiment with different models**: Try Claude 4, or other Bedrock models
2. **Add custom tools**: Create tools specific to your use case
3. **Deploy to production**: Use Bedrock AgentCore Runtime for serverless deployment
4. **Add monitoring**: Leverage Strands' built-in tracing and observability
5. **Multi-agent systems**: Combine multiple Strands agents for complex workflows

## Support

For issues or questions:
- Strands SDK: [GitHub Issues](https://github.com/strands-agents/sdk-python/issues)
- This workshop: [Workshop Issues](https://github.com/amitkalawat/aws-genai-rag-workshop/issues)

## License

This lab is part of the AWS GenAI RAG Workshop. See the main repository for license information.
