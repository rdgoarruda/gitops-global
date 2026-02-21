# ADR-004: ArgoCD como Ferramenta de Entrega Contínua GitOps

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

A equipe precisava de uma ferramenta GitOps para sincronizar o estado dos repositórios
com os clusters Kubernetes. Os requisitos eram:

1. **Pull-based delivery** — o cluster puxa as mudanças, não o pipeline empurra
2. **Multi-cluster** — capaz de gerenciar deploys em `nprod-bu-x` e `prod-bu-x` a partir de um ponto central
3. **UI visual** — observabilidade do estado de sincronização sem uso de CLI
4. **Kustomize/Helm nativo** — suporte aos padrões de templating já usados
5. **Controle de divergência (drift detection)** — alertar se o estado do cluster divergir do repositório

---

## Decisão

**Adotamos o ArgoCD como ferramenta de entrega contínua GitOps.**

ArgoCD roda no cluster de gerenciamento (`gerencia-global`) e é responsável por:
- Sincronizar os manifestos dos repositórios para os clusters worker
- Servir como UI de observabilidade do estado da plataforma
- Ser a "cola" entre o repositório Git e o OCM Hub

---

## Consequências

**Positivas:**
- **Pull-based nativo:** O agente ArgoCD no Hub puxa as mudanças — sem credenciais de cluster expostas em pipelines externos
- **ApplicationSet:** Permite criar N aplicações a partir de um template + lista de clusters — escalável para 10, 100 clusters
- **Drift detection automático:** ArgoCD detecta e alerta (ou corrige automaticamente) qualquer mudança manual no cluster
- **Integração com OCM:** ArgoCD pode usar os clusters registrados no OCM como destinos de deploy
- **Projeto CNCF Graduated:** Status de maior maturidade no ecossistemas CNCF — usado em produção por Intuit, Red Hat, Tesla, Alibaba

**Negativas / Trade-offs aceitos:**
- ArgoCD adiciona ~200MB de RAM no cluster Hub — aceitável dado que nosso `gerencia-global` é dedicado à gestão
- Requer manutenção de atualização do próprio ArgoCD
- A UI é somente-leitura para operadores — deploys sempre passam pelo Git

---

## Modelo de Uso no Projeto

```
Git Repository (02-platform-policies)
        │
        │  (polling / webhook)
        ▼
   ArgoCD (em gerencia-global)
        │
        │  aplica manifests (OCM Policies, PlacementRules)
        ▼
   OCM Hub (em gerencia-global)
        │
        │  distribui para clusters selecionados por label
        ├──▶ nprod-bu-x  (label: env=nprod)
        └──▶ prod-bu-x   (label: env=prod)
```

---

## Alternativas Consideradas

### ❌ Flux CD
Projeto igualmente maduro (CNCF Graduated), também pull-based.

**Por que não escolhido:**
- Sem UI nativa (requer Weave GitOps separado)
- Menor adoção corporativa no Brasil no momento
- ArgoCD já estava instalado no ambiente e funcionando

> Nota: Flux seria uma escolha igualmente válida. A decisão é contextual, não técnica.

### ❌ Pipeline CI/CD push-based (GitHub Actions / Jenkins)
Pipeline que executa `kubectl apply` diretamente.

**Por que descartado:**
- **Credenciais expostas:** O pipeline precisa de kubeconfig com acesso ao cluster — aumenta superfície de ataque
- **Sem drift detection:** Se alguém faz `kubectl apply` manual, o pipeline não sabe
- **Não é GitOps:** Push-based entrega não garante que o estado do Git = estado do cluster
- Requer manutenção de pipelines por cluster

### ❌ Helm standalone (sem ArgoCD)
Usar apenas `helm upgrade` em pipelines.

**Por que descartado:**
- Sem reconcile loop: o estado pode divergir sem detecção
- Sem multi-cluster nativo
- Não integra com OCM

---

## Referências

- [ArgoCD — CNCF Graduated Project](https://www.cncf.io/projects/argo/)
- [ArgoCD Multi-Cluster Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/#clusters)
- [ArgoCD ApplicationSet Controller](https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/)
- [GitOps Principles — CNCF OpenGitOps](https://opengitops.dev/)
- [ArgoCD vs Flux — Community Comparison](https://blog.container-solutions.com/argo-cd-versus-flux-cd-right-gitops-tool)
- [Intuit's Journey with ArgoCD at Scale](https://www.youtube.com/watch?v=ahgE4BbPs40)
