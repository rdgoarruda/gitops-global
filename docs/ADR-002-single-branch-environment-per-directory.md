# ADR-002: Branch Única (main) + Overlays por Ambiente

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

Com múltiplos ambientes (`nprod`, `prod`) sendo gerenciados a partir do mesmo repositório
de políticas, precisávamos definir como controlar o que vai para cada ambiente de forma
segura, auditável e sem introduzir complexidade operacional desnecessária.

A pressão inicial era usar branches separadas por ambiente (padrão pre-GitOps de equipes
que migraram de um workflow de feature branches para múltiplos ambientes).

---

## Decisão

**Adotamos uma única branch protegida (`main`) com separação de ambientes por diretório
(Kustomize overlays) e controle de acesso via `CODEOWNERS`.**

```
main (branch protegida)
├── base/                    # Recursos compartilhados
├── overlays/
│   ├── nprod/               # Configurações de não-produção
│   └── prod/                # Configurações de produção (CODEOWNERS restrito)
└── .github/
    └── CODEOWNERS
```

### Configuração de CODEOWNERS
```
# Produção exige aprovação explícita do time SRE
overlays/prod/**        @org/sre-team

# Base afeta todos os ambientes — aprovação ampla
base/**                 @org/platform-team

# Não-produção — qualquer membro do time de plataforma
overlays/nprod/**       @org/platform-team
```

### GitHub Branch Protection Rules (`main`)
- ✅ Require pull request before merging
- ✅ Require approvals: mínimo 1 (nprod) / 2 (prod via CODEOWNERS)
- ✅ Require review from CODEOWNERS
- ✅ Dismiss stale reviews when new commits are pushed
- ✅ Require status checks (lint, dry-run `kubectl apply --dry-run`)

---

## Consequências

**Positivas:**
- **História linear:** Todo o estado da plataforma está visível numa única branch, sem divergências ou cherrypicks
- **Promoção explícita:** Ir de nprod para prod exige um segundo PR deliberado — não acontece automaticamente
- **Auditoria por path:** É possível ver exatamente quem aprovou cada mudança em prod via histórico de PR
- **Sem merge hell:** Não há necessidade de manter branches sincronizadas entre si
- **Alinhado com GitOps puro:** O estado em `main` é a **fonte de verdade** — o que está no `main` é o que está rodando

**Negativas / Trade-offs aceitos:**
- Requer disciplina de PR: mudanças em `overlays/prod/` devem ser abertas como PRs separados dos de `overlays/nprod/`
- CODEOWNERS não substitui testes automatizados — recomenda-se adicionar `kubectl apply --dry-run=server` no CI

---

## Alternativas Consideradas

### ❌ Multi-branch por ambiente (main, nprod, prod)
```
main → nprod → prod  (promoção via merge/cherry-pick)
```

**Por que descartado:**
- **Complexidade operacional:** Cada mudança precisa ser aplicada em múltiplas branches via cherry-pick ou merge
- **Divergência inevitável:** Ao longo do tempo, branches divergem e "promotions" viram grandes PRs difíceis de revisar
- **Histórico confuso:** O log mostra merges entre branches, não mudanças reais de configuração
- **Rejeição da comunidade:** A comunidade GitOps (Flux, ArgoCD) ativamente desencoraja esse padrão desde 2021

### ❌ Tags por ambiente (v1.0.0-nprod, v1.0.0-prod)
**Por que descartado:**
- Útil para bibliotecas/charts, mas inviável para configurações que mudam frequentemente
- Requer pipeline adicional de tag management

### ❌ Sem CODEOWNERS (PR review genérico)
**Por que descartado:**
- Sem CODEOWNERS, qualquer membro do time pode aprovar uma mudança em produção
- Não há rastreabilidade de quem é o "dono" responsável por cada path

---

## Padrão na Indústria

Este padrão é chamado de **"Environment-per-Directory"** ou **"Trunk-Based GitOps"** e é
amplamente documentado e adotado:

- **Netflix, Spotify, Shopify** utilizam este modelo com ferramentas similares
- É a abordagem **padrão recomendada pelo ArgoCD** em sua documentação oficial de best practices
- O **Flux project** chama de _"Monorepo layout"_ e o documenta como caminho primário
- O **OpenGitOps CNCF Working Group** define como princípio: _"Version controlled, declarative description of the desired system state"_ — implicitamente single-source-of-truth

---

## Referências

- [ArgoCD Best Practices — Repository Structure](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/#separating-config-vs-source-code-repositories)
- [Flux Monorepo Guide](https://fluxcd.io/flux/guides/repository-structure/#monorepo)
- [OpenGitOps Principles — CNCF](https://opengitops.dev/)
- [GitHub CODEOWNERS Documentation](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [Codefresh GitOps Guide — Environment Branching Anti-patterns](https://codefresh.io/blog/how-to-model-your-gitops-environments-and-promote-releases-between-them/)
- [Weaveworks — Stop Using Branches for Deploying to Different GitOps Environments](https://www.weave.works/blog/stop-using-branches-deploying-different-gitops-environments)
