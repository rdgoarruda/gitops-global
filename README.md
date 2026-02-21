# gitops-global

Repositório central de **governança, políticas e infraestrutura lógica** da plataforma multi-cluster.

Faz parte de uma estratégia de [3 repositórios GitOps](docs/ADR-001-three-repo-gitops-strategy.md), sendo o responsável pela camada de plataforma e segurança — tudo que é mandatório independente de qual aplicação roda no cluster.

---

## Ambiente de Referência

O ambiente é composto por **3 clusters Kubernetes** (kind em laboratório, equivalentes a OpenShift/EKS em produção):

| Cluster | Papel | Contexto kubectl |
|---|---|---|
| `gerencia-global` | Hub de gerenciamento — roda ArgoCD e OCM Hub | `kind-gerencia-global` |
| `nprod-bu-x` | Worker de não-produção — recebe políticas com label `env: nprod` | `kind-nprod-bu-x` |
| `prod-bu-x` | Worker de produção — recebe políticas com label `env: prod` | `kind-prod-bu-x` |

> **Nota OCM/RHACM:** O ambiente usa [OCM](https://open-cluster-management.io/) (open-source) como engine de governança. As APIs (`Policy`, `Placement`, `PlacementBinding`) são 100% compatíveis com Red Hat ACM — o código funciona em produção sem alteração. Veja [ADR-003](docs/ADR-003-ocm-over-rhacm.md).

---

## Estrutura do Repositório

```text
gitops-global/
├── .github/
│   └── CODEOWNERS                  # Controle de quem pode aprovar mudanças por diretório
│
├── bootstrap/                      # 🚀 Ponto de entrada do ArgoCD — App-of-Apps
│   ├── nprod/
│   │   ├── root-app.yaml           # Application que aponta para bootstrap/nprod (instalada manualmente 1x)
│   │   ├── appset-governance.yaml  # ApplicationSet com Git Generator — autodetecta governance/*
│   │   └── kustomization.yaml      # Inclui appset + config/nprod
│   └── prod/
│       ├── root-app.yaml
│       ├── appset-governance.yaml
│       └── kustomization.yaml      # Inclui appset + config/prod
│
├── config/                         # ⚙️ Infraestrutura lógica do Hub OCM
│   ├── nprod/
│   │   ├── namespace.yaml          # Namespace "ocm-policies-nprod" no Hub
│   │   ├── placement.yaml          # Seleciona clusters com label env=nprod
│   │   ├── clusterset-binding.yaml # Vincula o Placement ao ClusterSet padrão
│   │   ├── binding-platform.yaml   # Liga PolicySet "platform" ao Placement nprod
│   │   ├── binding-observability.yaml
│   │   ├── binding-capacity.yaml
│   │   ├── binding-security.yaml
│   │   ├── binding-compliance.yaml
│   │   ├── binding-infrastructure.yaml
│   │   └── kustomization.yaml
│   └── prod/                       # Mesma estrutura para produção
│
├── governance/                     # ⚖️ Manifestos de Políticas OCM (auto-descobertos pelo ArgoCD)
│   ├── platform/                   # Namespaces base, RBAC
│   ├── security/                   # Pod Security Standards, NetworkPolicies
│   ├── observability/              # Agentes de monitoramento e log
│   ├── capacity/                   # ResourceQuotas, LimitRanges
│   ├── compliance/                 # Controles NIST, CIS
│   └── infrastructure/             # StorageClasses, Operators (OLM)
│
└── docs/                           # 📚 ADRs e guias de arquitetura
    ├── ADR-001-three-repo-gitops-strategy.md
    ├── ADR-002-single-branch-environment-per-directory.md
    ├── ADR-003-ocm-over-rhacm.md
    ├── ADR-004-argocd-as-delivery-tool.md
    ├── governance-categorization-guide.md
    └── README.md
```

---

## Como Funciona — Fluxo Completo

### 1. Bootstrap (execução única)

A única operação manual necessária é instalar a root-app no ArgoCD do cluster de gerenciamento. A partir daí, tudo é auto-gerenciado:

```bash
# Aplicar a root-app de nprod no ArgoCD do cluster hub
kubectl apply -f bootstrap/nprod/root-app.yaml --context kind-gerencia-global
```

A `root-app.yaml` aponta para `bootstrap/nprod/`, que pelo `kustomization.yaml` inclui:
- `appset-governance.yaml` — o ApplicationSet que faz autodiscovery
- `../../config/nprod` — toda a infraestrutura lógica OCM do ambiente

### 2. Autodiscovery de Políticas (ApplicationSet + Git Generator)

O `appset-governance.yaml` usa um **Git Generator** que varre `governance/*` no repositório:

```yaml
generators:
  - git:
      repoURL: 'https://github.com/rdgoarruda/gitops-global.git'
      revision: main
      directories:
        - path: governance/*     # detecta cada subpasta automaticamente
```

Para cada pasta encontrada (ex: `governance/platform`), o ArgoCD cria automaticamente uma `Application` com o nome `ocm-policy-<categoria>-nprod`. Resultado atual no cluster:

```
ocm-policy-platform-nprod         Synced  Healthy
ocm-policy-security-nprod         Synced  Healthy
ocm-policy-observability-nprod    Synced  Healthy
ocm-policy-capacity-nprod         Synced  Healthy
ocm-policy-compliance-nprod       Synced  Healthy
ocm-policy-infrastructure-nprod   Synced  Healthy
```

### 3. Distribuição OCM para Clusters Worker

Após o ArgoCD aplicar os manifestos no Hub, o OCM assume o controle de distribuição:

```
Git (governance/platform/)
        │
        │  ArgoCD detecta nova pasta → cria Application
        ▼
   ArgoCD (em gerencia-global)
        │
        │  aplica Policy + PolicySet no namespace ocm-policies-nprod
        ▼
   OCM Hub (em gerencia-global)
        │
        │  PlacementBinding liga PolicySet ao Placement
        │  Placement seleciona clusters com label env=nprod
        │
        └──▶ nprod-bu-x (label: env=nprod)
                 ├── Namespace "platform-ops" criado ✅
                 ├── Namespace "observability-ops" criado ✅
                 └── ... (políticas em Compliant)
```

### 4. Verificação de Conformidade

```bash
# Estado de todas as políticas no Hub
kubectl get policy,policyset -n ocm-policies-nprod --context kind-gerencia-global

# Conformidade vista do cluster worker
kubectl get policy -n open-cluster-management-policies --context kind-nprod-bu-x
```

---

## Como Adicionar uma Nova Categoria de Governança

1. **Crie a pasta** em `governance/<nova-categoria>/`
2. **Adicione os manifestos**:
   - `policy-<nome>.yaml` — a Policy OCM com `remediationAction: enforce|inform`
   - `policyset.yaml` — agrupa as políticas da categoria
   - `kustomization.yaml` — lista os recursos acima
3. **Vincule ao ambiente** em `config/nprod/` (e/ou `config/prod/`):
   - Crie `binding-<nova-categoria>.yaml` com um `PlacementBinding`
   - Adicione o novo arquivo ao `kustomization.yaml` local
4. **Abra o PR** — o ArgoCD detectará a nova pasta automaticamente e o OCM distribuirá para os clusters corretos

> Consulte o [Guia de Categorização](docs/governance-categorization-guide.md) para decidir em qual categoria cada política se encaixa.

---

## Controle de Acesso (CODEOWNERS)

Mudanças neste repositório exigem aprovação conforme o `CODEOWNERS`:

| Caminho | Aprovadores | Motivo |
|---|---|---|
| `docs/**` | `@rdgoarruda` | Decisões arquiteturais — tech lead |
| `ocm-policies/overlays/prod/**` | `@rdgoarruda` | Produção — controle restrito |
| `ocm-policies/overlays/nprod/**` | `@rdgoarruda` | Não-produção — time de plataforma |
| `ocm-policies/base/**` | `@rdgoarruda` | Afeta todos os ambientes |

> Em produção, substitua `@rdgoarruda` pelos grupos reais: `@org/sre-team`, `@org/platform-team`.

---

## Fluxo de Promoção nprod → prod

```
Mudança de política
        │
        ├─── PR para config/nprod/ + governance/  →  merge em main
        │         │
        │         └── ArgoCD sincroniza → OCM distribui para nprod-bu-x
        │                   │
        │                   └── Validação / aprovação humana
        │
        └─── PR para config/prod/                →  merge em main (exige @sre-team)
                  │
                  └── ArgoCD sincroniza → OCM distribui para prod-bu-x
```

---

## Ferramentas no Cluster de Gerenciamento

| Ferramenta | Namespace | Acesso |
|---|---|---|
| ArgoCD | `argocd` | `http://argocd.local` |
| Headlamp (UI Kubernetes) | `headlamp` | `http://headlamp.local` |
| OCM Hub | `open-cluster-management-hub` | via CLI / ArgoCD |

---

## Referências de Arquitetura (ADRs)

| # | Decisão | Status |
|---|---|---|
| [ADR-001](docs/ADR-001-three-repo-gitops-strategy.md) | Estratégia de 3 Repositórios GitOps | ✅ Aceito |
| [ADR-002](docs/ADR-002-single-branch-environment-per-directory.md) | Branch Única + Overlays por Ambiente | ✅ Aceito |
| [ADR-003](docs/ADR-003-ocm-over-rhacm.md) | OCM em vez de RHACM | ✅ Aceito |
| [ADR-004](docs/ADR-004-argocd-as-delivery-tool.md) | ArgoCD como Ferramenta de Entrega | ✅ Aceito |
| [Guia](docs/governance-categorization-guide.md) | Categorização de Políticas OCM | 📖 Referência |
