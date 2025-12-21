# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Multi-AI Agent application that provides a chat interface powered by Groq LLMs (Llama models) with optional Tavily web search capabilities. The application uses LangGraph to create ReAct agents and exposes them via a FastAPI backend with a Streamlit frontend.

## Development Commands

### Local Development

```bash
# Install dependencies
pip install -e .

# Run the application (starts both backend and frontend)
python app/main.py
```

The application starts two services:
- **Backend (FastAPI)**: Runs on `http://127.0.0.1:9999`
- **Frontend (Streamlit)**: Runs on `http://localhost:8501`

### Environment Setup

Create a `.env` file in the `app/` directory with:
```
GROQ_API_KEY=your_groq_api_key
TAVILY_API_KEY=your_tavily_api_key
```

### Docker

```bash
# Build Docker image
docker build -t multi-ai-agent .

# Run container (requires environment variables)
docker run -p 8501:8501 -p 9999:9999 \
  -e GROQ_API_KEY=your_key \
  -e TAVILY_API_KEY=your_key \
  multi-ai-agent
```

## Architecture

### Application Structure

The application follows a modular architecture with clear separation of concerns:

```
app/
├── main.py              # Application entry point - launches backend and frontend
├── backend/
│   └── api.py          # FastAPI REST API with /chat endpoint
├── core/
│   └── ai_agent.py     # LangGraph ReAct agent creation and invocation
├── frontend/
│   └── ui.py           # Streamlit UI
├── config/
│   └── settings.py     # Environment variables and configuration
└── common/
    ├── logger.py       # Centralized logging setup
    └── custom_exception.py  # Custom exception handling
```

### Request Flow

1. **User Input** → Streamlit UI ([ui.py:22-49](app/frontend/ui.py#L22-L49))
2. **HTTP POST** → FastAPI endpoint `/chat` ([api.py:19-44](app/backend/api.py#L19-L44))
3. **Agent Creation** → LangGraph ReAct agent with ChatGroq LLM ([ai_agent.py:9-29](app/core/ai_agent.py#L9-L29))
4. **Tool Selection** → Conditionally includes TavilySearchResults based on `allow_search` flag
5. **Response Extraction** → Returns last AI message from agent response
6. **Display** → Streamlit renders markdown response

### Key Components

**AI Agent ([ai_agent.py](app/core/ai_agent.py)):**
- Uses `create_react_agent` from LangGraph to build ReAct agents
- Conditionally adds Tavily search tool based on user preference
- Supports system prompts via `state_modifier` parameter
- Extracts final AI response from message history

**Backend API ([api.py](app/backend/api.py)):**
- Single `/chat` POST endpoint
- Validates model names against allowed list in settings
- Returns JSON response with `{"response": "..."}`

**Frontend ([ui.py](app/frontend/ui.py)):**
- Streamlit interface with model selection, system prompt, and query input
- Checkbox to enable/disable web search
- Hardcoded backend URL: `http://127.0.0.1:9999/chat`

**Configuration ([settings.py](app/config/settings.py)):**
- Centralized settings with environment variable loading
- `ALLOWED_MODEL_NAMES`: `["llama3-70b-8192", "llama-3.3-70b-versatile"]`

### Multi-Threading Architecture

The [main.py](app/main.py) file uses Python threading to run backend and frontend concurrently:
- Backend runs in a separate thread via `uvicorn`
- Frontend runs in the main thread via `streamlit run`
- 2-second delay ensures backend is ready before frontend starts

### Logging

- All modules use centralized logger from [common/logger.py](app/common/logger.py)
- Logs written to `logs/log_YYYY-MM-DD.log`
- Log format: `%(asctime)s - %(levelname)s - %(message)s`

## CI/CD Pipeline

This project uses Jenkins for CI/CD with deployment to AWS ECS Fargate. The pipeline is defined in [Jenkinsfile](Jenkinsfile).

### Pipeline Stages

1. **Checkout**: Clones repository from GitHub
2. **SonarQube Analysis**: Code quality scanning
3. **Build & Push**: Builds Docker image and pushes to AWS ECR
4. **Deploy**: Updates ECS Fargate service with new image

### Required Credentials in Jenkins

- `github-token`: GitHub personal access token
- `sonarqube-token`: SonarQube authentication token
- `aws-token`: AWS credentials (Access Key ID + Secret Access Key)

### Important Configuration

- **ECS Cluster**: Update cluster name in [Jenkinsfile:63](Jenkinsfile#L63)
- **ECS Service**: Update service name in [Jenkinsfile:64](Jenkinsfile#L64)
- **ECR Repository**: Set `ECR_REPO` environment variable in [Jenkinsfile:8](Jenkinsfile#L8)
- **AWS Region**: Set `AWS_REGION` in [Jenkinsfile:7](Jenkinsfile#L7)

## Adding New Models

To add a new Groq model:

1. Update `ALLOWED_MODEL_NAMES` in [settings.py:10-13](app/config/settings.py#L10-L13)
2. Ensure the model name matches Groq's API model identifiers

## Adding New Tools

To add LangChain tools beyond Tavily search:

1. Import the tool in [ai_agent.py](app/core/ai_agent.py)
2. Add conditional logic in `get_response_from_ai_agents()` to include the tool
3. Update the UI in [ui.py](app/frontend/ui.py) to expose tool configuration options
4. Update the `RequestState` schema in [api.py:13-17](app/backend/api.py#L13-L17) if needed

## Deployment Notes

- The application exposes ports **8501** (Streamlit) and **9999** (FastAPI)
- Environment variables must be set in ECS Task Definition or passed at runtime
- Do not commit `.env` files (already in `.gitignore`)
- The frontend hardcodes the backend URL to `127.0.0.1:9999` - this works in Docker since both services run in the same container
