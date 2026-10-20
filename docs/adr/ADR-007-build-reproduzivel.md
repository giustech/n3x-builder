# ADR-007: Build reproduzível

Status: Aceito
Data: 2026-10-20

## Contexto
Sem lockfile, assinatura nem SBOM na v1, a única forma de provar que um `.so` em produção veio de um Dockerfile é recompilar e comparar. Isso só funciona se o mesmo Dockerfile gerar o mesmo hash em qualquer máquina.

## Decisão
- `go build -trimpath -ldflags=-buildid=` no sidecar e no controller
- Código gerado (`main.go`, registro e schemas) em ordem fixa e formatado com `go/format`
- `built_at` fica no manifesto, fora do hash (ADR-002)
- A CI do builder compila o mesmo conjunto duas vezes, em amd64 e arm64, e compara os hashes

## Justificativa
- Hash igual vira prova de origem
- Toolchain travado na imagem já elimina a maior parte das diferenças

## Trade-off
- Caminhos e flags do cgo com `zig cc` podem quebrar a reprodutibilidade em arm64; risco registrado no HLD
- A CI fica mais longa por compilar duas vezes

## Requisitos relacionados
FR-015; segurança nos requisitos não funcionais.
