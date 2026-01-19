# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kagent is a Kubernetes-native framework for building, deploying, and managing AI agents. It's a CNCF project that enables declarative AI agent configuration and orchestration using Kubernetes custom resources.

**Core Components:**
- **Controller** (Go): Kubernetes controller managing Agent, ModelConfig, ToolServer, and RemoteMCPServer CRDs
- **Engine** (Python): Agent runtime using ADK (Agents Declarative Kit)
- **UI** (TypeScript/Next.js): Web interface for agent management
- **CLI** (Go): Command-line tool for local agent management

## Build & Development Commands

### Full Stack (Kind Cluster)
```bash
make create-kind-cluster          # Create local Kind cluster
make use-kind-cluster             # Configure kubectl and set namespace to kagent
make helm-install                 # Build all images and deploy via Helm
make helm-uninstall               # Remove Helm deployment
make kagent-addon-install         # Install observability addons (Istio, Grafana, Prometheus)
```

**Required Environment Variables:**
```bash
export KAGENT_DEFAULT_MODEL_PROVIDER=openAI  # or anthropic, azureOpenAI, gemini, ollama
export OPENAI_API_KEY=your-key               # matching your provider
```

### Go (go/)
```bash
cd go
make test                         # Run unit tests
make lint                         # Run golangci-lint
make lint-fix                     # Auto-fix lint issues
make manifests                    # Generate CRDs and RBAC
make generate                     # Generate DeepCopy methods
make run                          # Run controller locally
go test -race -skip 'TestE2E.*' -v ./...  # Run all unit tests
```

### Python (python/)
```bash
cd python
uv venv .venv && uv sync --all-extras     # Setup environment
uv run pytest tests packages/*/tests       # Run all tests
uv run ruff format                         # Format code
uv run ruff check                          # Lint
make audit                                 # CVE scanning
```

### UI (ui/)
```bash
cd ui
npm install
npm run dev                       # Start dev server on port 8001
npm run test                      # Run Jest unit tests
npm run lint                      # ESLint
npm run test:e2e                  # Cypress E2E tests
```

### Helm Tests
```bash
make helm-test                    # Validate Helm templates
helm unittest helm/kagent         # Run Helm unit tests (requires helm-unittest plugin)
```

## Architecture

```
kagent/
├── go/                          # Go components
│   ├── api/v1alpha2/            # CRD type definitions (current version)
│   ├── cmd/controller/          # Controller entry point
│   ├── internal/
│   │   ├── controller/          # Reconcilers and translators
│   │   ├── a2a/                 # Agent-to-Agent protocol server
│   │   ├── database/            # SQLite/PostgreSQL abstraction
│   │   └── httpserver/          # REST API endpoints
│   ├── cli/                     # CLI implementation
│   └── test/e2e/                # E2E tests and test agents
├── python/
│   └── packages/
│       ├── kagent-adk/          # Core ADK runtime
│       ├── kagent-core/         # Core utilities
│       └── kagent-*/            # Framework integrations
├── ui/src/
│   ├── app/                     # Next.js pages
│   ├── components/              # React components
│   └── lib/                     # Utilities
└── helm/
    ├── kagent/                  # Main chart
    ├── kagent-crds/             # CRD chart
    └── agents/                  # Pre-built agent charts
```

### Key CRDs (v1alpha2)
- **Agent**: System prompt, tools, LLM configuration
- **ModelConfig**: LLM provider settings (OpenAI, Anthropic, Azure, Gemini, Ollama)
- **RemoteMCPServer**: Remote MCP server connections
- **ToolServer** (v1alpha1): Local tool server definitions

### Data Flow
1. User creates Agent CRD → Controller reconciles
2. Controller translates CRD to ADK configuration
3. Engine pod runs agent with configured tools and LLM
4. A2A server (port 8083) handles agent-to-agent communication

## Testing Patterns

**Unit tests** are co-located with source files (`*_test.go`, `test_*.py`, `*.test.ts`).

**E2E tests** require a running Kind cluster:
```bash
# Setup cluster with test fixtures
make create-kind-cluster helm-install-provider push-test-agent push-test-skill
# Run E2E tests
cd go && go test -v ./test/e2e/...
```

## Code Style

- **Go**: Follow golangci-lint rules (`.golangci.yaml`)
- **Python**: Ruff with line-length 120, target Python 3.12+
- **TypeScript**: ESLint with Next.js config

## DCO Requirement

All commits must be signed off:
```bash
make init-git-hooks               # Auto-add signoff via hook
# or manually:
git commit -s -m "message"
```

## Common Tasks

**Port forwarding for local development:**
```bash
kubectl port-forward svc/kagent-ui 8001:8080         # UI
kubectl port-forward svc/kagent-controller 8083:8083 # Controller API
```

**Run agent locally against cluster:**
```bash
kubectl scale -n kagent deployment kagent-controller --replicas 0
# Set KAGENT_A2A_DEBUG_ADDR=localhost:8080 when running controller
```

**Using Postgres instead of SQLite:**
```bash
KAGENT_HELM_EXTRA_ARGS="--set database.type=postgres --set database.postgres.url=postgres://..." make helm-install
```
