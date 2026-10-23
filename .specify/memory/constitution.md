<!--
Sync Impact Report
==================
Versão: template sem valores → 1.0.0 (ratificação inicial)
Fonte: DRAFT_CONSTITUITION.md (PRD 1.5, HLD 1.0, ADR-001 a ADR-009, FDD-BLD-01 a FDD-BLD-07,
specs/README.md e decisões do mantenedor em 2026-10-23)

Princípios (o template tinha 5 vagas; esta versão define 9):
- [PRINCIPLE_1_NAME] → I. Falhar no build, nunca em produção
- [PRINCIPLE_2_NAME] → II. Build reproduzível
- [PRINCIPLE_3_NAME] → III. Determinismo, sem fallback silencioso
- [PRINCIPLE_4_NAME] → IV. Zero credencial
- [PRINCIPLE_5_NAME] → V. Contratos só crescem por adição
- novos: VI. Erros acionáveis; VII. Imagens imutáveis; VIII. Regras do SDK reusadas, nunca
  reimplementadas; IX. Segurança visível

Seções adicionadas:
- [SECTION_2_NAME] → Restrições técnicas e de arquitetura
- [SECTION_3_NAME] → Fluxo de desenvolvimento e portões de qualidade
- Governança preenchida: precedência, idioma, emendas, versionamento, conformidade e revisão

Seções removidas: nenhuma.

Templates e guias:
- ✅ .specify/templates/plan-template.md: portões da Constitution Check, contexto técnico e
  estrutura do repositório
- ✅ .specify/templates/spec-template.md: seções de cenários de falha e de contratos públicos
- ✅ .specify/templates/tasks-template.md: testes obrigatórios (TDD), caminhos Go e tarefas
  transversais (docs/erros.md, cobertura, compilação dupla, golden files, corpus)
- ✅ .specify/templates/checklist-template.md: sem mudança (não depende dos princípios)
- ✅ .claude/skills/speckit-*/ e .agents/skills/speckit-*/: sem mudança. O /speckit-tasks trata
  testes como opcionais; a seção "Testes" desta constituição conta como pedido explícito de TDD.
  O diretório .specify/templates/commands/ não existe neste projeto.
- ✅ README.md: link para esta constituição na seção Development

Pendências (não bloqueiam esta versão):
- ⚠ Documentos de design que ainda divergem desta constituição (vale a constituição até a correção):
  - docs/HLD.md: exit 1 ampliado; zig cc nas duas arquiteturas; as duas metades dos plugins
    compiladas nos dois artefatos; Dockerfile com FROM --platform=$BUILDPLATFORM; índice de
    símbolos; add baixando o ZIP inteiro e com os códigos que emite; host.Run com módulo; schemas
    por metade e vuln_scan em warnings; N3X_VULNDB, /n3x/build, format no catálogo e unsafe proibido
  - docs/HLD.json: remover "mount de secret"
  - ADR-004: índice JSON pré-gerado no lugar do binário do Envoy
  - ADR-006: exit 1 ampliado, ampliação dentro da família e limite de 100 falhas por etapa
  - BLD-04 e BLD-07: remover a regra runtime_unstable da série 0.x
  - BLD-02: remover a hipótese do .netrc e alinhar a mensagem do B001 not_in_sumdb ao --archive
  - BLD-05: remover os resíduos de "secret" e "token"
  - BLD-07: n3xci com cobra e testify; governança do bot conforme "Branches, PRs e releases";
    cabeçalhos da tabela de pares em inglês
  - BLD-03: alinhar as metas por passo sem cache aos 180 s da BLD-07
  - BLD-06: citar ADRs com o nome do repositório
  - Todas as FDDs: portas como interfaces (hexagonal), cobra e testify
  - specs/README.md: declarar os pré-requisitos externos de cada grupo
- ⚠ Fase 0: digest, commit e glibc do Envoy e pseudo-versão do SDK Go do Envoy no catálogo;
  spikes envoy-1.40 e istio-1.31
- ⚠ V-01 a V-09 (BLD-05): símbolos exportados, mensagens de ABI, sinais de subida, glibc e tempos
  reais do Envoy, a confirmar no spike
- ⚠ Destino do aviso archive_unverified no warnings do manifesto (a BLD-01 cria, a BLD-04 não trata)
- ⚠ Nome e versão do pacote artifact do SDK (hipótese: v0.2.0)
- Placeholders adiados: nenhum.
-->
# Constituição do n3x-builder

Regras inegociáveis do `n3x-builder`: as imagens de build OCI, a CLI `n3x-plugin` e o catálogo de
alvos que transformam a lista de plugins do Dockerfile do operador em `libn3x.so` e no binário do
`n3x-controller`, cada um com o manifesto embutido.

Os termos DEVE, NUNCA e DEVERIA têm o sentido de MUST, MUST NOT e SHOULD do RFC 2119. Metas
marcadas como hipótese valem até a medição que as confirma.

## Princípios fundamentais

### I. Falhar no build, nunca em produção

- Todo defeito detectável sem executar o artefato em produção DEVE parar o `add` ou o `build` com
  um código `N3X-Bxxx`: plugin incompatível com o SDK ou com a imagem, ID ou serviço gRPC
  duplicado, código que fura o isolamento entre plugins, símbolo ausente no Envoy do alvo e `.so`
  que não carrega.
- O `build` falha cedo: a etapa que falhou é a última que roda.

Por quê: o Envoy resolve símbolos de forma preguiçosa, então um `.so` errado carrega e só falha na
primeira chamada, já com tráfego. O build é o último ponto em que um erro custa só um ciclo de CI.

### II. Build reproduzível

- A mesma entrada (Dockerfile, plugins e versões) com a mesma imagem de build DEVE gerar seções
  `SHF_ALLOC` idênticas em qualquer host, nativo ou cruzado, amd64 ou arm64 (ADR-007).
- O toolchain vive na imagem: Go com patch fixado por digest e `GOTOOLCHAIN=local`, `zig cc` com a
  glibc do alvo e bases fixadas por digest. Além dos módulos resolvidos, nada de fora da imagem
  entra na compilação.
- A compilação usa `-trimpath` e `-ldflags=-buildid=`. O código gerado (`main.go`, registro e
  schemas) tem ordem fixa e passa por `go/format`. Nenhuma lista que entra no código gerado ou no
  manifesto segue a ordem de um mapa ou do sistema de arquivos.
- O hash de integridade (`alloc-sections`, ADR-002) é o SHA-256 do conteúdo das seções `SHF_ALLOC`,
  em ordem de arquivo. A seção `.n3x_manifest` não é `SHF_ALLOC`, e `built_at` fica fora do hash.

Por quê: sem assinatura nem SBOM na v1, recompilar o mesmo Dockerfile e comparar o hash é a única
prova de origem de um artefato em produção.

### III. Determinismo, sem fallback silencioso

- O `add` só aceita versão exata ou ZIP de módulo. O builder NUNCA troca a versão pedida e NUNCA
  usa `latest` ou `replace`.
- O builder NUNCA entrega artefato parcial: `build` com falha não deixa nada novo em `/out`, e
  `add` com falha não altera o workspace. Repetir um `add` não muda nada.
- O builder NUNCA pula etapa sem registro. Etapa pulada por desenho aparece no log com status
  `skipped`: a etapa 7 inteira na `controller-builder` e o teste de carga em build cruzado, que
  também grava o aviso `load_test_skipped` no manifesto.
- Todo fallback previsto numa FDD é visível: gera aviso no log, no resumo e, quando a FDD pede, no
  manifesto (por exemplo, `vuln_db_fallback`). Fallback não previsto em FDD é proibido.

Por quê: um build que "dá um jeito" esconde a diferença entre o que o operador pediu e o que vai
rodar.

### IV. Zero credencial

Regra absoluta, sem exceção.

- Nenhuma credencial chega ao builder: nem por arquivo, variável de ambiente, argumento, `ARG`,
  montagem de secret ou `.netrc`.
- Plugin privado entra só como ZIP de módulo Go copiado para o contexto do build
  (`add --archive <zip> [--sha256 <hex>]`). Dependências privadas vêm de uma fonte local de
  módulos.
- Só os `ARG` da lista de permissão (seção "Contratos públicos") chegam ao processo `go`.
- O builder NUNCA lê o `n3x.yaml`, a chave do `ENC[...]` nem dados pessoais, e o `n3x-plugin`
  NUNCA importa o pacote `secret/` do SDK.
- Credencial embutida em URL e caminho absoluto do cache de módulos NUNCA aparecem no log, em texto
  ou JSON, nem no manifesto.

Por quê: o builder roda na CI do operador, em qualquer máquina, e tudo o que entra nele pode parar
numa camada ou num cache de imagem. Sem credencial, não há o que vazar.

### V. Contratos só crescem por adição

- Um código `N3X-Bxxx` NUNCA muda de família nem de exit code, e NUNCA é removido ou reutilizado.
  Ele pode ampliar o cenário dentro da mesma família (ADR-006). Código novo só é acrescentado.
- Nomes de regra, tipos de aviso, campos do manifesto, eventos JSON e seus campos e rótulos
  `io.n3x.*` só são acrescentados: NUNCA renomeados, removidos ou com o sentido trocado.
- O manifesto (ADR-003) cresce só por campo novo dentro do `format` 1. Leitores ignoram campo
  desconhecido e recusam `format` maior do que o que conhecem.
- Toda adição a um contrato DEVE atualizar, no mesmo PR, o `docs/erros.md` (quando couber), os
  golden files e a documentação do contrato.

Por quê: a CI do operador decide pelo exit code e pelo evento JSON, e o entrypoint do sidecar, o
host do controller e o plugin core leem o manifesto. Uma renomeação quebra todos eles sem aviso.

### VI. Erros acionáveis

- Toda falha DEVE trazer o código, a regra, o motivo (o que foi exigido e o que havia), uma linha
  "como resolver" e o link para a âncora do código em `docs/erros.md`.
- O `docs/erros.md` é gerado do catálogo de códigos e NUNCA é editado à mão.
- Cada etapa lista todas as falhas dela, até 100, antes de parar.
- Exit 1 (uso, ambiente ou erro interno) não tem código `N3X-Bxxx`. Aviso NUNCA muda o exit code.

Por quê: o autor de plugin e o operador corrigem o problema sem abrir o código do builder, e ver
todas as falhas da etapa evita um ciclo de build por erro.

### VII. Imagens imutáveis

- Tag publicada NUNCA é sobrescrita nem apagada, e não existe tag móvel como `latest` (ADR-009).
- Cada imagem de build tem semver próprio e evolui sozinha (ADR-001): `sidecar-builder` com o
  sufixo do alvo (`<semver>-<alvo>`) e `controller-builder` sem sufixo.
- Imagens base (`golang`, `envoyproxy/envoy` e `istio/proxyv2`) e ações do GitHub Actions são
  fixadas por digest ou SHA.
- A publicação envia por digest, verifica cada plataforma, recusa tag existente e só então cria o
  índice OCI com amd64 e arm64.

Por quê: o par de imagens de build e de runtime testado na CI só vale se a tag aponta sempre para o
mesmo digest.

### VIII. Regras do SDK reusadas, nunca reimplementadas

- Regras de ID, leitura e validação do `n3x-plugin.yaml`, faixa `sdk`, namespace, `schemagen` e o
  pacote `artifact` (leitura do manifesto, hash e regra de compatibilidade) vêm do `n3x-sdk`. O
  builder chama essas funções e NUNCA as copia nem reescreve. Não existe expressão regular de ID
  fora do SDK.
- Quando falta uma regra no SDK, a mudança é pedida no SDK. Até ela sair, o builder usa um fake nos
  testes, nunca uma cópia da regra.
- Dos outros repositórios do N3X, o builder só depende de `n3x-sdk`, de `core/entry` do
  `n3x-sidecar`, de `host` do `n3x-controller` e de `n3x-plugin-core`.

Por quê: duas implementações da mesma regra divergem, e o build passaria com um plugin que o
runtime recusa.

### IX. Segurança visível

- O `govulncheck` roda em todo build e só avisa (ADR-008): os achados vão para o log, o resumo e o
  `warnings` do manifesto e NUNCA mudam o exit code. Falha da base de vulnerabilidades também não
  falha o build: vira aviso.
- Configuração insegura do operador é aceita e sempre exposta: `GOSUMDB=off` gera `sumdb_off` e
  `GOINSECURE` gera `insecure_fetch`, no log e no resumo.
- Checksums de módulo são conferidos contra `go.sum` e `sum.golang.org`, salvo em `GOPRIVATE`,
  `GONOSUMDB` ou com `GOSUMDB=off`. Divergência é erro (`N3X-B015`).
- A análise estática não é sandbox. Todo desvio encontrado vira caso no corpus de plugins
  maliciosos, e caso do corpus NUNCA é removido.

Por quê: o builder não decide a política de risco do operador. Ele mostra o risco, e a CI do
operador decide se bloqueia o deploy.

## Restrições técnicas e de arquitetura

### Escopo

- O `n3x-builder` atende o FR-015 do PRD, com as regras de build do FR-012 e do FR-013.
- Atores: operador (time de plataforma ou SRE, que escreve o Dockerfile multi-stage e roda o build
  na própria CI); autor de plugin (na v1, só o responsável pelo projeto); mantenedor do builder e
  bot de CI; consumidores do manifesto (entrypoint do sidecar, host do controller e plugin core).
- O builder só existe durante o `docker build` (BuildKit e `docker buildx`). Não tem serviço, porta
  nem estado. Os artefatos rodam em Kubernetes, ECS e docker compose; o Istio é requisito de
  compatibilidade (alvo `istio-1.31`), não de funcionamento.
- Dono de: catálogo `targets/<alvo>.yaml`; comandos `n3x-plugin`; códigos `N3X-B001` a `N3X-B016` e
  exit codes; formato 1 do manifesto na seção ELF `.n3x_manifest`; hash `alloc-sections`;
  versionamento das imagens de build e regra de compatibilidade com as imagens de runtime.
- Consome sem ser dono: `n3x-plugin.yaml`, regras de ID, `schemagen` e pacote `artifact` do
  `n3x-sdk`; `core/entry` do `n3x-sidecar`; `host` do `n3x-controller`; `n3x-plugin-core`, sempre
  incluído na versão da imagem.
- Fora da v1: assinatura, SBOM, atestados, lockfile, arquivo de lista de plugins, comando de
  remoção, tags móveis, tracing e CLI de build fora do Docker. Spec que inclua um desses itens DEVE
  ser recusada; incluí-lo exige mudança no PRD e emenda desta constituição.

### Stack

- Go 1.26, com patch fixado por digest na imagem `golang:1.26.x-trixie` e `GOTOOLCHAIN=local`.
- `cobra` em todas as CLIs (`n3x-plugin`, `n3xci` e ferramentas de `images/tools`) e `testify` em
  todos os testes.
- Também: biblioteca padrão (`debug/elf`, `go/format`, `go/types`), `golang.org/x/mod`,
  `golang.org/x/tools` (`go/analysis`, `go/packages`), `govulncheck` e `n3x-sdk`.
- `zig cc` com a glibc do alvo nas duas arquiteturas, e `objcopy` (`binutils-multiarch`) para
  gravar a seção ELF.
- Bases `envoyproxy/envoy` 1.40.x e `istio/proxyv2` 1.31.x, fixadas por digest. Alvos `envoy-1.40`
  e `istio-1.31`.
- Imagens no GHCR, amd64 e arm64 num índice OCI:
  `ghcr.io/giustech/n3x/sidecar-builder:<semver>-<alvo>` e
  `ghcr.io/giustech/n3x/controller-builder:<semver>`.
- CI no GitHub Actions, com runners nativos `ubuntu-24.04` e `ubuntu-24.04-arm`.
- Dependência Go fora desta lista DEVE ser justificada na Complexity Tracking do plano.

### Arquitetura

- Hexagonal formal. As regras do pipeline ficam no núcleo, como funções puras. Portas são
  interfaces Go, e adaptadores separados cuidam dos recursos externos: toolchain `go`, `zig cc`,
  `objcopy`, processo do Envoy, rede de módulos, sistema de arquivos e relógio. O núcleo NUNCA
  acessa esses recursos sem passar por uma porta.
- O `build` tem 9 etapas, nesta ordem: (1) workspace e alvo, (2) resolução, (3) compatibilidade,
  (4) análise estática e `govulncheck`, (5) geração, (6) compilação, (7) símbolos e teste de carga,
  (8) manifesto e (9) saída em `/out`.
- Áreas do código: `cmd/n3x-plugin`, `internal/` (catálogo, imagem, workspace, resolução, checagem,
  análise, geração, compilação, símbolos, manifesto e relator), `targets/`, `images/`, `testdata/`,
  `ci/n3xci`, `compat/` e `examples/`. Área nova DEVE ser justificada no plano.
- No `build`, só o adaptador de resolução acessa a rede de módulos, e só no passo 2.
- O relator (`internal/report`) é o único caminho de saída. Fora dele e do `main`, nenhum código
  escreve no stdout ou no stderr.

### Contratos públicos

- Comandos: `n3x-plugin add <módulo>@<versão>` ou `add --archive <zip> [--sha256 <hex>]`, uma linha
  por plugin (ADR-005); `build`, `version` e `targets`. Todos aceitam `--log-format text|json`.
- Exit codes: 0 sucesso; 1 uso, ambiente ou erro interno; 2 resolução; 3 compatibilidade; 4 análise
  estática; 5 compilação e schema; 6 símbolos ou carga; 7 manifesto.
- Saídas: `/out/libn3x.so` ou `/out/n3x-controller`, mais `/out/manifest.json`, idêntico byte a
  byte à seção embutida.
- Compatibilidade no runtime: mesma major, e minor da imagem de runtime igual ou maior que a do
  manifesto. A regra vale também na série 0.x, sem exceção (ADR-001).
- `ARG` aceitos (lista de permissão): `GOPROXY`, `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB`, `GOSUMDB`,
  `GOINSECURE`, `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `SSL_CERT_FILE`, `SSL_CERT_DIR`,
  `N3X_LOG_FORMAT` e `N3X_VULNDB`, mais `TARGETOS` e `TARGETARCH` do BuildKit. Nenhuma outra
  variável chega ao processo `go`.

### Metas não funcionais

Hipóteses das FDDs. Todo plano que afete uma meta DEVE dizer como a mede.

- Build com 5 plugins além do core, em amd64: até 60 s com cache e até 180 s sem cache (medição na
  BLD-07, que fecha a L-29).
- Teste de carga do `.so`: perto de 2 s (p95 de 3 s), só na arquitetura nativa.
- Patch novo do Envoy 1.40.x ou do Istio 1.31.x vira imagem candidata em até 7 dias.
- Tamanho comprimido: `sidecar-builder` até 650 MiB e `controller-builder` até 450 MiB.

## Fluxo de desenvolvimento e portões de qualidade

### Testes

- TDD a partir dos cenários de falha (PRD, BLD-02, BLD-03 e BLD-04): o teste é escrito antes do
  código e falha antes da implementação. Para o `/speckit-tasks`, esta regra é o pedido explícito
  de TDD: toda lista de tarefas DEVE trazer as tarefas de teste antes das de implementação.
- Cada código `N3X-Bxxx` tem pelo menos um teste que confere código, regra, exit code e `/out`
  intocado (ou o workspace intocado, no `add`).
- `testify` em todos os testes, com o relator em modo estrito.
- Golden files para as saídas geradas, corpus de plugins maliciosos (caso NUNCA removido) e
  fuzzing noturno dos leitores de entrada externa.
- Cobertura mínima de linhas: 85% em geral; 90% em `internal/symbols` e no pacote `artifact`.

### CI

- Todo PR roda `go vet`, `gofmt -l` (sem saída), `go test -race ./...`, `gosec`, `govulncheck`,
  `actionlint`, a cobertura mínima, os golden files, o corpus e a compilação dupla com comparação de
  hash. A CI é o portão do merge.
- A matriz noturna cobre alvo × arquitetura em runners nativos, sem QEMU: símbolos, teste de carga e
  reprodutibilidade nas quatro combinações de host e destino, mais o fuzzing.
- Alvo ou patch novo entra como `candidate` e só vira `supported` com a matriz nativa verde e merge
  humano.

### Branches, PRs e releases

- Uma branch `NNN-slug` do Spec Kit por feature e PR obrigatório para a `main`. As mensagens seguem
  Conventional Commits.
- Todo merge é humano, inclusive os PRs do bot.
- O bot pode criar tags `-rc.N`. Tags estáveis (`sidecar-builder/vX.Y.Z` e
  `controller-builder/vX.Y.Z`) só pelo mantenedor, num commit da `main`.
- O `compat/` só recebe acréscimos: pares testados, inclusive os que falharam, NUNCA são apagados.

### Especificação

- Fluxo: PRD → HLD → FDD → Spec Kit. Os 4 grupos de `specs/README.md`, na ordem dele, passam por
  `specify`, `clarify`, `plan`, `tasks` e `implement`.
- Pré-requisitos externos (SDK `v0.1.0`, pacote `artifact`, FDDs do sidecar e valores da fase 0)
  bloqueiam o fechamento do grupo que depende deles, não o início: a implementação pode começar com
  fakes.

## Governança

- Esta constituição prevalece sobre specs, planos, tarefas e práticas do repositório. Quando um
  documento de design diverge dela, vale a constituição até o documento ser corrigido.
- Entre os documentos de design, quando uma FDD diverge do HLD ou de um ADR, vale a FDD, e o HLD ou
  o ADR é atualizado. Exceção decidida: a regra de compatibilidade da série 0.x segue o ADR-001, sem
  exceção para a 0.x, e não a BLD-04.
- Idioma: documentação (esta constituição e os artefatos do Spec Kit em `specs/` inclusive) e
  mensagens da CLI em pt-BR; identificadores, campos, comandos e rótulos em inglês. O `README.md` é
  em inglês.
- Licença Apache 2.0. Mantenedor único: Guilherme Giusti, que aprova toda emenda.
- Emendas: PR para a `main` gerado com `/speckit-constitution`, com o Sync Impact Report atualizado
  e os templates de `.specify/templates/` sincronizados no mesmo PR.
- Versão desta constituição em semver:
  - MAJOR: remove ou redefine um princípio, ou relaxa uma regra DEVE ou NUNCA;
  - MINOR: acrescenta princípio ou seção, ou amplia uma regra de forma material;
  - PATCH: redação, correção ou ajuste de meta marcada como hipótese.
- Conformidade:
  - o `/speckit-plan` confere os portões da Constitution Check antes da pesquisa e de novo depois do
    desenho;
  - o `/speckit-analyze` trata violação desta constituição como CRÍTICA;
  - toda revisão de PR confere os princípios tocados pela mudança;
  - princípios e regras DEVE ou NUNCA não admitem exceção num plano: mudá-los exige emenda. A
    Complexity Tracking do plano só justifica escolhas que esta constituição deixa abertas, como
    dependência fora da stack ou área de código nova.
- Revisão: ao fechar cada grupo de `specs/README.md`, o mantenedor confere se as pendências
  resolvidas (fase 0, V-01 a V-09 da BLD-05 e metas medidas) mudam alguma regra e, se mudam, emenda
  esta constituição.
- Guias de uso: `README.md` (comportamento da v1) e `specs/README.md` (ordem de implementação).

**Versão**: 1.0.0 | **Ratificada em**: 2026-10-23 | **Última emenda**: 2026-10-23
