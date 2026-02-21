# Architecture Decision Records (ADRs)

Este diretório contém os registros de decisão arquitetural (ADRs) do projeto de plataforma.  
Cada arquivo documenta **o que** foi decidido, **por que**, e **quais alternativas foram consideradas**.

---

## Índice

| Documento | Título | Tipo | Status |
|---|---|---|---|
| [ADR-001](./ADR-001-three-repo-gitops-strategy.md) | Estratégia de 3 Repositórios GitOps | ADR | ✅ Aceito |
| [ADR-002](./ADR-002-single-branch-environment-per-directory.md) | Branch Única + Overlays por Ambiente | ADR | ✅ Aceito |
| [ADR-003](./ADR-003-ocm-over-rhacm.md) | OCM em vez de RHACM | ADR | ✅ Aceito |
| [ADR-004](./ADR-004-argocd-as-delivery-tool.md) | ArgoCD como ferramenta de entrega | ADR | ✅ Aceito |
| [Guia de Categorização](./governance-categorization-guide.md) | Como classificar e criar políticas OCM | Guia | 📖 Referência |

---

## Mapa de Dependências entre Decisões

```
ADR-001 (3 repos)
    └── ADR-002 (branch única)    ← como versionar o repo de políticas
    └── ADR-004 (ArgoCD)          ← qual ferramenta faz a entrega
            └── ADR-003 (OCM)     ← qual engine distribui para os clusters
```

---

## Como Criar um Novo ADR

1. Copie o cabeçalho padrão abaixo
2. Nomeie o arquivo como `ADR-00X-titulo-curto.md`
3. Abra um PR com a proposta — a discussão acontece no PR antes do merge
4. Atualize este índice

### Template de Cabeçalho

```markdown
# ADR-00X: Título

- **Status:** Proposto | Aceito | Depreciado | Supersedido por ADR-00Y
- **Data:** YYYY-MM-DD
- **Decisores:** Times envolvidos

---

## Contexto
## Decisão
## Consequências
## Alternativas Consideradas
## Referências
```

---

## Formato ADR

Utilizamos o formato [Nygard](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions):
- **Contexto** — situação que motivou a decisão
- **Decisão** — o que foi escolhido
- **Consequências** — trade-offs aceitos (positivos e negativos)
- **Alternativas Consideradas** — o que foi descartado e por quê
- **Referências** — links para discussões da comunidade e RFCs
