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

> **Dual-Hub:** Cada ambiente (HO/PR) tem seu próprio cluster de gerência com ArgoCD e OCM Hub independentes. Garante isolamento total entre homologação e produção.

> **Nota OCM/RHACM:** O ambiente usa [OCM](https://open-cluster-management.io/) (open-source) como engine de governança. As APIs (`Policy`, `Placement`, `PlacementBinding`) são 100% compatíveis com Red Hat ACM — o código funciona em produção sem alteração. Veja [ADR-003](docs/ADR-003-ocm-over-rhacm.md).

---

## Estrutura do Repositório

```text
gitops-global/
├── bootstrap/                       # 🚀 Ponto de entrada do ArgoCD — App-of-Apps
│   ├── ho/
│   │   ├── root-app.yaml            # Application instalada manualmente 1x no hub HO
│   │   ├── app-ocm-config.yaml      # Infraestrutura lógica OCM (namespaces, placements)
│   │   ├── appset-governance.yaml   # ApplicationSet: autodescobre governance/*
│   │   ├── app-haproxy-bu-a-ho.yaml # HAProxy Ingress no cluster bu-a-ho
│   │   ├── app-haproxy-bu-b-ho.yaml # HAProxy Ingress no cluster bu-b-ho
│   │   └── kustomization.yaml
│   └── pr/                          # Mesma estrutura para produção
│       ├── app-haproxy-bu-a-pr.yaml
│       ├── app-haproxy-bu-b-pr.yaml
│       └── ...
│
├── config/                          # ⚙️ Infraestrutura lógica do Hub OCM
│   ├── ho/
│   │   ├── namespace.yaml           # Namespace "ocm-policies-ho" no Hub
│   │   ├── acm-placement-configmap.yaml  # Duck-typing: ensina ArgoCD a ler PlacementDecisions
│   │   ├── bu-a-placement.yaml      # Placement OCM: seleciona env=ho + bu=bu-a
│   │   ├── bu-b-placement.yaml      # Placement OCM: seleciona env=ho + bu=bu-b
│   │   ├── placement.yaml           # Placement compartilhado (governance/PolicySets)
│   │   ├── clusterset-binding.yaml
│   │   ├── binding-*.yaml (×6)      # Liga cada PolicySet ao Placement ho
│   │   └── kustomization.yaml
│   └── pr/                          # Mesma estrutura para produção
│
├── domains/                         # 📁 ApplicationSets e Bootstraps por BU
│   ├── bu-a/
│   │   ├── bootstrap/ho/            # root-bu-a-ho.yaml → domains/bu-a/ho
│   │   ├── bootstrap/pr/            # root-bu-a-pr.yaml → domains/bu-a/pr
│   │   ├── ho/
│   │   │   ├── appset-tools-ho.yaml # ApplicationSet: clusterDecisionResource + Matrix Git
│   │   │   ├── placement.yaml       # Placement bu-a-placement-ho no namespace argocd
│   │   │   └── kustomization.yaml
│   │   └── pr/
│   │       ├── appset-tools-pr.yaml
│   │       └── placement.yaml
│   └── bu-b/                        # Mesma estrutura para BU-B
│
├── governance/                      # ⚖️ Políticas OCM (auto-descobertas pelo ArgoCD)
│   ├── capacity/                    # ResourceQuotas, LimitRanges
│   ├── compliance/                  # Controles NIST, CIS
│   ├── infrastructure/              # StorageClasses, Operators
│   ├── observability/               # Monitoramento e log
│   ├── platform/                    # Namespaces base, RBAC
│   └── security/                    # Pod Security Standards, NetworkPolicies
│
└── docs/                            # 📚 ADRs e guias de arquitetura
    ├── ADR-001-three-repo-gitops-strategy.md
    ├── ADR-002-single-branch-environment-per-directory.md
    ├── ADR-003-ocm-over-rhacm.md
    ├── ADR-004-argocd-as-delivery-tool.md
    └── governance-categorization-guide.md
```

---

## Arquitetura de Deploy por BU — `clusterDecisionResource`

A segregação de deployments por BU é feita através de um **Matrix Generator** no ApplicationSet, combinando um **Git Generator** (autodescobre pastas de tools) com um **clusterDecisionResource Generator** (seleciona o cluster-alvo via OCM Placement):

```
gitops-bu-a/ho/tools/sample-app/    ← Git Generator detecta pasta
         +
bu-a-placement-ho (Placement OCM)  ← seleciona apenas bu-a-ho
         ↓
ApplicationSet gera Application     → destino: name: bu-a-ho
         ↓
ArgoCD deploya no cluster bu-a-ho   ← usando o cluster secret correto
```

### Peças-chave

| Recurso | Namespace | Função |
|---|---|---|
| `acm-placement-configmap.yaml` | `argocd` | Duck-typing: `kind: placementdecisions` — ensina ArgoCD a consultar a API OCM |
| `bu-a-placement-ho` (Placement) | `argocd` | Seleciona clusters com `env=ho` + `bu=bu-a` |
| `appset-tools-ho.yaml` (ApplicationSet) | `argocd` | Matrix Generator: Git × clusterDecisionResource |
| ArgoCD cluster secret `bu-a-ho-secret` | `argocd` | TLS autenticado, gerado pelo `connect-clusters.sh` |

> ⚠️ O `kind` no ConfigMap `acm-placement` **deve ser `placementdecisions`** (minúsculo, plural). ArgoCD usa duck-typing para construir o caminho de API discovery.

---

## Como Funciona — Fluxo Completo

### 1. Bootstrap (execução única por ambiente)

```bash
# Ambiente HO
kubectl apply -f bootstrap/ho/root-app.yaml --context kind-gerencia-ho

# Ambiente PR
kubectl apply -f bootstrap/pr/root-app.yaml --context kind-gerencia-pr
```

### 2. Autodiscovery de Políticas

O `appset-governance.yaml` usa um **Git Generator** que varre `governance/*`:

```yaml
generators:
  - git:
      directories:
        - path: governance/*    # detecta cada subpasta automaticamente
```

Para cada pasta, o ArgoCD cria uma `Application` apontando para `overlays/ho` ou `overlays/pr`.

### 3. Distribuição OCM para Clusters Worker

```
Git (governance/platform/)
        │
        │  ArgoCD detecta pasta → cria Application
        ▼
   ArgoCD (em gerencia-ho)
        │  aplica Policy + PolicySet em ocm-policies-ho
        ▼
   OCM Hub
        ├──▶ bu-a-ho (env=ho, bu=bu-a)  ✅ Compliant
        └──▶ bu-b-ho (env=ho, bu=bu-b)  ✅ Compliant
```

---

## Acesso aos Ambientes

### Clusters de Gerência

| Ferramenta | HO | PR |
|---|---|---|
| ArgoCD | http://argocd-ho.local | http://argocd-pr.local |
| Headlamp | http://headlamp-ho.local | http://headlamp-pr.local |

### Clusters das BUs

| Cluster | Sample App | Headlamp |
|---|---|---|
| `bu-a-ho` | http://sample-app-bu-a-ho.local | http://headlamp-bu-a-ho.local |
| `bu-a-pr` | http://sample-app-bu-a-pr.local | http://headlamp-bu-a-pr.local |
| `bu-b-ho` | http://sample-app-bu-b-ho.local | http://headlamp-bu-b-ho.local |
| `bu-b-pr` | http://sample-app-bu-b-pr.local | http://headlamp-bu-b-pr.local |

> Os nomes DNS são resolvidos via `/etc/hosts`. Após reboot do Docker, rode `./scripts/fix-ips.sh` no `gitops-ocm-foundation`.

---

## Controle de Acesso (CODEOWNERS)

| Caminho | Aprovadores | Motivo |
|---|---|---|
| `config/pr/**` | `@rdgoarruda` | Produção — controle restrito |
| `bootstrap/pr/**` | `@rdgoarruda` | Produção — controle restrito |
| `governance/**` | `@rdgoarruda` | Afeta todos os clusters |
| `config/ho/**` | `@rdgoarruda` | Homologação |
| `bootstrap/ho/**` | `@rdgoarruda` | Homologação |

---

## Fluxo de Promoção HO → PR

```
Mudança de política
        │
        ├─── PR para config/ho/ + governance/  →  merge em main
        │         └── ArgoCD (gerencia-ho) sincroniza → OCM distribui para workers HO
        │                   └── Validação / aprovação humana
        │
        └─── PR para config/pr/                →  merge em main (exige @sre-team)
                  └── ArgoCD (gerencia-pr) sincroniza → OCM distribui para workers PR
```

---

## Referências de Arquitetura (ADRs)

| # | Decisão | Status |
|---|---|---|
| [ADR-001](docs/ADR-001-three-repo-gitops-strategy.md) | Estratégia de 3 Repositórios GitOps | ✅ Aceito |
| [ADR-002](docs/ADR-002-single-branch-environment-per-directory.md) | Branch Única + Overlays por Ambiente | ✅ Aceito |
| [ADR-003](docs/ADR-003-ocm-over-rhacm.md) | OCM em vez de RHACM | ✅ Aceito |
| [ADR-004](docs/ADR-004-argocd-as-delivery-tool.md) | ArgoCD como Ferramenta de Entrega | ✅ Aceito |
