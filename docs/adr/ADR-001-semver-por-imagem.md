# ADR-001: Semver próprio por imagem e regra de compatibilidade no runtime

Status: Aceito
Data: 2026-10-20

## Contexto
O operador escreve duas linhas `FROM` no Dockerfile: a imagem de build (`sidecar-builder` ou `controller-builder`) e a imagem de runtime (`sidecar` ou `controller`). O HLD de plataforma deixou para este HLD decidir o que significa a versão na tag. A alternativa era uma versão única da plataforma em todas as imagens `n3x/*`.

## Decisão
- Cada imagem tem semver próprio: `sidecar-builder`, `controller-builder`, `sidecar` e `controller`
- O manifesto embutido grava o módulo e a versão do runtime compilado (`core/` no `.so`, `host/` no controller)
- Na subida, o `n3x run` e o controller aceitam a mesma major com minor da imagem de runtime igual ou maior que a do manifesto; fora disso, recusam e dizem qual versão usar
- O README do builder publica a tabela dos pares testados na CI

## Justificativa
- Cada repositório evolui no próprio ritmo, sem release conjunta
- Um patch numa imagem não obriga a publicar todas as outras
- A conferência na subida impede que um par errado receba tráfego

## Trade-off
- O operador precisa combinar duas versões; a mensagem de erro e a tabela de pares reduzem o esforço
- A regra de compatibilidade precisa ser implementada igual no sidecar e no controller

## Requisitos relacionados
FR-001, FR-011, FR-015; regras de versionamento entre repositórios do HLD de plataforma.
