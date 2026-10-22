# Especificações do n3x-builder

Ordem de implementação das especificações. Cada diretório agrupa FDDs de `docs/fdd/` e deve ser processado pelo Spec Kit em sessões separadas (`specify`, `clarify`, `plan`, `tasks`, `implement`), na ordem abaixo.

| Ordem | Diretório | FDDs | Depende de | Por que nesta ordem |
| --- | --- | --- | --- | --- |
| 1 | `01_catalogo_resolucao_e_erros` | BLD-01, BLD-02 | — | Base de tudo: catálogo de alvos, `add`, workspace, resolução, compatibilidade e relator de erros. As duas FDDs dependem uma da outra, por isso ficam juntas. |
| 2 | `02_analise_geracao_e_compilacao` | BLD-06, BLD-03 | 1 | Passos 4 a 6 do `build`: análise estática, geração do código e compilação. Usam o workspace, a lista checada e o relator do grupo 1. |
| 3 | `03_manifesto_simbolos_e_carga` | BLD-04, BLD-05 | 2 | Passos 7 a 9: símbolos, teste de carga e manifesto embutido. Precisam do artefato compilado no grupo 2. A BLD-04 fornece a porta `Stager` que a BLD-05 usa. |
| 4 | `04_imagens_e_matriz_ci` | BLD-07 | 1, 2, 3 | Empacota e publica o pipeline completo em imagens e roda a matriz de CI. Só faz sentido com o pipeline pronto. |

## Pendências não bloqueantes

- Valores reais do catálogo (digest, commit, glibc e pseudo-versão do SDK Go do Envoy), que dependem da fase 0.
- Capacidades do alvo `istio-1.31`, que dependem do spike.
- Hipóteses V-01 a V-09 da BLD-05, que dependem do comportamento real do Envoy.
- Se o aviso `archive_unverified` do `add --archive` vai para o `warnings` do manifesto (BLD-01 e BLD-04).
- Algumas FDDs citam FDDs de outros repositórios (SDK e sidecar), fora do escopo deste repositório.
