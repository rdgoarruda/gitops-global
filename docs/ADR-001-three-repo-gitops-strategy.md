# ADR-001: Estratégia de 3 Repositórios GitOps

- **Status:** Aceito
- **Data:** 2026-02-21
- **Decisores:** Time de Plataforma

---

## Contexto

Ao estruturar uma plataforma multi-cluster com múltiplas unidades de negócio (BUs),
é necessário definir onde e como os artefatos GitOps são armazenados. As opções vão de
um monorepo único até repositórios completamente separados por cluster.

O ambiente alvo consiste em:
- Um cluster de gerenciamento central (`gerencia-global`)
- Clusters worker por ambiente de negócio (`nprod-bu-x`, `prod-bu-x`)
- Equipes distintas: Infraestrutura, Plataforma/SecOps, e Desenvolvimento de Aplicações

---

## Decisão

**Adotamos a estrutura de 3 repositórios com responsabilidade claramente separada:**

| Repositório | Responsabilidade | Ferramental |
|---|---|---|
| `01-infra-terraform` | Provisionamento de clusters e cloud resources | Terraform + Atlantis |
| `02-platform-policies` | Governança, segurança e configurações mandatórias | ArgoCD + OCM/RHACM |
| `03-workloads` | Aplicações de negócio e ferramentas por BU | ArgoCD ApplicationSet |

---

## Consequências

**Positivas:**
- **Separação de responsabilidades (SoC):** Times de Infra, Plataforma e Dev operam de forma independente sem conflitos de merge
- **Blast radius controlado:** Um erro em `03-workloads` não pode afetar políticas de segurança em `02-platform-policies`
- **Permissões granulares:** O repositório de políticas pode ter acesso restrito ao time de SecOps, enquanto workloads são gerenciados pelo time de produto
- **Audit trail limpo:** O histórico de um repositório reflete apenas as mudanças do seu domínio

**Negativas / Trade-offs aceitos:**
- Necessita de **3 fluxos de PR** separados para mudanças que afetam camadas diferentes
- **Coordenação entre repositórios** pode ser necessária (ex: uma mudança de infra deve preceder uma mudança de políticas)

---

## Alternativas Consideradas

### ❌ Monorepo único
Todas as configurações em um único repositório.

**Por que descartado:**
- Mistura de responsabilidades cria conflitos entre times
- Permissões de acesso são all-or-nothing: um dev de aplicação veria toda a configuração de segurança
- Pipeline de CI/CD mais complexo para determinar o que mudou e o que precisa ser aplicado

### ❌ Repositório por cluster
Um repo para `nprod-bu-x`, outro para `prod-bu-x`, etc.

**Por que descartado:**
- **Drift inevitável:** configurações divergem entre repos ao longo do tempo
- Sem mecanismo nativo de "promoção" (aplicar a mesma mudança em múltiplos lugares)
- Não escala: com 10+ clusters, a gestão se torna inviável

### ❌ 2 repositórios (Infra + App)
Agrupando Políticas junto com Workloads ou Infra.

**Por que descartado:**
- Políticas de governança (SecOps) têm ciclo de vida e revisores **completamente diferentes** de workloads
- Uma política de NetworkPolicy não deve passar pelo mesmo processo de review de um deploy de aplicação

---

## Referências

- [GitOps Working Group — Best Practices](https://github.com/open-gitops/documents/blob/main/PRINCIPLES.md)
- [ArgoCD User Guide — Repository Structure](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)
- [Flux Multi-Tenancy Guide](https://fluxcd.io/flux/guides/multi-tenancy/)
- [Google Anthos Config Management — Multi-Repo](https://cloud.google.com/anthos-config-management/docs/how-to/multiple-repositories)
- [Weaveworks GitOps Guide](https://www.weave.works/technologies/gitops/)
