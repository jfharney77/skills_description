# skills_description

This repository stores reusable skill definitions for Claude Code agent workflows. Each skill lives under `saved_skills/<name>/SKILL.md` and can be invoked via a slash command.

## Skills

| Skill | Trigger | Description |
|---|---|---|
| [cerebras-setup](saved_skills/cerebras-setup/SKILL.md) | `/cerebras-setup` | Add Cerebras AI API support (`gpt-oss-120b`) alongside existing Ollama/OpenAI config in any LangChain/LangGraph Python project |
| [composio-cli](saved_skills/composio-cli/SKILL.md) | auto | Operate the Composio CLI — discover tool slugs, connect accounts, execute tools, subscribe to triggers, script workflows |
| [create-run-scripts](saved_skills/create-run-scripts/SKILL.md) | `/create-run-scripts` | Create executable `start_backend.sh`, `start_frontend.sh`, and `start_all.sh` for a FastAPI + React/Vite monorepo |
| [create-stop-script](saved_skills/create-stop-script/SKILL.md) | `/create-stop-script` | Create `scripts/stop_all.sh` that kills the running FastAPI backend and Vite dev server by PID or port |
| [explain-code](saved_skills/explain-code/SKILL.md) | auto | Explain code using analogies, ASCII diagrams, step-by-step walkthroughs, and common gotchas |
| [github-railway](saved_skills/github-railway/SKILL.md) | `/github-railway` | Set up GitHub Actions CI/CD to deploy a FastAPI + React/Vite monorepo to Railway |
| [gitlab-aws-ecs](saved_skills/gitlab-aws-ecs/SKILL.md) | `/gitlab-aws-ecs` | Set up GitLab CI/CD to deploy a FastAPI + React/Vite monorepo to AWS ECS Express Mode via ECR |
| [gitlab-railway](saved_skills/gitlab-railway/SKILL.md) | `/gitlab-railway` | Set up GitLab CI/CD to deploy a FastAPI + React/Vite monorepo to Railway |
| [graphify](saved_skills/graphify/SKILL.md) | `/graphify` | Convert any folder of files into an interactive knowledge graph with community detection, HTML viz, GraphRAG JSON, and audit report |
| [langgraph-cv](saved_skills/langgraph-cv/SKILL.md) | `/langgraph-cv` | Instrument a LangGraph agent to export `state.json` for the Context Visualizer UI |
| [ollama-wsl](saved_skills/ollama-wsl/SKILL.md) | `/ollama-wsl` | Configure a WSL2 app to reach an Ollama instance running on the Windows host |
| [security-https-bruteforce](saved_skills/security-https-bruteforce/SKILL.md) | `/security-https-bruteforce` | Audit and fix HTTPS (self-signed TLS) and brute-force protection (rate limiting + CAPTCHA) for FastAPI + vanilla JS apps |
| [security-web-vulns](saved_skills/security-web-vulns/SKILL.md) | `/security-web-vulns` | Audit and fix 5 web vulnerabilities in FastAPI + vanilla JS apps: session token leakage, stored XSS, `javascript:` URL injection, mass-assignment in PUT, secrets in DOM |
| [simple-auth](saved_skills/simple-auth/SKILL.md) | `/simple-auth` | Add JWT-based username/password auth with an admin role to a FastAPI + React (Vite) monorepo |

## Project Structure

```text
saved_skills/
  cerebras-setup/SKILL.md
  composio-cli/SKILL.md
  create-run-scripts/SKILL.md
  create-stop-script/SKILL.md
  explain-code/SKILL.md
  github-railway/SKILL.md
  gitlab-aws-ecs/SKILL.md
  gitlab-railway/SKILL.md
  graphify/SKILL.md
  langgraph-cv/SKILL.md
  ollama-wsl/SKILL.md
  security-https-bruteforce/SKILL.md
  security-web-vulns/SKILL.md
  simple-auth/SKILL.md
```

## How To Use This Repository

1. Each skill is self-contained: frontmatter (`name`, `description`, `trigger`) at the top, followed by action steps.
2. Skills with a `/trigger` are invoked explicitly; skills without one are applied automatically by Claude based on context.
3. Hard-won lessons are captured in each skill so future runs avoid known pitfalls.
