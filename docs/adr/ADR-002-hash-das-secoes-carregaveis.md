# ADR-002: Hash SHA-256 das seções carregáveis, sem assinatura na v1

Status: Aceito
Data: 2026-10-20

## Contexto
O HLD do `n3x-sidecar` confere a integridade do `.so` contra o hash do manifesto na subida, antes de carregá-lo, e deixou o formato do hash para este HLD. O manifesto fica dentro do próprio arquivo, então o hash não pode incluir a seção dele.

## Decisão
- Algoritmo SHA-256, sem chave e sem assinatura
- Escopo `alloc-sections`: conteúdo das seções ELF com a flag `SHF_ALLOC`, concatenadas na ordem do arquivo
- A seção `.n3x_manifest` não tem `SHF_ALLOC` e fica fora do cálculo, assim como o `built_at`
- O runtime refaz a conta com `debug/elf`, sem remover nada do arquivo
- O mesmo formato vale para o binário do controller

## Justificativa
- Simples, sem gestão de chave
- Detecta arquivo corrompido ou trocado por engano
- Recalculável sem reescrever o arquivo, o que evita diferenças de bytes geradas pelo `objcopy`

## Trade-off
- Não protege contra quem altera o arquivo de propósito e recalcula o hash; isso exige assinatura, fora de escopo na v1
- Mudanças em seções não carregáveis (ex.: símbolos de debug) não são detectadas, o que não afeta a execução

## Requisitos relacionados
FR-001, FR-015; segurança nos requisitos não funcionais.
