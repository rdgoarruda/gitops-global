# ADR-005: Ambiente de Demo — KCD São Paulo 2026

- **Status:** Aceito
- **Data:** 2026-09-01
- **Decisores:** Rodrigo Arruda (Platform Engineer, Banco Bradesco)
- **Sessão KCD:** *Como o Bradesco governa 200+ políticas mandatórias em centenas de clusters com ArgoCD e OCM*

---

## Contexto

Este repositório faz parte de um ambiente de **demonstração técnica** preparado para a sessão no KCD São Paulo 2026. A sessão aborda a estratégia de GitOps adotada no Banco Bradesco para governar centenas de clusters Kubernetes com políticas mandatórias.

O ambiente de demo deve:
1. **Reproduzir a estratégia** sem expor detalhes sensíveis do ambiente real
2. **Ser executável localmente** por qualquer pessoa com Docker + 16 GB RAM
3. **Contar a história** de forma visual e didática durante a apresentação

---

## Mapeamento: Demo → Produção

| Elemento do Demo | Equivalente em Produção | Notas |
|---|---|---|
| 6 clusters Kind | 200+ clusters OpenShift / EKS | Mesma estratégia, escala diferente |
| 2 hubs (`gerencia-ho`, `gerencia-pr`) | Hubs ArgoCD por região/ambiente | Dual-hub para isolamento HO/PR |
| 2 BUs (`bu-a`, `bu-b`) | Dezenas de Business Units | BUs representativas do modelo multi-tenant |
| 6 categorias de políticas | 200+ políticas mandatórias | Políticas representativas por categoria |
| OCM (open-source) | RHACM (Red Hat ACM) | APIs 100% compatíveis — ver [ADR-003](ADR-003-ocm-over-rhacm.md) |
| HAProxy + Kind Ingress | Load Balancers reais | Apenas para exposição local |

---

## Decisões Específicas do Demo

### ✅ Usar `bu-a` / `bu-b` como identificadores de BU
**Motivação:** Nomenclatura abstrata e neutra. O objetivo é demonstrar o **conceito** da estratégia multi-tenant — não expor detalhes organizacionais reais do banco.

### ✅ Manter 6 clusters (2 hubs + 4 workers)
**Motivação:** Número mínimo para demonstrar os dois eixos da estratégia:
- **Eixo ambiente:** HO × PR (isolamento e promoção controlada)
- **Eixo tenant:** BU-A × BU-B (multi-tenancy com Placement OCM)

Com apenas 6 clusters, qualquer pessoa pode reproduzir em um laptop com 16 GB RAM.

### ✅ Single-branch com overlays por diretório (`ho/` e `pr/`)
Em vez de branches por ambiente — ver [ADR-002](ADR-002-single-branch-environment-per-directory.md).

### ✅ Push Model (sem argocd-pull-integration)
O ArgoCD do hub conecta diretamente nos workers via TLS. Adequado para ambientes Kind onde todos os clusters estão na mesma rede Docker. Ver README do `gitops-ocm-foundation` para os trade-offs.

---

## O que foi Simplificado em relação ao Ambiente Real

| Aspecto | Demo | Produção |
|---|---|---|
| Provisionamento | Scripts bash (`create-clusters.sh`) | Terraform + Atlantis |
| Secrets / Credenciais | `.ocm-token-*` locais | Vault / Secrets Manager |
| Quantidade de políticas | ~6 categorias representativas | 200+ políticas reais |
| Observabilidade | Headlamp apenas | Prometheus + Grafana + SIEM |
| Network | Rede Docker local | Segmentação real com NetworkPolicies + firewalls |
| RBAC | CODEOWNERS básico | RBAC granular + SSO |

---

## Como Reproduzir

```bash
# 1. Clone os repositórios
git clone https://github.com/rdgoarruda/gitops-ocm-foundation
git clone https://github.com/rdgoarruda/gitops-global
git clone https://github.com/rdgoarruda/gitops-bu-a
git clone https://github.com/rdgoarruda/gitops-bu-b

# 2. Siga o README do gitops-ocm-foundation para montar o ambiente completo
cd gitops-ocm-foundation
./scripts/create-clusters.sh
./scripts/bootstrap.sh --env ho
./scripts/bootstrap.sh --env pr
./scripts/connect-clusters.sh --env ho
./scripts/connect-clusters.sh --env pr
```

> **Requisitos mínimos:** Docker 24+, Kind 0.20+, kubectl 1.28+, Helm 3.12+, clusteradm 0.8+, 16 GB RAM, 30 GB disco.

---

## Referências

- [Sessão KCD São Paulo 2026](https://community.cncf.io/kcd-sao-paulo/)
- [Open Cluster Management (OCM)](https://open-cluster-management.io/)
- [ArgoCD — ApplicationSet clusterDecisionResource Generator](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/Generators-Cluster-Decision-Resource/)
- [RHACM — Policy Framework](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/)
- [GitOps Principles — OpenGitOps](https://opengitops.dev/)
