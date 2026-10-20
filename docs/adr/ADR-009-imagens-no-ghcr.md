# ADR-009: Imagens publicadas no GHCR

Status: Aceito
Data: 2026-10-20

## Contexto
O PRD exige um registro de containers para as imagens `n3x/*` e não define qual. Os repositórios já ficam no GitHub.

## Decisão
- Imagens publicadas no GitHub Container Registry: `ghcr.io/giustech/n3x/sidecar-builder:<semver>-<alvo>`, `ghcr.io/giustech/n3x/controller-builder:<semver>`, `ghcr.io/giustech/n3x/sidecar:<semver>-<alvo>` e `ghcr.io/giustech/n3x/controller:<semver>`
- Publicação pela CI do GitHub Actions de cada repositório, com `docker buildx` para amd64 e arm64
- Tags imutáveis, nunca apagadas

## Justificativa
- Mesmo lugar dos repositórios e da CI, sem conta extra
- Permissões de publicação herdadas da organização no GitHub

## Trade-off
- Dependência de um único provedor para código, CI e imagens
- Limites de download do GHCR para imagens públicas podem exigir espelho no registro do operador

## Requisitos relacionados
FR-015; dependência externa "Registro de containers e repositórios" do PRD.
