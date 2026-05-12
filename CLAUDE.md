# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A curated collection of Claude Code skills for DevOps engineers. Each skill encodes real-world best practices so Claude generates production-ready DevOps artifacts (CI/CD pipelines, K8s manifests, Dockerfiles, runbooks, Terraform modules).

## Skill file format

Every skill is a `SKILL.md` with YAML frontmatter:

```yaml
---
name: skill-name
description: >
  When to trigger this skill. Mentions keywords, user intents, and non-triggers.
---
```

The `name` must match the skill's directory name (e.g., `skills/cicd-pipeline/SKILL.md` → `name: cicd-pipeline`). The `description` is used for skill matching — it should list trigger phrases and explicitly state when NOT to trigger.

Each skill lives in `skills/<name>/SKILL.md`. Supporting files (reference docs, templates) go in `skills/<name>/references/`.

See the [superpowers:writing-skills](https://claude.ai/code) skill for detailed authoring guidance.

## Current skills

| Skill | Location | Lines |
|-------|----------|-------|
| cicd-pipeline | `skills/cicd-pipeline/SKILL.md` | 485 |
| k8s-manifest | `skills/k8s-manifest/SKILL.md` | 520 |
| dockerfile-optimizer | `skills/dockerfile-optimizer/SKILL.md` | 658 |
| incident-runbook | `skills/incident-runbook/SKILL.md` | 420 |
| terraform-module | `skills/terraform-module/SKILL.md` | 550 |

Root `SKILL.md` is a copy of cicd-pipeline — the canonical file is under `skills/cicd-pipeline/`.

## Creating a new skill

1. Create `skills/<name>/` directory
2. Write `SKILL.md` with frontmatter (`name`, `description`) and the skill body
3. If the skill needs reference docs, add `skills/<name>/references/*.md`
4. Add at least one input/output example in `examples/<name>/`
5. When editing or creating skills, invoke the `superpowers:writing-skills` skill

## Skill design principles

- Skills encode real-world best practices, not just syntactically valid configs
- Security first: never hardcode secrets, use OIDC, pin action SHAs, scan images
- Performance: cache dependencies and Docker layers, parallelize independent jobs
- Reliability: include health checks, rollback logic, job timeouts
- Trigger descriptions should be specific enough to avoid false positives
