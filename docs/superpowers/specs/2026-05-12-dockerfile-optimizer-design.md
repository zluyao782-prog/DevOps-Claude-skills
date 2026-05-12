# Dockerfile Optimizer 技能设计

## 目标

创建一个 Claude Code 技能，能够分析已有 Dockerfile 的优化空间，或根据服务描述从零生成符合最佳实践的 Dockerfile。覆盖安全、构建效率、镜像体积、可维护性四个维度，支持 Go / Node.js / Python / Java / Rust 五种语言。

## 触发条件

关键词：Dockerfile、docker build、镜像优化、镜像太大、安全加固、多阶段构建、distroless、容器化、containerize

## 技能结构

### SKILL.md（主文件，目标 400-500 行）

1. **决策流程**（小流程图）：判断"审查优化"还是"从零生成"
2. **语言模板**：每种语言的最佳 Dockerfile 模板（多阶段构建 + 非 root + 最小基础镜像）
3. **安全规则**：非 root 用户、固定 SHA、Trivy 扫描、secret 排除、最小基础镜像选择
4. **构建效率**：层排序、BuildKit 缓存挂载、.dockerignore、多阶段并行
5. **体积优化**：基础镜像选型指南（alpine/distroless/scratch）、依赖精简
6. **可维护性**：HEALTHCHECK、OCI 标签、ARG 参数化、entrypoint vs CMD
7. **常见错误**：缓存未命中、权限问题、镜像膨胀、构建超时
8. **输出格式**：优化后 Dockerfile + 逐条修改说明 + 对比表

### references/dockerfile-checklist.md（快速检查清单）

包含按优先级排序的检查项列表，方便快速扫描。

## 设计原则

- 安全优先：所有生成/优化的 Dockerfile 默认非 root、使用固定版本
- 与 cicd-pipeline 风格一致：模板内嵌注释说明非显而易见的决策
- 不止生成语法正确的 Dockerfile，而是生成符合生产标准的 Dockerfile
