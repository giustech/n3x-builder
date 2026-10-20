# ADR-008: `govulncheck` só avisa

Status: Aceito
Data: 2026-10-20

## Contexto
O HLD de plataforma prevê o `govulncheck` em todo build, mas não define o efeito de uma vulnerabilidade encontrada. Foram avaliadas três opções: falhar só quando a função vulnerável é chamada, só avisar, e falhar em qualquer vulnerabilidade.

## Decisão
- O `govulncheck` roda em todo `n3x-plugin build` e só avisa, sem falhar
- Os avisos aparecem no log e no resumo final e são gravados no campo `warnings` do manifesto e em `/out/manifest.json`, com ID, módulo e se a função é chamada

## Justificativa
- Uma vulnerabilidade numa dependência de terceiro não bloqueia o deploy do operador
- O registro no manifesto permite que o painel e a CI do operador mostrem e decidam

## Trade-off
- Um plugin com vulnerabilidade chamada pode chegar à produção; a decisão fica com o operador

## Requisitos relacionados
FR-015; segurança nos requisitos não funcionais.
