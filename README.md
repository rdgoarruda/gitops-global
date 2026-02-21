# gitops-global

Repositório central de governança da plataforma multi-cluster.

Gerenciado pelo **time de Plataforma** — contém as políticas OCM que são distribuídas
automaticamente para os clusters managed (`nprod-bu-x`, `prod-bu-x`) com base em labels.

## Estrutura

```
gitops-global/
├── docs/                          # Architecture Decision Records (ADRs)
├── ocm-policies/
│   ├── base/                      # Políticas compartilhadas por todos os ambientes
│   └── overlays/
│       ├── nprod/                 # Configurações específicas do ambiente nprod
│       └── prod/                  # Configurações específicas do ambiente prod
├── argocd-apps/                   # ArgoCD Applications que apontam para os repos de BU
└── .github/
    └── CODEOWNERS                 # Controle de quem pode aprovar mudanças por path
```

## Fluxo de Governança

```
PR para ocm-policies/overlays/prod/  → requer @sre-team  → merge → ArgoCD sync → OCM distribui para prod-bu-x
PR para ocm-policies/overlays/nprod/ → requer @platform-team → merge → OCM distribui para nprod-bu-x
```

## Referências

- [ADR-001: Estratégia de 3 Repositórios](docs/ADR-001-three-repo-gitops-strategy.md)
- [ADR-002: Branch Única + Overlays](docs/ADR-002-single-branch-environment-per-directory.md)
- [ADR-003: OCM vs RHACM](docs/ADR-003-ocm-over-rhacm.md)
- [ADR-004: ArgoCD](docs/ADR-004-argocd-as-delivery-tool.md)
