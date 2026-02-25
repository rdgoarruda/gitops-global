# gitops-global

Repositório central de **governança, políticas e infraestrutura lógica** da plataforma multi-cluster.

Faz parte de uma estratégia de [3 repositórios GitOps](docs/ADR-001-three-repo-gitops-strategy.md), sendo o responsável pela camada de plataforma e segurança — tudo que é mandatório independente de qual aplicação roda no cluster.

---

## Ambiente de Referência

O ambiente é composto por **6 clusters Kubernetes** (Kind em laboratório, equivalentes a OpenShift/EKS em produção), organizados em 2 ambientes isolados:

| Cluster | Papel | Contexto kubectl | Ambiente |
|---|---|---|---|
| `gerencia-ho` | Hub HO — ArgoCD + OCM Hub + Headlamp | `kind-gerencia-ho` | Homologação |
| `bu-a-ho` | Worker HO — BU-A | `kind-bu-a-ho` | Homologação |
| `bu-b-ho` | Worker HO — BU-B | `kind-bu-b-ho` | Homologação |
| `gerencia-pr` | Hub PR — ArgoCD + OCM Hub + Headlamp | `kind-gerencia-pr` | Produção |
| `bu-a-pr` | Worker PR — BU-A | `kind-bu-a-pr` | Produção |
| `bu-b-pr` | Worker PR — BU-B | `kind-bu-b-pr` | Produção |

> **Dual-Hub:** Cada ambiente (HO/PR) tem seu próprio cluster de gerência com ArgoCD e OCM Hub independentes. Isso garante isolamento total entre homologação e produção.

> **Nota OCM/RHACM:** O ambiente usa [OCM](https://open-cluster-management.io/) (open-source) como engine de governança. As APIs (`Policy`, `Placement`, `PlacementBinding`) são 100% compatíveis com Red Hat ACM — o código funciona em produção sem alteração. Veja [ADR-003](docs/ADR-003-ocm-over-rhacm.md).

---

## Estrutura do Repositório

```text
gitops-global/
├── .github/
│   └── CODEOWNERS                  # Controle de quem pode aprovar mudanças por diretório
│
├── bootstrap/                      # 🚀 Ponto de entrada do ArgoCD — App-of-Apps
│   ├── ho/
│   │   ├── root-app.yaml           # Application que aponta para bootstrap/ho (instalada manualmente 1x)
│   │   ├── app-ocm-config.yaml     # Application que aponta para config/ho (domínio OCM)
│   │   ├── appset-governance.yaml  # ApplicationSet com Git Generator — autodetecta governance/*
│   │   └── kustomization.yaml      # Inclui app-ocm-config + appset
│   └── pr/
│       ├── root-app.yaml
│       ├── app-ocm-config.yaml
│       ├── appset-governance.yaml
│       └── kustomization.yaml
│
├── config/                         # ⚙️ Infraestrutura lógica do Hub OCM
│   ├── ho/
│   │   ├── namespace.yaml          # Namespace "ocm-policies-ho" no Hub
│   │   ├── placement.yaml          # Seleciona clusters com label env=ho
│   │   ├── clusterset-binding.yaml # Vincula o Placement ao ClusterSet "global"
│   │   ├── binding-*.yaml (×6)     # Liga cada PolicySet ao Placement ho
│   │   └── kustomization.yaml
│   └── pr/                         # Mesma estrutura para produção
│
├── domains/                        # 📁 Bootstraps e ApplicationSets por BU
│   ├── bu-a/
│   │   ├── bootstrap/ho/           # root-bu-a-ho.yaml → domains/bu-a/ho
│   │   ├── bootstrap/pr/           # root-bu-a-pr.yaml → domains/bu-a/pr
│   │   ├── ho/appset-tools-ho.yaml # Deploy de tools do gitops-bu-a nos clusters HO
│   │   └── pr/appset-tools-pr.yaml # Deploy de tools do gitops-bu-a nos clusters PR
│   └── bu-b/                       # Mesma estrutura para BU-B
│
├── governance/                     # ⚖️ Manifestos de Políticas OCM (auto-descobertos pelo ArgoCD)
│   ├── capacity/                   # ResourceQuotas, LimitRanges
│   │   ├── base/                   # Manifesto base compartilhado
│   │   └── overlays/
│   │       ├── ho/                 # Overlay Homologação
│   │       └── pr/                 # Overlay Produção
│   ├── compliance/                 # Controles NIST, CIS
│   ├── infrastructure/             # StorageClasses, Operators (OLM)
│   ├── observability/              # Agentes de monitoramento e log
│   ├── platform/                   # Namespaces base, RBAC
│   └── security/                   # Pod Security Standards, NetworkPolicies
│
└── docs/                           # 📚 ADRs e guias de arquitetura
    ├── ADR-001-three-repo-gitops-strategy.md
    ├── ADR-002-single-branch-environment-per-directory.md
    ├── ADR-003-ocm-over-rhacm.md
    ├── ADR-004-argocd-as-delivery-tool.md
    └── governance-categorization-guide.md
```

---

## Como Funciona — Fluxo Completo

### 1. Bootstrap (execução única por ambiente)

A única operação manual necessária é instalar a root-app no ArgoCD de cada hub. A partir daí, tudo é auto-gerenciado:

```bash
# Ambiente HO
kubectl apply -f bootstrap/ho/root-app.yaml --context kind-gerencia-ho

# Ambiente PR
kubectl apply -f bootstrap/pr/root-app.yaml --context kind-gerencia-pr
```

A `root-app.yaml` aponta para `bootstrap/ho/` (ou `pr/`), que pelo `kustomization.yaml` inclui:
- `app-ocm-config.yaml` — toda a infraestrutura lógica OCM do ambiente
- `appset-governance.yaml` — o ApplicationSet que faz autodiscovery de políticas

### 2. Autodiscovery de Políticas (ApplicationSet + Git Generator)

O `appset-governance.yaml` usa um **Git Generator** que varre `governance/*`:

```yaml
generators:
  - git:
      repoURL: 'https://github.com/rdgoarruda/gitops-global.git'
      revision: main
      directories:
        - path: governance/*     # detecta cada subpasta automaticamente
```

Para cada pasta (ex: `governance/platform`), o ArgoCD cria uma `Application` apontando para `governance/platform/overlays/ho`. Resultado no cluster:

```
ocm-policy-platform-ho         Synced  Healthy
ocm-policy-security-ho         Synced  Healthy
ocm-policy-observability-ho    Synced  Healthy
ocm-policy-capacity-ho         Synced  Healthy
ocm-policy-compliance-ho       Synced  Healthy
ocm-policy-infrastructure-ho   Synced  Healthy
```

### 3. Distribuição OCM para Clusters Worker

```
Git (governance/platform/)
        │
        │  ArgoCD detecta pasta → cria Application
        ▼
   ArgoCD (em gerencia-ho)
        │
        │  aplica Policy + PolicySet no namespace ocm-policies-ho
        ▼
   OCM Hub (em gerencia-ho)
        │
        │  PlacementBinding liga PolicySet ao Placement
        │  Placement seleciona clusters com label env=ho
        │
        ├──▶ bu-a-ho (label: env=ho, bu=bu-a)
        │       └── Policies em Compliant ✅
        └──▶ bu-b-ho (label: env=ho, bu=bu-b)
                └── Policies em Compliant ✅
```

### 4. Verificação de Conformidade

```bash
# Estado das políticas no Hub HO
kubectl get policy,policyset -n ocm-policies-ho --context kind-gerencia-ho

# Conformidade nos workers
kubectl get policy -n open-cluster-management-policies --context kind-bu-a-ho
kubectl get policy -n open-cluster-management-policies --context kind-bu-b-ho
```

---

## Como Adicionar uma Nova Categoria de Governança

1. **Crie a pasta** em `governance/<nova-categoria>/`
2. **Adicione os manifestos base**:
   - `base/policy-<nome>.yaml` — a Policy OCM com `remediationAction: enforce|inform`
   - `base/policyset.yaml` — agrupa as políticas da categoria
   - `base/kustomization.yaml` — lista os recursos acima
3. **Crie os overlays**:
   - `overlays/ho/kustomization.yaml` — referencia `../../base`
   - `overlays/pr/kustomization.yaml` — referencia `../../base` (com patches se necessário)
4. **Vincule ao ambiente** em `config/ho/` e `config/pr/`:
   - Crie `binding-<nova-categoria>.yaml` com um `PlacementBinding`
   - Adicione o novo arquivo ao `kustomization.yaml` local
5. **Abra o PR** — o ArgoCD detectará automaticamente

> Consulte o [Guia de Categorização](docs/governance-categorization-guide.md) para decidir em qual categoria cada política se encaixa.

---

## Controle de Acesso (CODEOWNERS)

| Caminho | Aprovadores | Motivo |
|---|---|---|
| `config/pr/**` | `@rdgoarruda` | Produção — controle restrito |
| `bootstrap/pr/**` | `@rdgoarruda` | Produção — controle restrito |
| `governance/**` | `@rdgoarruda` | Afeta todos os clusters |
| `config/ho/**` | `@rdgoarruda` | Homologação — time de plataforma |
| `bootstrap/ho/**` | `@rdgoarruda` | Homologação — time de plataforma |
| `docs/**` | `@rdgoarruda` | Decisões arquiteturais |

> Em produção, substitua `@rdgoarruda` pelos grupos reais: `@org/sre-team`, `@org/platform-team`.

---

## Fluxo de Promoção HO → PR

```
Mudança de política
        │
        ├─── PR para config/ho/ + governance/  →  merge em main
        │         │
        │         └── ArgoCD (gerencia-ho) sincroniza → OCM distribui para bu-a-ho, bu-b-ho
        │                   │
        │                   └── Validação / aprovação humana
        │
        └─── PR para config/pr/                →  merge em main (exige @sre-team)
                  │
                  └── ArgoCD (gerencia-pr) sincroniza → OCM distribui para bu-a-pr, bu-b-pr
```

---

## Ferramentas nos Clusters de Gerenciamento

| Ferramenta | Namespace | Acesso HO | Acesso PR |
|---|---|---|---|
| ArgoCD | `argocd` | http://argocd-ho.local | http://argocd-pr.local:8080 |
| Headlamp | `headlamp` | http://headlamp-ho.local | http://headlamp-pr.local:8080 |
| OCM Hub | `open-cluster-management-hub` | via CLI / ArgoCD | via CLI / ArgoCD |

---

## Referências de Arquitetura (ADRs)

| # | Decisão | Status |
|---|---|---|
| [ADR-001](docs/ADR-001-three-repo-gitops-strategy.md) | Estratégia de 3 Repositórios GitOps | ✅ Aceito |
| [ADR-002](docs/ADR-002-single-branch-environment-per-directory.md) | Branch Única + Overlays por Ambiente | ✅ Aceito |
| [ADR-003](docs/ADR-003-ocm-over-rhacm.md) | OCM em vez de RHACM | ✅ Aceito |
| [ADR-004](docs/ADR-004-argocd-as-delivery-tool.md) | ArgoCD como Ferramenta de Entrega | ✅ Aceito |
| [Guia](docs/governance-categorization-guide.md) | Categorização de Políticas OCM | 📖 Referência |
