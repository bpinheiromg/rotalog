# Rotalog – Meta-Repository

This repository aggregates the five services that compose the Rotalog system:

* **rotalog-api-frotas** – Java / Spring Boot (vehicle & routing)
* **rotalog-api-entregas** – Node / Express (delivery dispatch)
* **rotalog-api-notificacoes** – Java / Spring Boot (notifications)
* **rotalog-workspace** – Nx utilities & shared scripts
* **rotalog-frontend** – Nx monorepo (Angular 18 + React 18)

## Cloning
```bash
git clone --recurse-submodules https://github.com/bpinheiromg/rotalog.git
cd rotalog
```

## Building / testing
See the CI workflow (`.github/workflows/ci.yml`) for the exact commands for each submodule.
