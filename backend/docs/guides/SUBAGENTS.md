# Backend Agent Guide

This guide explains the specialized backend agent roles for the `groupscout` project. The source-of-truth role specs live in the backend source repo under `.claude/agents/`; the coordinator repo also carries a repo-local Codex plugin bundle at `backend/plugins/groupscout-agents/` derived from those specs.

## Available Agents

Use these role names when delegating work or selecting the matching Codex skill from `backend/plugins/groupscout-agents/skills/`.

### 1. TDD Tester (`tdd-tester`)
- **Focus:** Test-Driven Development and high coverage.
- **Area:** `*_test.go`, `internal/`
- **Responsibilities:**
  - Create reproduction tests for bugs.
  - Draft test specifications for new features.
  - Refactor existing tests for better clarity and performance.
  - Ensure all tests pass before submitting changes.

### 2. Database Architect (`database-architect`)
- **Focus:** SQL migrations and storage layer optimization.
- **Area:** `migrations/`, `internal/storage/`
- **Responsibilities:**
  - Create and manage SQL migrations in the `migrations/` directory.
  - Optimize database queries and indexing strategies in `internal/storage/`.
  - Ensure data integrity and proper error handling in DB operations.
  - Handle database connection lifecycle and pooling configuration.

### 3. Collector Specialist (`collector-specialist`)
- **Focus:** Data ingestion logic and scraping.
- **Area:** `internal/collector/`
- **Responsibilities:**
  - Implement new data collectors for external APIs or websites.
  - Handle rate limiting, retries, and error parsing for external sources.
  - Maintain existing collectors in `internal/collector/news`, `events`, and `permits`.
  - Ensure collectors adhere to the `internal/collector.Collector` interface.

### 4. Enrichment Processor (`enrichment-processor`)
- **Focus:** Data transformation and AI processing.
- **Area:** `internal/enrichment/`, `internal/ollama/`
- **Responsibilities:**
  - Develop and refine data enrichment logic.
  - Manage LLM prompts and model files in `internal/ollama/modelfile`.
  - Implement scoring and extraction logic (e.g., lead scoring, disruption alerts).
  - Optimize the flow between raw data ingestion and enriched output.

### 5. API Integrator (`api-integrator`)
- **Focus:** API design, middleware, and service integration.
- **Area:** `internal/api/`, `internal/alert/`, `cmd/`
- **Responsibilities:**
  - Implement and maintain API handlers in `internal/api/`.
  - Orchestrate logic between `server` and `alertd` daemons.
  - Manage lead notification logic in `internal/leadnotify/` and alertd notification logic in `internal/alert/`.
  - Ensure robust request validation, error handling, and Slack command integration.

### 6. Infrastructure Specialist (`infrastructure-specialist`)
- **Focus:** Docker, Makefile, and deployment.
- **Area:** `Dockerfile`, `Makefile`, `scripts/`
- **Responsibilities:**
  - Maintain and optimize the Docker environment and `Makefile` commands.
  - Manage project configuration files and environment variable templates.
  - Automate build and deployment scripts in the `scripts/` directory.
  - Ensure system reliability and scalability across different deployment options.

## How to Use

When using Claude Code, refer to the relevant `.claude/agents/<agent>.md` spec. When using Codex, use the matching skill in `backend/plugins/groupscout-agents/skills/` when that plugin is installed for the workspace.

### 1. General Usage
Start from the source-of-truth `.claude/agents/` spec or the matching Codex skill, then keep the task scope limited to that agent's area.

### 2. Delegating Tasks
Ask the active coding assistant to use the named agent role for a specific task. For Codex, prefer the repo-local skill name when available, such as `tdd-tester`, `database-architect`, or `api-integrator`.

### 3. Combining Subagents (Choreography)
For complex tasks, you might use different subagents sequentially. This is called "Agent Choreography." For example, to implement the Audit Trail feature (Phase 27), you would:
1. Use `database-architect` to create a new migration and storage layer.
2. Use `collector-specialist` to update scrapers to capture raw data.
3. Use `enrichment-processor` to link raw input to the generated leads.
4. Use `api-integrator` to implement the API/CLI for manual verification.
5. Use `tdd-tester` at each step to ensure everything is properly tested.

See [AGENT_CHOREOGRAPHY_PHASE27.md](../planning/AGENT_CHOREOGRAPHY_PHASE27.md) for a detailed execution plan of this workflow.

#### 💡 Best Practices for Choreography:
- **Small Contexts:** Only give each agent the information relevant to its specific task.
- **Explicit Hand-offs:** Define clear deliverables for each agent (e.g., `database-architect` delivers a schema, `api-integrator` consumes it).
- **Iterative Testing:** Use `tdd-tester` to verify each component in isolation before moving to the next stage of the choreography.
- **Central Planning:** The main coding agent should maintain the high-level plan while specialized roles handle domain-specific implementation details.

## Adding or Updating Agents

Agent definitions are maintained in the backend source repo under `.claude/agents/`. The repo-local Codex plugin under `backend/plugins/groupscout-agents/` is generated/curated from those specs and should be checked for drift when agent instructions change.

### Claude Agent File Format:
```yaml
---
name: "agent-name"
description: "Brief description of the agent's role."
tools: ["Read", "Bash", "Glob", "Grep", "Write", "Edit", "WebSearch", "AskUserQuestion", "update_status"]
---
# Agent Name

Detailed instructions for the agent...
```

### Tips for Better Subagents:
- **Platform Tools:** Always include progress-reporting and user-question tools if you use a restricted allowlist. This ensures the assistant can still report progress and ask for clarification.
- **Focused Tools:** Only give an agent the tools it needs (e.g., `WebSearch` for `collector-specialist`).
- **Clear Instructions:** Use the instructions to define the agent's persona, core principles, and common tasks.
- **Reference Docs:** Point the agent to relevant project documentation (e.g., `DEVELOPER.md`, `PHASES.md`).

## 🔗 Related Resources
- [DEVELOPER.md](../DEVELOPER.md)
- [GroupScout Codex Agents](../../plugins/groupscout-agents/README.md)
