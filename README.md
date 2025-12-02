# AI-Driven Automation

An automated coding and review system powered by AI agents (currently only support Google Gemini).
This project uses a dual-agent architecture (Coder + Reviewer) to implement code based on task instructions and iteratively improve them through automated PR reviews.

## Architecture

```
┌─────────────────┐────▶┌─────────────────┐
│  Coder Agent    │     │  Reviewer Agent │
└─────────────────┘◀────└─────────────────┘
        │                       │
        ▼                       ▼
┌─────────────────────────────────────────┐
│           GitHub Actions                │
│  • Creates PRs with code changes        │
│  • Submits reviews                      │
│  • Handles '/ask' Q&A commands          │
└─────────────────────────────────────────┘
```

### Agent Roles

**Coder Agent**: Reads task instructions from `docs/tasks`, generates code implementations, and pushes changes to branches. Also handles Q&A queries via `/ask` command.

**Reviewer Agent**: Reviews generated code against task requirements and submits GitHub reviews (Approve or Request Changes).

## Prerequisites

- GitHub repository with Actions enabled
- Two GitHub Apps (see setup below)
- AI agent API key

## Setup

### 1. Create GitHub Apps

This project requires **two separate GitHub Apps** to enable the feedback loop between Coder and Reviewer agents.
Using separate apps prevents workflows from being filtered when one bot pushes changes that trigger another.

#### App 1: Coder

1. Go to **Settings → Developer settings → GitHub Apps → New GitHub App**
2. Configure:
   - **Name**: `Gemini-Coder` (or your preferred name)
   - **Homepage URL**: Your repository URL
   - **Webhook**: Uncheck "Active" (not needed)
3. Set **Repository Permissions**:
   - Contents: Read & Write
   - Pull requests: Read & Write
   - Issues: Read & Write
   - Actions: Read & Write
   - Metadata: Read-only
4. Click **Create GitHub App**
5. Note the **App ID**
6. Generate and download a **Private Key**
7. Install the app on your repository

#### App 2: Reviewer

Repeat the same steps above with:
- **Name**: `Gemini-Reviewer` (or your preferred name)
- Same permissions as Coder app

### 2. Configure Repository Secrets

Add the following secrets in **Settings → Secrets and variables → Actions**:

| Secret | Description |
|--------|-------------|
| `GEMINI_API_KEY` | Your Google Gemini API key |
| `APP_CODER_ID` | App ID for Coder |
| `APP_CODER_SECRET` | Private key (PEM) for Coder |
| `APP_REVIEWER_ID` | App ID for Reviewer |
| `APP_REVIEWER_SECRET` | Private key (PEM) for Reviewer |

### 3. Create Task Instructions

Create task files in `docs/tasks/`:

```
docs/tasks/
├── 00_overview.md      # Global project rules (applied to all tasks)
├── 01_task_name.md     # Task 01 instructions
├── 02_another_task.md  # Task 02 instructions
└── ...
```

## Usage

### Manual Trigger (Workflow Dispatch)

1. Go to **Actions → AI Agent**
2. Click **Run workflow**
3. Configure:
   - **Task ID**: The task number (e.g., `01`, `02`)
   - **Mode**: `coder` (default) or `reviewer`
   - **PR Number**: Required if mode is `reviewer`

### Automatic Triggers

The workflow automatically runs on:

| Event | Agent | Description |
|-------|-------|-------------|
| `workflow_dispatch` | Coder/Reviewer | Manual trigger |
| `pull_request` (opened/synchronize) | Reviewer | Reviews new PRs |
| `pull_request_review` (changes_requested) | Coder | Implements requested changes |
| Comment with `/ask` | Coder | Q&A mode |

### Q&A Mode (`/ask` Command)

Use `/ask` in PR comments or review comments to ask questions or request explanations:

```
/ask What does this function do?
/ask Can you refactor this to use a map instead?
```

The bot will respond in a threaded reply.

## Project Structure

```
.github/
├── scripts/
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── config.py      # Environment configuration
│   │   ├── context.py     # Codebase context loader
│   │   ├── llm.py         # AI API client with fallback
│   │   ├── roles.py       # Coder & Reviewer prompts
│   │   └── utils.py       # File parsing utilities
│   └── main.py            # Entry point
└── workflows/
    └── agent-gemini.yml   # GitHub Actions workflow

docs/tasks/
├── 00_overview.md         # Global rules
└── XX_task_name.md        # Task instructions
```

## Configuration

Environment variables (set automatically by workflow):

| Variable | Description | Default |
|----------|-------------|---------|
| `GEMINI_API_KEY` | API key for Gemini | Required |
| `TASK_ID` | Task number to execute | `01` |
| `MODE` | Agent mode (`coder`/`reviewer`) | `coder` |
| `PR_QUESTION` | Q&A query (auto-populated) | - |
| `FEEDBACK` | Review feedback for iteration | - |
| `MAX_RETRIES` | API retry attempts | `5` |

## How It Works

### Coder-Reviewer Loop

1. **Coder** reads task instructions and generates code
2. **Coder** pushes changes and creates/updates PR
3. **Reviewer** is triggered automatically
4. **Reviewer** analyzes code against requirements
5. If **PASS**: Reviewer approves PR
6. If **FAIL**: Reviewer requests changes with feedback
7. **Coder** receives feedback and iterates (back to step 1)

### Model Fallback

The system uses automatic model fallback for reliability:
- Primary: `gemini-3-pro-preview`
- Fallback: `gemini-2.5-pro`

## Writing Task Instructions

Task files should include:

```markdown
# Task XX: Title

## Objective
Clear description of what needs to be implemented.

## Requirements
- Specific requirement 1
- Specific requirement 2

## Files to Create/Modify
- `path/to/file.go` - Description

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
```

The `00_overview.md` file contains global rules applied to all tasks (coding standards, patterns to follow, etc.).
