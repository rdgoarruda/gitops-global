# gitops-global

Repositório central de governança e infraestrutura multi-cluster.

Este repositório utiliza uma estratégia **GitOps Descentralizada**. Cada categoria de governança ou setup é um módulo autossuficiente, permitindo uma gestão clara e escalável de políticas OCM e configurações de cluster.

## Estrutura do Repositório

```text
gitops-global/
├── bootstrap/             # 🚀 Ponto de entrada do ArgoCD (App-of-Apps)
│   ├── nprod/             # Root-App para os ambientes de Não-Produção
│   └── prod/              # Root-App para os ambientes de Produção
├── setup/                 # 🏗️ Preparação da Infraestrutura do Hub (OCM)
│   ├── nprod/             # Namespaces e Placements base para nprod
│   └── prod/              # Namespaces e Placements base para prod
├── governance/            # ⚖️ Categorias de Políticas (OCM)
│   └── platform/          # Políticas de Plataforma base (Namespaces, etc.)
│       ├── argocd-app-*.yaml # Definição da entrega no ArgoCD
│       └── policy-*.yaml     # Manifests das políticas OCM
└── config/                # ⚙️ Configurações Específicas de Ambiente
    ├── nprod/             # Bindings e ClusterSetBindings vinculando políticas aos clusters
    └── prod/              # Bindings e ClusterSetBindings vinculando políticas aos clusters
```

## Lógica de Funcionamento

1.  **Bootstrap**: O ArgoCD monitora a pasta `bootstrap/<env>`. O `root-app` carrega via Kustomize o `setup` do ambiente e as `Applications` das categorias de governança necessárias.
2.  **Governance**: Cada subpasta (ex: `platform`) é autossuficiente. Ela contém a política e o objeto `Application` que diz ao ArgoCD como entregá-la.
3.  **Config**: Contém os `PlacementBinding` e `ManagedClusterSetBinding`. É aqui que decidimos **quais** políticas de qual categoria serão aplicadas em **quais** clusters do ambiente.

## Como Adicionar uma Nova Categoria de Governança

1.  Crie uma nova pasta em `governance/<categoria>`.
2.  Adicione seus manifests de `Policy` e `PolicySet`.
3.  Crie os manifests de `Application` (ex: `argocd-app-nprod.yaml`) apontando para a pasta da categoria.
4.  No arquivo `bootstrap/<env>/kustomization.yaml`, adicione o path para o novo `argocd-app-<env>.yaml`.
5.  Em `config/<env>/`, adicione os `PlacementBinding` especificando quais clusters devem receber essa nova categoria.

---

## Referências de Arquitetura (ADRs)

- [ADR-001: Estratégia de 3 Repositórios](docs/ADR-001-three-repo-gitops-strategy.md)
- [ADR-002: Branch Única + Overlays](docs/ADR-002-single-branch-environment-per-directory.md)
- [ADR-003: OCM vs RHACM](docs/ADR-003-ocm-over-rhacm.md)
- [ADR-004: ArgoCD como Ferramenta de Entrega](docs/ADR-004-argocd-as-delivery-tool.md)
