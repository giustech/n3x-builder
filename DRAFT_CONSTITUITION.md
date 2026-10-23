# DRAFT_CONSTITUITION: n3x-builder

Insumo para gerar a constituição do repositório com o Spec Kit (`/speckit-constitution`). Não é a constituição.
Fontes: PRD 1.5, HLD 1.0, ADR-001 a ADR-009, FDD-BLD-01 a FDD-BLD-07, `specs/README.md` e decisões do mantenedor em 2026-10-23.

## 1. Identidade

- O N3X (Network Navigation Node eXtensible) é um middleware de rede extensível: uma camada de override sobre o Envoy em modelo sidecar. Cada aplicação ganha um sidecar como única porta de entrada, e os sidecars se conectam a um controller central que só observa.
- Comportamentos de rede viram plugins Go com até duas metades: a de proxy, compilada num único dynamic module (`libn3x.so`) carregado no Envoy, e a de controller, compilada no binário do controller.
- O `n3x-builder` transforma a lista de plugins escrita no Dockerfile do operador nesses dois artefatos. Ele entrega as imagens de build OCI com toolchain travado, a CLI `n3x-plugin` (`add`, `build`, `version`, `targets`), escrita em Go, e o catálogo de alvos.
- Problema que resolve: barrar no build tudo o que só quebraria em produção (plugin incompatível com o SDK, ID duplicado, código que fura o isolamento, símbolo ausente no Envoy do alvo, `.so` que não carrega) e fazer o mesmo Dockerfile gerar o mesmo hash em qualquer máquina.
- Requisitos do PRD atendidos: FR-015, com as regras de build do FR-012 e do FR-013.

## 2. Quem usa e onde

- Operador (time de plataforma/SRE, persona principal do PRD): escreve o Dockerfile multi-stage, roda o build na própria CI e combina as tags das imagens de build e de runtime.
- Autor de plugin: corrige o que o build recusa. Na v1, só o responsável pelo projeto escreve plugins.
- Mantenedor do builder e bot de CI: mantêm o catálogo, a matriz de CI e as publicações.
- Consumidores do manifesto embutido: entrypoint do sidecar, host do controller e plugin core (inventário e alerta de divergência, FR-006).
- O builder só existe durante o `docker build` (BuildKit e `docker buildx`), em qualquer máquina ou CI. Não tem serviço, porta nem estado.
- Os artefatos rodam onde houver containers: Kubernetes, ECS e docker compose. O Istio é requisito de compatibilidade (alvo `istio-1.31`), não de funcionamento.

## 3. Escopo e limites

- Dono de: catálogo `targets/<alvo>.yaml`, comandos `n3x-plugin`, códigos `N3X-B001` a `N3X-B016` e exit codes, formato 1 do manifesto na seção ELF `.n3x_manifest`, hash `alloc-sections`, versionamento das imagens de build e regra de compatibilidade com as imagens de runtime.
- Consome, sem ser dono: `n3x-plugin.yaml`, regras de ID, `schemagen` e pacote `artifact` do `n3x-sdk`; `core/entry` do `n3x-sidecar`; `host` do `n3x-controller`; `n3x-plugin-core`, sempre incluído na versão da imagem.
- O builder nunca reimplementa uma regra do SDK e nunca depende de módulo do sidecar ou do controller além de `core/entry` e `host`.
- Fora da v1: assinatura, SBOM, atestados, lockfile, arquivo de lista de plugins, comando de remoção, tags móveis (`latest`), tracing e CLI de build fora do Docker.

## 4. Stack técnica

- Go 1.26, com patch fixado por digest na imagem `golang:1.26.x-trixie` e `GOTOOLCHAIN=local`.
- `cobra` para as CLIs (`n3x-plugin`, `n3xci` e ferramentas de `images/tools`) e `testify` em todos os testes.
- Também usa: biblioteca padrão (`debug/elf`, `go/format`, `go/types`), `golang.org/x/mod`, `golang.org/x/tools` (`go/analysis`, `go/packages`), `govulncheck` e `n3x-sdk`.
- `zig cc` com a glibc do alvo nas duas arquiteturas e `objcopy` (`binutils-multiarch`) para gravar a seção ELF.
- Imagens base `envoyproxy/envoy` 1.40.x e `istio/proxyv2` 1.31.x, fixadas por digest.
- Imagens no GHCR, amd64 e arm64 num índice OCI: `sidecar-builder:<semver>-<alvo>` e `controller-builder:<semver>`.
- CI no GitHub Actions, com runners nativos `ubuntu-24.04` e `ubuntu-24.04-arm` e ações fixadas por SHA.

## 5. Arquitetura interna

- Arquitetura hexagonal formal. As regras do pipeline ficam no núcleo, como funções puras. Portas são interfaces Go, e adaptadores separados cuidam dos recursos externos: toolchain `go`, `zig cc`, `objcopy`, processo do Envoy, rede de módulos, sistema de arquivos e relógio.
- Pipeline de 9 etapas que falha cedo: workspace e alvo, resolução, compatibilidade, análise estática e `govulncheck`, geração, compilação, símbolos e teste de carga, manifesto e saída em `/out`.
- Áreas do código: `cmd/n3x-plugin`, `internal/` (catálogo, imagem, workspace, resolução, checagem, análise, geração, compilação, símbolos, manifesto e relator), `targets/`, `images/`, `testdata/`, `ci/n3xci`, `compat/` e `examples/`.
- Só o adaptador de resolução usa a rede de módulos, e só no passo 2. O relator é o único caminho de saída: nenhum outro código escreve no stdout ou no stderr.
- O `n3x-plugin` nunca importa `secret/` do SDK.

## 6. Princípios candidatos

1. Falhar no build, nunca em produção.
2. Reprodutibilidade: mesma entrada e mesma imagem geram as mesmas seções `SHF_ALLOC` em qualquer host (ADR-007).
3. Determinismo, sem fallback silencioso: nunca troca versão, nunca usa `latest` ou `replace`, nunca pula etapa, nunca entrega artefato parcial.
4. Zero credencial absoluto: nenhuma credencial chega ao builder por arquivo, variável, argumento ou montagem. Plugin privado entra só por ZIP de módulo Go (`add --archive`).
5. Contratos só por adição: códigos `N3X-Bxxx`, nomes de regra, campo `format`, eventos JSON e rótulos `io.n3x.*`. Um código pode ampliar o cenário dentro da mesma família, sem mudar o exit code (ADR-003, ADR-006).
6. Erros acionáveis: código, regra, "como resolver" e link para `docs/erros.md`, gerado do catálogo. A etapa lista todas as falhas dela, até 100.
7. Imagens imutáveis: tags nunca sobrescritas nem apagadas, bases fixadas por digest, semver próprio por imagem (ADR-001, ADR-009).
8. Reuso, nunca reimplementação, das regras do SDK.
9. Segurança visível: `govulncheck` em todo build só avisa (ADR-008). `GOSUMDB=off` e `GOINSECURE` do operador são aceitos com aviso no log e no resumo.

## 7. Contratos públicos

- `n3x-plugin add <módulo>@<versão>` ou `add --archive <zip> [--sha256 <hex>]`, uma linha por plugin (ADR-005); `build`, `version` e `targets`, todos com `--log-format text|json`.
- Exit codes: 0 sucesso, 1 uso, ambiente ou erro interno, 2 resolução, 3 compatibilidade, 4 análise estática, 5 compilação e schema, 6 símbolos ou carga, 7 manifesto.
- Saídas: `/out/libn3x.so` ou `/out/n3x-controller` e `/out/manifest.json`, idêntico à seção embutida.
- Compatibilidade no runtime: mesma major e minor da imagem de runtime igual ou maior que a do manifesto, sem exceção na série 0.x (ADR-001).
- `ARG` aceitos (lista de permissão): `GOPROXY`, `GOPRIVATE`, `GONOPROXY`, `GONOSUMDB`, `GOSUMDB`, `GOINSECURE`, `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`, `SSL_CERT_FILE`, `SSL_CERT_DIR`, `N3X_LOG_FORMAT` e `N3X_VULNDB`, mais `TARGETOS` e `TARGETARCH` do BuildKit.

## 8. Requisitos não funcionais (hipóteses das FDDs)

- Build com 5 plugins além do core, amd64: até 60 s com cache e 180 s sem cache (BLD-07, que fecha a L-29).
- Teste de carga do `.so` perto de 2 s (p95 de 3 s), só na arquitetura nativa.
- Patch novo do Envoy 1.40.x ou do Istio 1.31.x vira imagem candidata em até 7 dias.
- Tamanho: `sidecar-builder` até 650 MiB e `controller-builder` até 450 MiB, comprimidos.
- Credenciais em URL e caminhos absolutos do cache nunca aparecem em log.

## 9. Qualidade, testes e CI

- TDD a partir dos cenários de falha (PRD, BLD-02, BLD-03 e BLD-04): um teste por código `N3X-Bxxx`, conferindo código, regra, exit e `/out` intocado.
- `go vet`, `gofmt -l`, `go test -race ./...`, `gosec`, `govulncheck` e `actionlint`.
- Cobertura mínima de linhas: 85% em geral e 90% em `internal/symbols` e no pacote `artifact`.
- Golden files, corpus de plugins maliciosos (caso nunca removido), fuzzing noturno e relator em modo estrito nos testes.
- CI em todo PR com compilação dupla e comparação de hash. Matriz noturna alvo × arquitetura em runners nativos, sem QEMU.
- Alvo ou patch novo entra como `candidate` e só vira `supported` com a matriz nativa verde e merge humano.

## 10. Governança e fluxo

- Licença Apache 2.0. Mantenedor único: Guilherme Giusti.
- Fluxo: PRD → HLD → FDD → Spec Kit. Ordem de implementação em `specs/README.md` (4 grupos).
- Precedência: quando uma FDD diverge do HLD ou de um ADR, vale a FDD, e o HLD ou o ADR é atualizado. Exceção decidida: a regra de compatibilidade da série 0.x segue o ADR-001.
- Pré-requisitos externos (SDK `v0.1.0`, pacote `artifact`, FDDs do sidecar e valores da fase 0) bloqueiam o fechamento do grupo que depende deles. A implementação pode começar com fakes.
- Idioma: documentação e mensagens em pt-BR; identificadores, campos, comandos e rótulos em inglês.
- Uma branch `NNN-slug` do Spec Kit por feature e PR obrigatório para a `main`, com a CI como portão. Mensagens seguem Conventional Commits.
- Todo merge é humano, inclusive os PRs do bot. O bot pode criar tags `-rc.N`; tags estáveis (`sidecar-builder/vX.Y.Z`, `controller-builder/vX.Y.Z`) só pelo mantenedor, num commit da `main`.

## 11. Documentos a atualizar antes do Spec Kit

- `HLD.md`:
  - exit 1 ampliado;
  - `zig cc` nas duas arquiteturas;
  - as duas metades dos plugins compiladas nos dois artefatos;
  - Dockerfile com `FROM --platform=$BUILDPLATFORM`;
  - índice de símbolos;
  - `add` baixando o ZIP inteiro e com os códigos que emite;
  - `host.Run` com módulo;
  - `schemas` por metade e `vuln_scan` em `warnings`;
  - `N3X_VULNDB`, `/n3x/build`, `format` no catálogo e `unsafe` proibido.
- `HLD.json`: remover "mount de secret".
- ADR-004: índice JSON pré-gerado no lugar do binário do Envoy.
- ADR-006: exit 1 ampliado, ampliação dentro da família e limite de 100 falhas por etapa.
- BLD-04 e BLD-07: remover a regra `runtime_unstable` da série 0.x.
- BLD-02: remover a hipótese do `.netrc` e alinhar a mensagem do B001 `not_in_sumdb` ao `--archive`.
- BLD-05: remover os resíduos de "secret" e "token".
- BLD-07: `n3xci` com `cobra` e `testify`, governança do bot conforme a seção 10 e cabeçalhos da tabela de pares em inglês, porque o README é em inglês.
- BLD-03: alinhar as metas por passo sem cache aos 180 s da BLD-07.
- BLD-06: citar ADRs com o nome do repositório.
- Todas as FDDs: portas como interfaces (hexagonal), `cobra` e `testify`.
- `specs/README.md`: declarar os pré-requisitos externos de cada grupo.

## 12. Pendências conhecidas

- Fase 0: digest, commit e glibc do Envoy e pseudo-versão do SDK Go do Envoy no catálogo; spikes `envoy-1.40` e `istio-1.31`.
- V-01 a V-09 (BLD-05): símbolos exportados, mensagens de ABI, sinais de subida, glibc e tempos reais do Envoy, a confirmar no spike.
- Destino do aviso `archive_unverified` no `warnings` do manifesto: a BLD-01 cria o aviso e a BLD-04 não o trata.
- Nome e versão do pacote `artifact` do SDK (hipótese: `v0.2.0`).
