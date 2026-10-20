# ADR-006: Códigos de erro estáveis `N3X-Bxxx`

Status: Aceito
Data: 2026-10-20

## Contexto
O PRD lista os cenários de falha do build e pede mensagens claras. Sem código, uma mensagem em texto livre é difícil de buscar, de citar em issue e de tratar na CI do operador.

## Decisão
- Cada cenário tem um código fixo, de `N3X-B001` a `N3X-B013`, que nunca muda de significado; códigos novos só por adição
- Cada mensagem traz o código, o motivo, uma linha "como resolver" e o link para `docs/erros.md`
- Exit code por categoria: 1 uso, 2 resolução, 3 compatibilidade, 4 análise estática, 5 compilação, 6 símbolos ou carga, 7 manifesto
- O build termina a etapa em que falhou e lista todas as falhas dela

## Justificativa
- Fácil de buscar, de documentar e de automatizar
- O operador corrige tudo de uma vez, sem repetir o build a cada falha

## Trade-off
- O catálogo de erros precisa ser mantido junto com o código e a documentação

## Requisitos relacionados
FR-012, FR-013, FR-015.
