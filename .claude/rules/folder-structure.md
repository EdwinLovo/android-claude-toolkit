---
name: folder-structure
description: Clean-architecture package layout for a single-module Android app (domain / data / presentation)
paths:
  - "**/*.kt"
---

# Folder structure

Single module (`app`) with three top-level packages under `<PKG_ROOT>`.

```
<PKG_ROOT>/
├── domain/
│   ├── model/
│   │   ├── response/<feature>/   ← plain data classes, no framework deps
│   │   ├── request/              ← domain-level request models
│   │   └── uimodel/              ← display-oriented models
│   ├── repository/               ← interfaces only
│   └── usecase/<feature>/        ← only when logic is non-trivial or reused
│
├── data/
│   ├── di/                       ← ApiModule, DataSourceModule, RepositoryModule
│   ├── mappers/<feature>/        ← network → domain extension functions
│   ├── paging/                   ← PagingSource implementations
│   ├── remote/
│   │   ├── datasource/           ← RemoteDataSource interface + impl (call factories)
│   │   └── <net-lib>/            ← e.g. graphql/ or api/ definitions grouped by feature
│   └── repository/<feature>/     ← implementations (extend BaseRepository)
│
└── presentation/
    ├── delegates/<name>/         ← OPTIONAL. Shared @ViewModelScoped delegates injected by multiple hosts
    │                                (e.g. clientselector/, createclient/). Contract co-located with delegate.
    │                                See rules/delegates.md.
    ├── ui/
    │   ├── navigation/           ← NavigationRoute, ViewModelNav, AppNavHost
    │   ├── components/<category>/← shared Compose components (buttons/, dialogs/, ...)
    │   └── theme/                ← Material3 theme, colors, typography, dimens
    ├── utils/                    ← ErrorEventBus, UiText, ext/, <ICONS>.kt
    └── ux/<feature>/             ← one folder per screen (see rules/screens.md)
                                    Feature-scoped delegates live inside each ux/<feature>/delegates/.
```

## Rules of thumb

- **Domain has no framework imports.** No Android, no Compose, no networking library types.
- **Presentation never imports from `data/`.** It talks to `domain/repository/` interfaces via Hilt.
- **`data/` never returns network types past its own layer.** Mappers convert at the boundary.
- **Feature folder in `ux/` is lowercase, no separators** (e.g. `patientprofile`, not `PatientProfile` / `patient-profile`).
- **Feature folder in `mappers/`, `repository/`, `usecase/` matches the domain, not the screen.** A `catalog/` feature can back multiple screens.

## Common misplacements to fix on sight

- Response model outside `domain/model/response/<feature>/` → move it there
- Mapper that only touches domain models placed in `data/mappers/` → move to `domain/model/<area>/<Name>Mappers.kt`
- Feature-only component placed in `presentation/ui/components/` → move to `presentation/ux/<feature>/components/`
- Reusable component placed in `ux/<feature>/components/` → move to `presentation/ui/components/<category>/`
- Delegate used by multiple hosts placed in a single feature's `ux/<feature>/delegates/` → move to top-level `presentation/delegates/<name>/`
- Feature-only delegate placed in top-level `presentation/delegates/` → move under its host feature's `ux/<feature>/delegates/`
