# System Overview

AI Code Reviewer is a two-part application that combines a React/Vite user
interface with a FastAPI backend. It reviews Python, JavaScript, and TypeScript
code, optionally uses an OpenAI-compatible LLM provider for deeper reasoning,
and verifies proposed repairs in a sandbox before presenting them as verified.

## End-to-end workflow

```text
User
  |
  v
React/Vite frontend
  |
  | REST requests and SSE agent events
  v
FastAPI routers
  |
  +--> Static analysis (Ruff, AST, Node syntax/rules)
  +--> LLM review and repair (Groq by default)
  +--> Security review and rule cross-checking
  +--> ChromaDB repository indexing/retrieval
  +--> Local or Docker sandbox execution
  +--> GitHub REST API inspection and PR creation
```

### 1. Single-file review

The user submits source code from the **Single-File Review** tab. The backend
validates the request, runs deterministic static analysis when enabled, and
then sends the code and deterministic findings to the configured LLM. The
response combines static findings, model findings, a summary score, diffs, and
token usage. If no model key is available, the backend can still return
deterministic findings through its fallback path.

### 2. Security review

The security pass can be enabled for a single-file review and is enabled by
default for the repository/RAG review path. Rule-based security matches are
cross-referenced with the LLM's findings so that each security result includes
severity, exploitability, confidence, and whether a rule confirmed it.

### 3. Autonomous fix loop

From a finding, the user can open the **Sandbox Fix Agent**:

1. Use supplied tests, or ask the LLM to generate a focused smoke-test suite.
2. Run the original code against those tests in the sandbox.
3. Ask the LLM for a complete repaired file.
4. Run the repaired file against the same tests.
5. Repeat until tests pass or the retry limit is reached.
6. Return the final code, unified diff, test output, attempt history, and
   verification status to the frontend as Server-Sent Events.

A repair is only marked verified when the sandbox test run passes. The sandbox
uses Docker when enabled and available; otherwise it uses a temporary local
directory with language-specific subprocess execution and timeouts.

### 4. GitHub repository and pull-request workflow

The **GitHub PR Mode** tab parses a repository or pull-request URL, loads
metadata and reviewable files through the GitHub REST API, and lets the user
review selected files. After a fix passes sandbox verification, the backend can
create a uniquely named branch, commit the verified file through the Contents
API, and open a pull request. Creating a PR requires a GitHub token with the
necessary repository permissions and remains a proposal workflow; it does not
merge changes automatically.

### 5. Whole-repository RAG workflow

The **Whole-Repo RAG** tab sends repository files to the backend for indexing.
Python files are split at AST symbols, while JavaScript and TypeScript files
are split using structural patterns. ChromaDB stores the chunks with
deterministic hashed-token embeddings in an in-memory collection. A target file
query retrieves the most relevant cross-file chunks, which are added to the
LLM review context without sending the entire repository in one prompt.

### 6. Benchmarks

The **Benchmarks** tab lists and runs the curated cases in `benchmarks/`.
Each case exercises baseline reproduction and autonomous repair verification.
Results are also available through the benchmark API.

## File and directory responsibilities

### Repository root

| Path | Responsibility |
| --- | --- |
| `README.md` | Project introduction, architecture decisions, setup, deployment, and benchmark notes. |
| `SYSTEM_OVERVIEW.md` | This workflow and file-responsibility reference. |
| `docker-compose.yml` | Runs the backend on port 8000 and the frontend on port 3000. |
| `fly.toml` | Fly.io deployment configuration. |
| `render.yaml` | Render deployment configuration. |
| `vercel.json` | Frontend deployment/routing configuration for Vercel. |
| `.gitignore` | Excludes local environments, secrets, caches, and generated files. |

### Backend

| Path | Responsibility |
| --- | --- |
| `backend/app/main.py` | Creates the FastAPI app, configures CORS, and registers API routers. |
| `backend/app/config.py` | Loads environment settings, provider/model configuration, limits, and sandbox options. |
| `backend/app/models/schemas.py` | Pydantic request and response contracts shared by the API workflows. |
| `backend/app/api/health.py` | Health-check endpoint used by the frontend status indicator. |
| `backend/app/api/review.py` | Single-file review endpoint and review rate limiting. |
| `backend/app/api/agent.py` | Synchronous and SSE streaming endpoints for the autonomous repair agent. |
| `backend/app/api/github.py` | GitHub inspection, file retrieval, and verified-fix PR endpoints. |
| `backend/app/api/rag.py` | Repository indexing, context retrieval, and RAG review endpoints. |
| `backend/app/api/benchmarks.py` | Benchmark listing and execution endpoints. |
| `backend/app/services/static_analysis.py` | Python AST/Ruff checks and JavaScript/TypeScript syntax/rule checks. |
| `backend/app/services/llm_service.py` | LLM review calls, fallback findings, summaries, scores, and diffs. |
| `backend/app/services/security_scanner.py` | Deterministic security rules for supported languages. |
| `backend/app/services/security_service.py` | Dedicated security prompts, finding normalization, and rule cross-referencing. |
| `backend/app/services/agent_loop.py` | Test generation, repair attempts, sandbox orchestration, SSE event data, and diffs. |
| `backend/app/services/sandbox.py` | Docker/local execution, language detection, timeouts, and sandbox results. |
| `backend/app/services/github_service.py` | GitHub URL parsing, REST API calls, file loading, branch/commit creation, and PR opening. |
| `backend/app/services/rag_service.py` | Symbol chunking, deterministic embeddings, ChromaDB indexing, and retrieval. |
| `backend/requirements.txt` | Python runtime and test dependencies. |
| `backend/Dockerfile` | Backend container image definition. |
| `backend/.env.example` | Backend environment-variable template; secrets belong in a local `.env`. |
| `backend/tests/` | Backend endpoint and service tests. |

### Frontend

| Path | Responsibility |
| --- | --- |
| `frontend/src/main.jsx` | React application entry point. |
| `frontend/src/App.jsx` | Main shell, health polling, API-key/model state, and feature-tab routing. |
| `frontend/src/services/api.js` | REST client and SSE client for backend workflows. |
| `frontend/src/components/SingleFileReview.jsx` | Single-file review form and results view. |
| `frontend/src/components/AgentLoopView.jsx` | Agent inputs, live event stream, attempts, diffs, and verification results. |
| `frontend/src/components/GitHubReviewView.jsx` | GitHub target inspection, file selection, review, and PR actions. |
| `frontend/src/components/WholeRepoRAGView.jsx` | Repository indexing, context retrieval, and RAG review UI. |
| `frontend/src/components/BenchmarkView.jsx` | Benchmark listing and execution UI. |
| `frontend/src/components/FindingCard.jsx` | General review finding presentation. |
| `frontend/src/components/SecurityFindingCard.jsx` | Security finding presentation and severity details. |
| `frontend/src/components/DiffViewer.jsx` | Unified diff display. |
| `frontend/src/components/CodeEditor.jsx` | Source and test code editing surface. |
| `frontend/src/components/ApiConfigModal.jsx` | Browser-local LLM key and model configuration. |
| `frontend/src/components/ReviewSummaryBanner.jsx` | Review score and issue summary. |
| `frontend/src/components/AttemptHistoryCard.jsx` | Individual repair-attempt details. |
| `frontend/src/components/AgentTerminal.jsx` | Live agent log/terminal output. |
| `frontend/src/components/FullCodeModal.jsx` | Full source and repaired-code display. |
| `frontend/src/constants/` | Sample snippets and sample RAG data used by the UI. |
| `frontend/src/index.css` | Global styles and Tailwind entry styles. |
| `frontend/index.html` | Vite HTML entry document. |
| `frontend/package.json` | Frontend scripts and dependencies. |
| `frontend/vite.config.js` | Vite development/build configuration. |
| `frontend/tailwind.config.js` | Tailwind theme/content configuration. |
| `frontend/nginx.conf` | Production static-file serving and SPA fallback configuration. |
| `frontend/Dockerfile` | Frontend build and serving image definition. |
| `frontend/.env.example` | Frontend environment-variable template, including `VITE_API_URL`. |

### Benchmarks

| Path | Responsibility |
| --- | --- |
| `benchmarks/dataset.json` | Curated benchmark cases and expected behavior. |
| `benchmarks/dataset.py` | Python access helpers for benchmark data. |
| `benchmarks/evaluate.py` | Runs baseline tests and autonomous repair verification. |
| `benchmarks/eval_results.json` | Checked-in evaluation output. |

## Runtime configuration

Copy the environment templates before running locally:

```powershell
Copy-Item backend\.env.example backend\.env
Copy-Item frontend\.env.example frontend\.env
```

The backend reads provider, model, CORS, rate-limit, and sandbox settings from
`backend/.env`. The frontend uses `VITE_API_URL` when the API is not served
from the same origin. LLM and GitHub tokens are credentials: keep them in
local environment files or browser storage as intended, and never commit them.

## Running the system

For the complete local stack:

```powershell
docker compose up --build
```

Then open `http://localhost:3000`. The FastAPI documentation is available at
`http://localhost:8000/docs`, and the health endpoint is
`http://localhost:8000/health`.

For development, run the backend from `backend/` with Uvicorn and the
frontend from `frontend/` with Vite. The exact commands and provider setup are
documented in `README.md`.
