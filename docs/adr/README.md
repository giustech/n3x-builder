# ADRs do n3x-builder

Decisões de arquitetura registradas no HLD do builder (`../HLD.md`). A numeração é própria deste repositório; ADRs de outros repositórios são citados com o nome dele (por exemplo, "ADR-013 do `n3x-sdk`"). Cada ADR traz contexto, decisão, justificativa, trade-off e requisitos relacionados do PRD (`../prd/PRD-N3X.md`).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-semver-por-imagem.md) | Semver próprio por imagem e regra de compatibilidade no runtime | Aceito |
| [ADR-002](ADR-002-hash-das-secoes-carregaveis.md) | Hash SHA-256 das seções carregáveis, sem assinatura na v1 | Aceito |
| [ADR-003](ADR-003-manifesto-em-secao-elf.md) | Manifesto JSON na seção ELF `.n3x_manifest` | Aceito |
| [ADR-004](ADR-004-simbolos-e-teste-de-carga.md) | Comparação de símbolos e teste de carga na arquitetura nativa | Aceito |
| [ADR-005](ADR-005-lista-por-run-add.md) | Lista de plugins só por `RUN n3x-plugin add` | Aceito |
| [ADR-006](ADR-006-codigos-de-erro.md) | Códigos de erro estáveis `N3X-Bxxx` | Aceito |
| [ADR-007](ADR-007-build-reproduzivel.md) | Build reproduzível | Aceito |
| [ADR-008](ADR-008-govulncheck-so-avisa.md) | `govulncheck` só avisa | Aceito |
| [ADR-009](ADR-009-imagens-no-ghcr.md) | Imagens publicadas no GHCR | Aceito |
