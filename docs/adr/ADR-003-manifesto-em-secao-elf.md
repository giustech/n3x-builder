# ADR-003: Manifesto JSON na seção ELF `.n3x_manifest`

Status: Aceito
Data: 2026-10-20

## Contexto
O sidecar lê o manifesto do `.so` sem carregá-lo (HLD do `n3x-sidecar`), e o controller e o plugin core leem o manifesto do binário para validar configs e registrar o inventário. Era preciso um formato único e um lugar no arquivo legível sem executar código.

## Decisão
- Manifesto em JSON, gravado na seção ELF `.n3x_manifest` com `objcopy --add-section`, no `.so` e no binário do controller
- Campos: `format`, `kind`, `target` (só no sidecar), `arch`, `builder`, `runtime` (módulo e versão), `sdk`, `go`, `built_at`, `plugins` (ID, módulo, versão, metades, `n3x-plugin.yaml` e JSON Schema), `warnings` e `integrity`
- `plugins` inclui todos os plugins da lista, mesmo os sem a metade daquele artefato
- O campo `format` só cresce por adição: o runtime ignora campo desconhecido e recusa `format` maior do que conhece
- Uma cópia legível sai em `/out/manifest.json`

## Justificativa
- Leitura com `debug/elf`, sem carregar o `.so`
- O mesmo leitor serve ao sidecar, ao controller e ao plugin core
- JSON dispensa gerador de código e é fácil de inspecionar na CI

## Trade-off
- Depende do `objcopy` na imagem de build
- O manifesto cresce com os schemas de todos os plugins

## Requisitos relacionados
FR-001, FR-006, FR-007, FR-011, FR-015, FR-016.
