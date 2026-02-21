# gitops-global

Repositório central de governança e infraestrutura multi-cluster.

Este repositório utiliza uma estratégia **GitOps com Autodiscovery (ApplicationSets)**. As categorias de governança são módulos autossuportados que o ArgoCD detecta automaticamente, facilitando a expansão da plataforma sem configurações manuais repetitivas.

## Estrutura do Repositório

```text
gitops-global/
├── bootstrap/             # 🚀 Ponto de entrada do ArgoCD (App-of-Apps)
│   ├── nprod/             # Root-App e ApplicationSet para Não-Produção
│   └── prod/              # Root-App e ApplicationSet para Produção
├── governance/            # ⚖️ Categorias de Políticas (OCM)
│   ├── platform/          # Políticas base, RBAC, etc.
│   ├── security/          # Hardening e Security Standards
│   ├── observability/     # Agentes de monitoramento e logs
│   ├── capacity/          # Quotas e limites
│   └── [categoria]/       # Novas categorias detectadas via Git Discovery
└── config/                # ⚙️ Configurações de Ambiente (Hub)
    ├── nprod/             # Namespaces, Placements e Bindings para nprod
    └── prod/              # Namespaces, Placements e Bindings para prod
```

## Lógica de Funcionamento

1.  **Bootstrap & Discovery**: O ArgoCD monitora as pastas em `bootstrap/<env>`. O arquivo `appset-governance.yaml` usa um **Git Generator** que varre a pasta `governance/*`. Para cada subpasta encontrada, o ArgoCD cria automaticamente uma `Application`.
2.  **Governance**: Cada subpasta (ex: `platform`) contém apenas os manifests de `Policy`, `PolicySet` e o `kustomization.yaml`. **Não há necessidade de manifests do ArgoCD aqui**, pois o Discovery as gerencia.
3.  **Config**: Contém a infraestrutura lógica do Hub. Aqui definimos os `Namespaces` de destino no Hub, os `Placement` (quem recebe) e os `PlacementBinding` (quem liga as políticas aos clusters).

## Como Adicionar uma Nova Categoria de Governança

1.  Crie uma nova pasta em `governance/<nova-categoria>`.
2.  Adicione seus manifests (`policy-*.yaml`, `policyset.yaml`).
3.  Crie um `kustomization.yaml` na nova pasta listando os recursos.
4.  **Vinculação**: Em `config/<env>/`, crie um `PlacementBinding` vinculando o novo `PolicySet` ao `Placement` do ambiente e adicione-o ao `kustomization.yaml` local.
5.  **Pronto!** O ArgoCD detectará a nova pasta automaticamente e o OCM fará a distribuição baseada no binding.

---

## Referências de Arquitetura (ADRs)

- [ADR-001: Estratégia de 3 Repositórios](docs/ADR-001-three-repo-gitops-strategy.md)
- [ADR-002: Branch Única + Overlays](docs/ADR-002-single-branch-environment-per-directory.md)
- [ADR-003: OCM vs RHACM](docs/ADR-003-ocm-over-rhacm.md)
- [ADR-004: ArgoCD como Ferramenta de Entrega](docs/ADR-004-argocd-as-delivery-tool.md)
- [Guia de Categorização](docs/governance-categorization-guide.md)
