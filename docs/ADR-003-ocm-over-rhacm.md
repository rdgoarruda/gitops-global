# ADR-003: Open Cluster Management (OCM) em vez de Red Hat ACM (RHACM)

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

Para implementar governança multi-cluster (distribuição de políticas, conformidade, RBAC
centralizado) precisávamos de uma **engine de gestão de clusters** que permita:

1. Registrar múltiplos clusters em um Hub central
2. Distribuir políticas (_Configuration Policies_) para clusters selecionados por label
3. Monitorar conformidade em tempo real

A escolha estava entre o produto comercial **Red Hat Advanced Cluster Management (RHACM)**
e o projeto upstream open-source **Open Cluster Management (OCM)**.

---

## Decisão

**Para o ambiente de laboratório local (kind), adotamos o OCM.**

Em ambientes de produção reais, o RHACM **seria a escolha** se a organização já usa
Red Hat OpenShift — dado que vem com suporte comercial, UI avançada e integrações extras.

---

## Quando usar cada um

| Critério | OCM | RHACM |
|---|---|---|
| Ambiente | Lab, CI, cloud-agnostic | Produção OpenShift |
| Recursos de máquina | Leve (~256MB) | Pesado (~8GB+) |
| Suporte | Comunidade CNCF | Red Hat (pago) |
| UI | Nenhuma nativa | Console completo |
| Licença | Apache 2.0 | Comercial |
| API de Policies | ✅ Idêntica ao RHACM | ✅ Completa |
| Roadmap | Base do RHACM | Superset do OCM |

---

## Consequências

**Positivas:**
- **Viabilidade local:** RHACM requer 3+ nós de worker com 8GB+ RAM — inviável em ambiente kind
- **API idêntica:** As `Policy`, `PlacementRule` e `ManagedCluster` do OCM são **100% compatíveis** com RHACM — o código funcionaria em produção com RHACM sem alteração
- **Custo zero:** OCM é Apache 2.0, sem licenciamento
- **CNCF-backed:** OCM é projeto sandbox do CNCF, com roadmap independente de fornecedor

**Negativas / Trade-offs aceitos:**
- Sem UI nativa de conformidade (dashboard de policies) — compensado pelo uso do ArgoCD UI para visualização
- Suporte apenas via comunidade

---

## Relação OCM ↔ RHACM

```
RHACM (produto Red Hat)
    └── construído 100% sobre OCM (open source)
            ├── cluster-manager (Hub)
            ├── klusterlet (agente nos workers)
            ├── policy-framework
            └── placement API
```

> O OCM É o coração do RHACM. Toda validação feita com OCM é válida para um ambiente RHACM de produção.

---

## Referências

- [OCM — Open Cluster Management (CNCF Sandbox)](https://open-cluster-management.io/)
- [RHACM Product Page — Red Hat](https://www.redhat.com/en/technologies/management/advanced-cluster-management)
- [OCM GitHub](https://github.com/open-cluster-management-io/ocm)
- [RHACM Architecture Overview](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/)
- [OCM Policy Framework](https://open-cluster-management.io/concepts/policy/)
