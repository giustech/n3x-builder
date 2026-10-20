# ADR-004: Comparação de símbolos e teste de carga na arquitetura nativa

Status: Aceito
Data: 2026-10-20

## Contexto
O istio-proxy 1.31 resolve símbolos de forma tardia, então um `.so` errado só cairia na primeira chamada. A comparação de símbolos pega um símbolo ausente, mas não pega uma diferença na versão de ABI que o Envoy confere ao carregar o módulo.

## Decisão
- Comparar sempre os símbolos importados e exportados do `.so` com os do binário do Envoy do alvo, que vem na imagem `sidecar-builder` (`N3X-B011`)
- Na arquitetura nativa, subir o Envoy do alvo por cerca de 2 s com um bootstrap mínimo que carrega o `.so` e encerrar (`N3X-B012`)
- Na compilação cruzada (arm64 em host amd64), rodar só a comparação de símbolos; o teste de carga em arm64 roda na CI nativa do builder para cada imagem

## Justificativa
- Pega no build os dois tipos de erro que só apareceriam na subida ou na primeira chamada
- Evita emulação com QEMU, que deixaria o build lento

## Trade-off
- Cerca de 2 s a mais por build nativo
- No build cruzado, um erro de ABI só aparece no `n3x run`, que falha antes de receber tráfego

## Requisitos relacionados
FR-003, FR-015; riscos de quebra do Istio e do SDK Go do Envoy no PRD.
