# skills_description

This repository stores reusable skill definitions for agent workflows.

## Current Status

The project currently contains one saved skill:

- `gitlab-aws-ecs`

## Project Structure

```text
saved_skills/
  gitlab-aws-ecs/
    SKILL.md
```

## Included Skill

### gitlab-aws-ecs

Purpose:

- Guides setup and repair of a GitLab CI/CD pipeline for deploying a monorepo to AWS ECS Express Mode via ECR.
- Targets a common stack: FastAPI backend + React/Vite frontend.

What the skill documents:

- Practical deployment lessons and edge cases discovered from real usage.
- Required AWS IAM roles, permissions, and ECR prerequisites.
- Recommended Dockerfile patterns for backend and frontend services.
- Example `.gitlab-ci.yml` build/deploy job patterns.
- First-deploy checklist and ordering for `VITE_API_URL` dependencies.

## How To Use This Repository

1. Add each skill under `saved_skills/<skill-name>/SKILL.md`.
2. Keep each skill self-contained with frontmatter (`name`, `description`, `trigger`) and clear action steps.
3. Capture non-obvious pitfalls in a "hard-won lessons" section so future runs avoid repeated failures.

## Notes

- This repository currently appears to be in an initial bootstrap state with no prior commits.
- The default branch is `main`, and a remote named `origin` is configured.