# K8s Manifest 技能设计

## 目标

根据服务描述生成生产级 Kubernetes 清单。覆盖 Deployment、HPA、Service、Ingress、NetworkPolicy、PodDisruptionBudget、ConfigMap、Secret 等资源类型。编码安全、可靠性、资源管理三个维度的最佳实践。

## 触发条件

k8s、kubernetes、deploy、部署到 k8s、写 k8s 配置、HPA、Ingress、Service、Deployment

## 技能结构

1. 决策流程：判断需生成哪些资源
2. Deployment 模板（含 probe、resources、securityContext）
3. HPA 模板（CPU/Memory 阈值、行为策略）
4. Service + Ingress 模板（TLS、路径路由）
5. NetworkPolicy 模板（零信任默认拒绝 + 白名单）
6. PodDisruptionBudget 模板
7. 安全规范（securityContext、非 root、只读文件系统）
8. 常见错误排查
