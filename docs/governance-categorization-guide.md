# Guia de Categorização de Políticas (OCM)

Este documento define os critérios para classificar novas políticas de governança no ecossistema multi-cluster.

## Hierarquia de Governança

As políticas são organizadas em **Módulos (Categorias)**. Cada módulo representa um domínio de responsabilidade ou um requisito técnico específico.

### 1. Platform (Plataforma)
Foco na estrutura básica do cluster necessária para que outros times operem.
- **Exemplos**: Namespaces base, RBAC de times globais, ServiceAccounts de infra.
- **Quando usar**: Se a política prepara o ambiente para o uso geral.

### 2. Observability (Observabilidade)
Garante a telemetria e o monitoramento funcional de todos os clusters.
- **Exemplos**: Log Forwarding, configuração de agentes Prometheus/Loki, dashboards base.
- **Quando usar**: Se a política trata de visibilidade ou coleta de dados.

### 3. Security (Segurança)
Hardening do cluster e proteção de workloads.
- **Exemplos**: NetworkPolicies, Pod Security Standards (PSS), Image Registry Whitelists.
- **Quando usar**: Se o objetivo é reduzir a superfície de ataque ou restringir ações maliciosas.

### 4. Capacity (Capacidade)
Controle de recursos e sustentabilidade do cluster.
- **Exemplos**: ResourceQuotas, LimitRanges, PriorityClasses.
- **Quando usar**: Se a política evita o "noisy neighbor" ou gerencia limites de consumo.

### 5. Compliance (Conformidade)
Auditoria e aplicação de benchmarks externos/legais.
- **Exemplos**: Controles NIST SP 800-53, CIS Hardening, Requisitos de Auditoria.
- **Quando usar**: Se a política é derivada de um framework de conformidade externo.

### 6. Infrastructure (Infraestrutura)
Gestão do ciclo de vida dos componentes do cluster.
- **Exemplos**: Subscriptions (OLM), CatalogSources, MachineConfigs.
- **Quando usar**: Se a política gerencia a instalação de ferramentas ou configurações de hardware/S.O.

---

## Como Criar uma Nova Política

1.  **Identifique a Categoria**: Use as definições acima.
2.  **Crie a Policy**: Adicione o manifest `policy-<nome>.yaml` na pasta correspondente.
3.  **Atualize o PolicySet**: Toda categoria deve ter um `policyset.yaml` que agrupa suas políticas. Isso facilita o binding.
4.  **Crie o Vínculo (Config)**: Em `config/<env>/`, crie um `PlacementBinding` vinculando o `PolicySet` da categoria ao `Placement` do ambiente.

## Boas Práticas
- **Dinamismo**: Nunca coloque `namespace` fixo dentro das políticas na pasta `governance/`. Deixe que o ArgoCD/Kustomize injete o namespace de destino.
- **Atomicidade**: Uma política deve fazer apenas uma coisa (ex: Validar Registro de Imagem ≠ Validar NetworkPolicy).
