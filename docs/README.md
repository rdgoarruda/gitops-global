# Architecture Decision Records (ADRs)

Este diretório contém os registros de decisão arquitetural (ADRs) do projeto de plataforma.  
Cada arquivo documenta **o que** foi decidido, **por que**, e **quais alternativas foram consideradas**.

## Índice

| # | Decisão | Status |
|---|---|---|
| [ADR-001](./ADR-001-three-repo-gitops-strategy.md) | Estratégia de 3 Repositórios GitOps | ✅ Aceito |
| [ADR-002](./ADR-002-single-branch-environment-per-directory.md) | Branch Única + Overlays por Ambiente | ✅ Aceito |
| [ADR-003](./ADR-003-ocm-over-rhacm.md) | OCM em vez de RHACM | ✅ Aceito |
| [ADR-004](./ADR-004-argocd-as-delivery-tool.md) | ArgoCD como ferramenta de entrega | ✅ Aceito |

## Formato ADR

Utilizamos o formato [Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions):
- **Contexto** — situação que motivou a decisão
- **Decisão** — o que foi escolhido
- **Consequências** — trade-offs aceitos
- **Alternativas Consideradas** — o que foi descartado e por quê
- **Referências** — links para discussões da comunidade e RFCs
