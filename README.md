# DevOps Claude Skills

> A collection of production-grade [Claude Code](https://claude.ai/code) skills for DevOps engineers.
> Stop writing boilerplate configs. Let Claude generate them the right way.

English | [中文](README_CN.md)

---

## 🚀 What is this?

This repo packages 5 expert-level Claude Code skills that turn natural language into production-ready DevOps artifacts — CI/CD pipelines, Kubernetes manifests, optimized Dockerfiles, incident runbooks, and Terraform modules.

Each skill encodes real-world best practices so Claude doesn't just generate *a* config — it generates *the right* config.

---

## 📦 Skills included

| Skill | What it does | Trigger phrases |
|-------|-------------|-----------------|
| [cicd-pipeline](skills/cicd-pipeline/) | Generate GitHub Actions / GitLab CI / Jenkins pipelines | "create a pipeline", "set up CI/CD", "写个流水线" |
| [k8s-manifest](skills/k8s-manifest/) | Generate production K8s manifests (Deployment, HPA, Ingress...) | "deploy to k8s", "generate manifests", "写 k8s 配置" |
| [dockerfile-optimizer](skills/dockerfile-optimizer/) | Analyze and optimize Dockerfiles | "optimize my Dockerfile", "镜像太大了", "Dockerfile 安全加固" |
| [incident-runbook](skills/incident-runbook/) | Auto-generate incident runbooks from service descriptions | "create a runbook", "故障手册", "on-call guide" |
| [terraform-module](skills/terraform-module/) | Generate Terraform modules with IaC best practices | "write terraform for", "provision infra", "写 IaC" |

---

## ⚡ Quick Install

```bash
# Install all skills
claude skill install https://github.com/zluyao782-prog/DevOps-Claude-skills/releases/latest/download/cicd-pipeline.skill
claude skill install https://github.com/zluyao782-prog/DevOps-Claude-skills/releases/latest/download/k8s-manifest.skill
claude skill install https://github.com/zluyao782-prog/DevOps-Claude-skills/releases/latest/download/dockerfile-optimizer.skill
claude skill install https://github.com/zluyao782-prog/DevOps-Claude-skills/releases/latest/download/incident-runbook.skill
claude skill install https://github.com/zluyao782-prog/DevOps-Claude-skills/releases/latest/download/terraform-module.skill
```

Or clone and install locally:

```bash
git clone https://github.com/zluyao782-prog/DevOps-Claude-skills
cd devops-claude-skills

# Install individual skill
claude skill install skills/cicd-pipeline/
```

---

## 🎬 Demo

### cicd-pipeline

**You say:**
> "I have a Go service that deploys to EKS. Set up a GitHub Actions pipeline with testing, Docker build, and blue-green deployment."

**Claude generates:**
- Complete `.github/workflows/deploy.yml` with caching, matrix builds, OIDC auth
- Separate jobs for lint → test → build → security-scan → deploy
- Blue-green deployment logic with automatic rollback on failure

See [examples/cicd-pipeline/](examples/cicd-pipeline/) for full input/output samples.

---

### k8s-manifest

**You say:**
> "Deploy my payment-service (3 replicas, needs 512Mi RAM, connects to Postgres, exposes /api on port 8080). Scale between 2-10 pods based on CPU."

**Claude generates:**
- `Deployment` with proper resource limits/requests, liveness/readiness probes
- `HorizontalPodAutoscaler` with sane thresholds
- `Service` + `Ingress` with TLS
- `PodDisruptionBudget` for zero-downtime deployments
- `NetworkPolicy` for pod isolation

---

## 🗂️ Project structure

```
devops-claude-skills/
├── README.md
├── skills/
│   ├── cicd-pipeline/
│   │   └── SKILL.md
│   ├── k8s-manifest/
│   │   ├── SKILL.md
│   │   └── references/
│   │       ├── k8s-best-practices.md
│   │       └── troubleshooting.md
│   ├── dockerfile-optimizer/
│   │   └── SKILL.md
│   ├── incident-runbook/
│   │   ├── SKILL.md
│   │   └── references/
│   │       └── runbook-template.md
│   └── terraform-module/
│       ├── SKILL.md
│       └── references/
│           ├── aws.md
│           ├── gcp.md
│           └── azure.md
└── examples/
    ├── cicd-pipeline/
    ├── k8s-manifest/
    └── dockerfile-optimizer/
```

---

## 🤝 Contributing

PRs welcome! If you have a DevOps skill that's made your life easier, package it up and submit.

Guidelines:
- Each skill should encode real-world best practices, not just generate syntactically valid configs
- Include at least one input/output example in `examples/`
- Skill descriptions should be specific enough to trigger correctly

---

## License

MIT
