# ADR-005: Lista de plugins só por `RUN n3x-plugin add`

Status: Aceito
Data: 2026-10-20

## Contexto
O sidecar e o controller precisam receber a mesma lista de plugins, escrita em dois Dockerfiles. Foi avaliado aceitar também um arquivo de lista (`n3x-plugin add -f plugins.txt`), compartilhado pelos dois.

## Decisão
- A lista de plugins é declarada só com `RUN n3x-plugin add <módulo>@<versão>`, uma linha por plugin, com versão exata
- Não há arquivo de lista na v1
- A diferença entre as listas do sidecar e do controller continua sendo detectada pelo manifesto e alertada no painel (FR-006)

## Justificativa
- Mantém o fluxo do PRD, explícito e legível no próprio Dockerfile
- Cada linha vira uma camada própria e aproveita o cache do Docker

## Trade-off
- A lista fica escrita duas vezes, e a divergência só aparece depois do deploy, como alerta

## Requisitos relacionados
FR-006, FR-015.
