# Multi AI Agent Project

About This App

- Lightweight chat application with a Streamlit frontend and FastAPI backend.

- Uses a single LangGraph ReAct agent powered by Groq (ChatGroq) and optional Tavily web search.

- Frontend posts to POST /chat; backend validates the model, builds the agent with your system prompt and search setting, then returns the answer.

- Runs locally or in Docker; CI/CD via Jenkins to AWS ECS Fargate.

Add More Agents

- Define additional agent builders (different prompts/tools/models) in app/core/ai_agent.py.

- Create a simple agent registry (e.g., {"default": ..., "researcher": ..., "analyst": ...}) and select from it at runtime.

- Extend the API request schema to include agent_name and route to the chosen builder in app/backend/api.py.

- Update the Streamlit UI to add an “Agent” dropdown and include agent_name in the payload in app/frontend/ui.py.

- Optional: Use LangGraph to wire multiple agents into a collaboration graph (router/supervisor, debate, or hand‑off patterns).


A lightweight multi‑agent chat system featuring:
- FastAPI backend that hosts an AI agent endpoint.
- Streamlit frontend to configure a system prompt, choose a Groq model, optionally enable web search (Tavily), and chat with the agent.
- LangGraph ReAct agent using Groq LLMs and optional Tavily search tool.
- Jenkins pipeline for CI (SonarQube) and CD to AWS (ECR + ECS Fargate).

![Workflow](work%20flow.jpg)

## Overview

- Frontend (`Streamlit`) calls the backend (`FastAPI`) at `POST /chat`.
- Backend validates requested model and delegates to a ReAct agent built with `langgraph.prebuilt.create_react_agent`.
- Agent uses a Groq LLM (`ChatGroq`) and, when enabled, the Tavily search tool to answer.
- The main entrypoint (`app/main.py`) starts the backend on port `9999` and the frontend on Streamlit (default 8501) in the same container or local machine.

## Repository Structure

```
.
├─ app/
│  ├─ main.py                        # Orchestrates backend (uvicorn) + frontend (streamlit)
│  ├─ backend/
│  │  └─ api.py                      # FastAPI app with /chat endpoint
│  ├─ core/
│  │  └─ ai_agent.py                 # ReAct agent wiring (Groq LLM + Tavily tool)
│  ├─ frontend/
│  │  └─ ui.py                       # Streamlit UI for system prompt/model/search + chat
│  ├─ config/
│  │  └─ settings.py                 # Env + allowed model list
│  ├─ common/
│  │  ├─ custom_exception.py         # Error wrapper with file/line context
│  │  └─ logger.py                   # File logger to logs/log_YYYY-MM-DD.log
│  └─ templates/                     # (none used; UI is Streamlit)
├─ custom_jenkins/
│  └─ Dockerfile                     # Jenkins LTS with Docker (DinD style) for CI
├─ Dockerfile                        # Container for both backend + frontend
├─ Jenkinsfile                       # Jenkins pipeline (SonarQube → build → push → deploy)
├─ FULL_DOCUMENTATION.md             # Expanded platform setup (WSL/Jenkins/AWS)
├─ requirements.txt                  # Python deps
├─ setup.py                          # Editable install
├─ .gitignore                        # Ignore venv, logs, .env
└─ work flow.jpg                     # Architecture diagram
```

## Environment Variables

- `GROQ_API_KEY` (required): API key for Groq models.
- `TAVILY_API_KEY` (optional): enables the Tavily search tool when "Allow web search" is checked.

Set them locally (e.g., `.env`) or in your deployment (Jenkins/ECS task env).

## Installation (Local)

1) Create a virtual environment and activate it, then install:

```bash
pip install -e .
```

2) Create a `.env` with your keys:

```env
GROQ_API_KEY=your_groq_key
TAVILY_API_KEY=your_tavily_key  # optional
```

3) Run the app (spawns backend then frontend):

```bash
python app/main.py
```

- Backend: `http://127.0.0.1:9999`
- Frontend: opens Streamlit on port `8501`

Alternatively, run them separately:

```bash
uvicorn app.backend.api:app --host 127.0.0.1 --port 9999
streamlit run app/frontend/ui.py
```

## Frontend Usage (Streamlit)

- Define a system prompt that shapes your agent’s behavior.
- Select a model (from `settings.ALLOWED_MODEL_NAMES`).
- Optionally enable web search (requires `TAVILY_API_KEY`).
- Enter your query and click "Ask Agent" to see the response.

## Backend API

- `POST /chat`

Request body:
```json
{
  "model_name": "llama3-70b-8192",
  "system_prompt": "You are a helpful assistant",
  "messages": ["What is the capital of France?"],
  "allow_search": false
}
```

Response:
```json
{ "response": "The capital of France is Paris." }
```

Validation:
- `model_name` must be included in `settings.ALLOWED_MODEL_NAMES`.
- `messages` is forwarded to the agent as the conversation state.

## Docker

Build the container:
```bash
docker build -t multi-ai-agent:latest .
```

Run it (exposes frontend 8501 and backend 9999):
```bash
docker run --rm -p 8501:8501 -p 9999:9999 \
  -e GROQ_API_KEY=your_groq_key \
  -e TAVILY_API_KEY=your_tavily_key \
  multi-ai-agent:latest
```

## CI/CD (Jenkins → AWS ECR → ECS Fargate)

Pipeline (`Jenkinsfile`) stages:
- Clone GitHub using credential `github-token`.
- SonarQube analysis using scanner tool `Sonarqube` and `sonarqube-token`.
- Build Docker image and push to AWS ECR using `aws-token`.
- Trigger ECS service rolling deployment (`aws ecs update-service --force-new-deployment`).

Requirements:
- Jenkins agent with Docker (see `custom_jenkins/Dockerfile`).
- Credentials in Jenkins:
  - `github-token`: GitHub PAT.
  - `sonarqube-token`: SonarQube token.
  - `aws-token`: AWS credentials with ECR/ECS permissions.
- AWS: ECR repo and ECS Fargate service & cluster.

See `FULL_DOCUMENTATION.md` for step‑by‑step environment setup (WSL, Jenkins bootstrap, AWS).

## Design Notes

- Agent: `langgraph.prebuilt.create_react_agent(model=ChatGroq(...), tools=[Tavily], state_modifier=system_prompt)`.
- Search tool: added only when the frontend checkbox is selected.
- Logging: all modules use `app.common.logger.get_logger` with daily log file under `logs/`.
- Error handling: `CustomException` augments messages with file and line number.

## Known Limitations

- Assumes `GROQ_API_KEY` (and `TAVILY_API_KEY` if used) are configured in the environment.
- The frontend/agent currently treat `messages` as a simple list; richer multi‑turn structure (roles, timestamps) can be added.
- SonarQube server/tool names in Jenkins must match your instance.

## Future Improvements

- Streaming token responses to the frontend for better UX.
- Expand message schema to support full conversational history and roles.
- Add retries/rate‑limit handling and observability (tracing/metrics).
- Parameterize backend host/port and API URL in the frontend.
- Add health checks and readiness probes in Docker/Kubernetes/ECS.
- Secrets management via AWS Secrets Manager or SSM Parameter Store.

---

For platform details (WSL setup, Jenkins, AWS), consult `FULL_DOCUMENTATION.md`.

