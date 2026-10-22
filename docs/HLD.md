### HLD: n3x-builder (imagens de build, `n3x-plugin` e catálogo de alvos)

Versão: 1.0
Data: 2026-10-20
Responsável: Guilherme Giusti

---

### Objetivo técnico
Transformar a lista de plugins escrita no Dockerfile do operador em dois artefatos reproduzíveis, gerados dentro do Docker: o `libn3x.so` do sidecar (um por alvo e arquitetura) e o binário do controller, ambos com o manifesto embutido. O builder barra no build tudo o que só quebraria em produção: plugin incompatível com o SDK, ID duplicado, análise estática reprovada, símbolo ausente no Envoy do alvo e `.so` que não carrega. Atende o FR-015 e implementa as regras de build definidas no FR-012 e no FR-013.

Este HLD é dono destes contratos (conforme `docs/hld/HLDs-N3X.md`):
- Catálogo de alvos (`targets/<alvo>.yaml`)
- Comandos `n3x-plugin` (`add`, `build`, `version` e `targets`), códigos de erro e exit codes
- Formato do manifesto embutido no `.so` e no binário do controller, incluindo o hash de integridade
- Versionamento das imagens de build e regra de compatibilidade com as imagens de runtime

Dependências com outros sistemas
- `n3x-sdk`: manifesto `n3x-plugin.yaml`, estrutura do repositório de plugin, regras da análise estática, geração do JSON Schema e padrão de IDs (só referenciados aqui)
- `n3x-sidecar`: módulo `core/` importado pelo `main` gerado do `.so` e imagem de runtime `n3x/sidecar`, que confere alvo, integridade e versão do manifesto na subida
- `n3x-controller`: módulo `host/` chamado pelo `main` gerado (`host.Run(core.Plugin(), <plugins>...)`) e imagem de runtime `n3x/controller`
- `n3x-plugin-core`: incluído sempre, na versão fixada pela imagem de build
- Go module proxy e `sum.golang.org`, GitHub (plugins privados entram por ZIP de módulo Go copiado no build, sem credencial) e GHCR

---

### Arquitetura geral
O builder **só existe durante o `docker build`**: não tem serviço, porta nem estado. Ele é um conjunto de imagens OCI com toolchain fixado e a ferramenta `n3x-plugin`, escrita em Go, que executa um pipeline de build em etapas que falha cedo.

```
 Dockerfile do operador
 ┌──────────────────────────────────────────────────────────────────┐
 │ FROM ghcr.io/giustech/n3x/sidecar-builder:1.0.0-envoy-1.40 AS b  │
 │ RUN n3x-plugin add github.com/giustech/n3x-plugin-ratelimit@v1.0.0│
 │ RUN --mount=type=cache,...                                       │
 │     n3x-plugin build                                             │
 │ FROM ghcr.io/giustech/n3x/sidecar:1.0.0-envoy-1.40               │
 │ COPY --from=b /out/libn3x.so /usr/lib/n3x/libn3x.so              │
 └───────────────┬──────────────────────────────────────────────────┘
                 ▼
 sidecar-builder (alvo × arquitetura)
 ┌──────────────────────────────────────────────────────────────────┐
 │ Go 1.26 · toolchain C (glibc <= a do Envoy) · zig cc · Envoy alvo│
 │ /etc/n3x/target.yaml · /n3x/work (workspace) · n3x-plugin        │
 │                                                                  │
 │ add → resolve → check → analysis → generate → compile            │
 │     → symbols + carga → manifest → /out                          │
 └──────────────────────────────────────────────────────────────────┘
```

**Imagens** (ADR-001 e ADR-009)
| Imagem | Tag | Conteúdo |
| --- | --- | --- |
| `ghcr.io/giustech/n3x/sidecar-builder` | `<semver>-envoy-1.40`, `<semver>-istio-1.31` | Go 1.26, toolchain C com glibc igual ou anterior à da imagem do Envoy do alvo, `zig cc` para arm64, binário do Envoy do alvo (para símbolos e teste de carga), `/etc/n3x/target.yaml`, `n3x-plugin`, cache dos módulos `core/`, SDK e plugin core já baixados |
| `ghcr.io/giustech/n3x/controller-builder` | `<semver>` | Go 1.26, `n3x-plugin`, módulos `host/`, SDK e plugin core já baixados; sem toolchain C, porque o binário é estático e sem cgo |

- amd64 e arm64 em manifest list OCI, publicadas com `docker buildx` pela CI do repositório
- Imagens base (`golang`, `envoyproxy/envoy`, `istio/proxyv2`) fixadas por digest
- O controller não depende do alvo do Envoy, por isso o `controller-builder` não tem sufixo de alvo

**Versionamento e compatibilidade** (ADR-001)
- Cada imagem tem semver próprio: `sidecar-builder`, `controller-builder`, `sidecar` e `controller` evoluem de forma independente
- O manifesto embutido grava o módulo e a versão do runtime compilado: `core/` no `.so`, `host/` no controller
- Na subida, o `n3x run` e o controller aceitam a **mesma major** com **minor da imagem de runtime igual ou maior** que a do manifesto; fora disso, recusam e dizem qual versão usar:
  ```
  erro: libn3x.so feito com core 1.3.0; esta imagem (sidecar 1.1.2) aceita core >=1.0 <=1.1
  use n3x/sidecar >= 1.3.0-envoy-1.40
  ```
- O README do repositório publica a tabela dos pares testados na CI

**Catálogo de alvos** (`targets/<alvo>.yaml`)
```yaml
id: envoy-1.40
envoy:
  image: envoyproxy/envoy:v1.40.2@sha256:...
  commit: 7c1f0e...                       # commit exato do Envoy
  glibc: "2.31"                           # hipótese; limite do toolchain C
envoy_go_sdk: github.com/envoyproxy/envoy/source/extensions/dynamic_modules/sdk/go v0.0.0-2026...-7c1f0e
runtime_image: ghcr.io/giustech/n3x/sidecar   # imagem do estágio final
capabilities: [request_body, route_change]    # o que o adaptador do alvo oferece ao SDK
archs: [amd64, arm64]
status: supported                             # candidate | supported | deprecated
```
- O alvo `istio-1.31` traz também `istio.version` e o commit do Envoy derivado do `istio.deps` daquele patch
- Um alvo novo ou um patch novo entra como `candidate`: a CI roda comparação de símbolos e teste de carga na imagem exata, em amd64 e arm64 nativos, e só então passa para `supported`
- A pseudo-versão do SDK Go do Envoy é confirmada com `go list -m` antes de entrar no catálogo
- Cada imagem `sidecar-builder` embute só o arquivo do seu alvo em `/etc/n3x/target.yaml`

Ambiente de implantação
- Não há implantação em runtime: as imagens são publicadas no GHCR e usadas no `docker build` do operador, em qualquer máquina ou CI com Docker e BuildKit
- A CI do `n3x-builder` (GitHub Actions) publica as imagens por tag semver, com matriz alvo × arquitetura em runners nativos

Tecnologias principais
- Go 1.26 (`go/analysis`, `debug/elf`, `go/format`) para a ferramenta `n3x-plugin`
- Toolchain C com glibc fixada e `zig cc` para compilar arm64 a partir de amd64
- `objcopy` para gravar a seção ELF do manifesto
- `govulncheck` em todo build
- Docker BuildKit (`--mount=type=cache` e `COPY` de arquivos locais) e `docker buildx`
- GHCR como registro das imagens

Padrões adotados
- Pipeline de build em etapas que falha cedo e lista todas as falhas da etapa
- Geração de código (`main.go`, registro dos plugins e JSON Schema), com saída determinística
- Imagens imutáveis, fixadas por digest, e build reproduzível (ADR-007)
- Toolchain travado na imagem: o mesmo Dockerfile dá o mesmo resultado em qualquer máquina

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| Catálogo de alvos (`targets/`) | Fixar commit do Envoy, pseudo-versão do SDK Go do Envoy, digest das imagens, glibc, capacidades e status de cada alvo | CI do builder |
| Workspace (`/n3x/work`) | Guardar `go.mod`, `go.sum` e a lista de plugins entre um `RUN` e outro | Sistema de arquivos da camada |
| `add` | Resolver `<módulo>@<versão>`, ler o `n3x-plugin.yaml`, conferir estrutura (manifesto, pastas das metades declaradas) e registrar no workspace; aceita o core só na versão da imagem | Go module proxy, GitHub, regras de estrutura do SDK |
| Resolvedor | `go mod download` com o toolchain fixado, conferindo o `go.sum` no `sum.golang.org` (exceto `GOPRIVATE`) | Go module proxy, ZIPs privados e fonte local de módulos (`/n3x/private-proxy/`) |
| Checador de compatibilidade | Conferir faixa `sdk`, IDs e serviços gRPC duplicados, `depends_on` (faltando ou em ciclo), namespace `n3x` restrito a `github.com/giustech/` e presença do core | Manifestos dos plugins, versão do SDK da imagem |
| Analisador estático | Aplicar as regras do HLD do `n3x-sdk` com `go/analysis` sobre os pacotes de cada plugin e rodar o `govulncheck` (só aviso) | Regras do SDK, base de vulnerabilidades do Go |
| Gerador | Criar `main.go` (sidecar: importa `core/` e chama `agent.Plugin()`; controller: chama `host.Run(core.Plugin(), ...)`), o registro dos plugins e o JSON Schema de cada config, em ordem fixa | SDK (gerador de schema), `core/`, `host/` |
| Compilador | `go build -buildmode=c-shared -trimpath -ldflags=-buildid=` no sidecar (com `zig cc` para arm64); binário estático com `CGO_ENABLED=0` no controller | Toolchain Go e C |
| Verificador de símbolos e carga | Comparar símbolos importados e exportados do `.so` com os do Envoy do alvo; na arquitetura nativa, subir o Envoy com bootstrap mínimo que carrega o `.so` (cerca de 2 s) | Binário do Envoy na imagem |
| Gerador do manifesto | Montar o JSON, calcular o hash, gravar a seção `.n3x_manifest` e escrever `/out/manifest.json` | `objcopy`, `debug/elf` |
| Relator | Log por etapa (texto ou JSON), resumo final e mensagens `N3X-Bxxx` | Todos os anteriores |

- Quem orquestra é o `n3x-plugin build`, que roda as etapas em sequência
- Nada é persistido fora do workspace e de `/out`; o cache de módulos fica no cache do BuildKit e não entra na imagem
- Plugin só de proxy: o `controller-builder` não compila nada dele, mas o mantém no manifesto, para o painel mostrar no inventário
- Plugin só de controller: o `sidecar-builder` faz o mesmo

**Estrutura do repositório**
```
n3x-builder/
├── cmd/n3x-plugin/     add, build, version, targets
├── internal/           resolve, check, analysis, generate, compile, symbols, manifest, report
├── targets/            envoy-1.40.yaml, istio-1.31.yaml
├── images/             Dockerfiles das duas imagens de build
├── testdata/           plugins de teste, inclusive o corpus "malicioso" da análise estática
└── docs/               HLD, ADRs, erros.md e cópia do PRD
```

---

### Fluxo de requisições e de dados
**Fluxo de requisição** (um `docker build`)
- `RUN n3x-plugin add <módulo>@<versão>`, uma linha por plugin (ADR-005):
  - resolve o módulo e baixa só o `n3x-plugin.yaml` e a árvore de pastas
  - confere a estrutura e registra o plugin no workspace
  - falha com `N3X-B001`, `N3X-B002`, `N3X-B003` ou `N3X-B008`
- `RUN n3x-plugin build`, etapa por etapa:
  1. lê o workspace e o `/etc/n3x/target.yaml` e inclui o plugin core na versão da imagem
  2. `go mod download` com o toolchain fixado, usando a fonte local de módulos privados (`/n3x/private-proxy/`) quando houver repositório privado
  3. confere a faixa `sdk`, os IDs e serviços duplicados, o `depends_on` e o namespace `n3x`
  4. roda a análise estática e o `govulncheck`
  5. gera o `main.go`, o registro dos plugins e o JSON Schema de cada config
  6. compila o `libn3x.so` (no controller, o binário estático)
  7. verifica os símbolos contra o Envoy do alvo e, na arquitetura nativa, faz o teste de carga (só no sidecar; ADR-004)
  8. monta o manifesto, calcula o hash e grava a seção `.n3x_manifest`
  9. escreve `/out/libn3x.so` (ou `/out/n3x-controller`) e `/out/manifest.json`
- Estágio final: parte de `ghcr.io/giustech/n3x/sidecar:<semver>-<alvo>` (ou `.../controller:<semver>`) e copia o artefato
- Na subida, o runtime confere alvo, hash, `format` e versão do `core/` ou do `host/` (HLDs do `n3x-sidecar` e do `n3x-controller`)

**Fluxo de dados**
- Módulos do plugin (Go proxy ou GitHub) → cache de módulos do BuildKit → código gerado → `libn3x.so` ou binário → manifesto embutido → imagem final no registro do operador
- Struct de `types/` → JSON Schema → manifesto → validação do `n3x.yaml` no sidecar e no controller → mascaramento de segredos pelo plugin core
- `n3x-plugin.yaml` de cada plugin → manifesto embutido → registro no plugin core → inventário e alerta de divergência de manifesto no painel (FR-006)
- ZIPs de módulos privados e fonte local → copiados com `COPY` para o estágio de build e usados no `add --archive` e no passo 2; são código confidencial, não credencial, e não entram na imagem final nem no manifesto
- Avisos do `govulncheck` → campo `warnings` do manifesto e `/out/manifest.json`

**Exemplo de Dockerfile do sidecar**
```dockerfile
# syntax=docker/dockerfile:1
FROM ghcr.io/giustech/n3x/sidecar-builder:1.0.0-envoy-1.40 AS build
RUN n3x-plugin add github.com/giustech/n3x-plugin-ratelimit@v1.0.0
RUN n3x-plugin add github.com/acme/n3x-plugin-tenant@v0.4.2
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    n3x-plugin build

FROM ghcr.io/giustech/n3x/sidecar:1.0.0-envoy-1.40
COPY --from=build /out/libn3x.so /usr/lib/n3x/libn3x.so
```
```
docker buildx build --platform linux/amd64,linux/arm64 \
  -t registry.acme/n3x-sidecar:2026.10 --push .
```

**Plugin privado** (sem credencial no builder): o operador copia o ZIP no formato de módulo Go e as dependências privadas transitivas para o estágio de build.
```dockerfile
COPY dist/n3x-plugin-tenant-v0.4.2.zip /n3x/private/plugin.zip
COPY private-proxy/ /n3x/private-proxy/
RUN --mount=type=cache,target=/go/pkg/mod \
    n3x-plugin add --archive /n3x/private/plugin.zip --sha256 <hex>
```

**Exemplo de Dockerfile do controller**
```dockerfile
# syntax=docker/dockerfile:1
FROM ghcr.io/giustech/n3x/controller-builder:1.0.0 AS build
RUN n3x-plugin add github.com/giustech/n3x-plugin-ratelimit@v1.0.0
RUN n3x-plugin add github.com/acme/n3x-plugin-tenant@v0.4.2
RUN --mount=type=cache,target=/go/pkg/mod \
    n3x-plugin build

FROM ghcr.io/giustech/n3x/controller:1.0.0
COPY --from=build /out/n3x-controller /usr/local/bin/n3x-controller
```

---

### Modelo de dados (alto nível)
Entidades principais
- **Alvo:** `id`, imagem e commit do Envoy, glibc, pseudo-versão do SDK Go do Envoy, capacidades, arquiteturas e status
- **Workspace:** `go.mod`, `go.sum` e lista ordenada de plugins (`módulo@versão`)
- **Manifesto embutido** (formato 1, ADR-003), em JSON na seção ELF `.n3x_manifest`, gravada com `objcopy --add-section`:
  ```json
  {
    "format": 1,
    "kind": "sidecar",
    "target": "envoy-1.40",
    "arch": "arm64",
    "builder": "1.0.0",
    "runtime": { "module": "github.com/giustech/n3x-sidecar/core", "version": "1.0.0" },
    "sdk": "1.2.0",
    "go": "1.26.1",
    "built_at": "2026-10-20T14:00:00Z",
    "plugins": [
      {
        "id": "n3x.core",
        "module": "github.com/giustech/n3x-plugin-core",
        "version": "1.0.0",
        "halves": ["proxy", "controller"],
        "manifest": { "...": "n3x-plugin.yaml inteiro" },
        "schema": { "...": "JSON Schema da config" }
      }
    ],
    "warnings": [ { "kind": "vuln", "id": "GO-2026-1234", "module": "github.com/redis/go-redis/v9", "called": true } ],
    "integrity": { "alg": "sha256", "scope": "alloc-sections", "value": "9f2c..." }
  }
  ```
  - No controller, `kind` é `controller`, `target` é omitido e `runtime.module` é `github.com/giustech/n3x-controller/host`
  - `plugins` traz todos os plugins da lista, inclusive os sem a metade daquele artefato, em ordem de ID
- **Hash de integridade** (ADR-002): SHA-256 do conteúdo das seções ELF com a flag `SHF_ALLOC`, concatenadas na ordem do arquivo. A `.n3x_manifest` não tem `SHF_ALLOC`, então fica fora do cálculo; o runtime refaz a conta com `debug/elf`, sem alterar o arquivo. O `built_at` também fica fora do hash
- **Erro de build:** código `N3X-Bxxx`, categoria (exit code), mensagem, como resolver e link

Relações
- Manifesto 1:1 com `.so` ou binário
- Manifesto 1:N com plugins; plugin 1:1 com `n3x-plugin.yaml` e 1:1 com schema
- Alvo 1:N com imagens `sidecar-builder` (uma por arquitetura e versão)
- Imagem de build 1:1 com versão de `core/` ou `host/`, de SDK e de plugin core

Fonte de verdade
- Catálogo de alvos: `targets/` no repositório `n3x-builder`
- Lista de plugins: o Dockerfile do operador
- Depois do build: o manifesto embutido, lido pelo sidecar, pelo controller e pelo plugin core

Versionamento e retenção
- O campo `format` cresce só por adição de campos: o runtime ignora campo desconhecido e recusa `format` maior do que conhece
- O builder não guarda nada; não há retenção

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| `n3x-plugin add <módulo>@<versão>` ou `n3x-plugin add --archive <zip> [--sha256 <hex>]` | SDK | CLI | Externa | Versão exata obrigatória; uma linha por plugin; `--archive` para plugin privado, com identidade pelo prefixo do ZIP e `--sha256` opcional |
| `n3x-plugin build [--log-format text\|json]` | SDK | CLI | Externa | Menos de 60 s com cache e menos de 3 min sem cache, até 5 plugins, amd64 nativo (hipótese) |
| `n3x-plugin version` e `n3x-plugin targets` | SDK | CLI | Externa | Mostram versões embutidas e o alvo da imagem |
| `ghcr.io/giustech/n3x/sidecar-builder:<semver>-<alvo>` | SDK | Imagem OCI | Externa | amd64 e arm64; tags imutáveis |
| `ghcr.io/giustech/n3x/controller-builder:<semver>` | SDK | Imagem OCI | Externa | amd64 e arm64; tags imutáveis |
| `COPY` de ZIP e fonte privada, `ARG GOPROXY`, `ARG GOPRIVATE`, `TARGETARCH` | SDK | Docker BuildKit | Externa | Sem credencial no builder; o ZIP e a fonte local são código confidencial |
| `/out/libn3x.so`, `/out/n3x-controller`, `/out/manifest.json` | SDK | Arquivos | Externa | `manifest.json` é cópia legível do embutido |
| Seção `.n3x_manifest` | SDK | ELF + JSON (formato 1) | Interna (sidecar, controller e plugin core) | Só cresce por adição |
| `targets/<alvo>.yaml` | SDK | YAML | Interna | Alvo só vira `supported` após CI nativa |

**Erros do build** (ADR-006): uma mensagem por falha no stderr, com código, como resolver e link para `docs/erros.md`. O build termina a etapa em que falhou e lista todas as falhas dela.
```
erro N3X-B004: plugin giustech.ratelimit@1.0.0 exige sdk >=1.3.0 <2.0.0; esta imagem tem sdk 1.2.0
  como resolver: use n3x/sidecar-builder >= 1.4.0-envoy-1.40 ou uma versão anterior do plugin
  doc: https://github.com/giustech/n3x-builder/blob/main/docs/erros.md#n3x-b004
```

| Código | Cenário | Exit code |
| --- | --- | --- |
| N3X-B001 | Módulo não encontrado ou sem acesso (plugin privado: use `add --archive`) | 2 |
| N3X-B002 | Repositório fora da estrutura: manifesto ausente ou metade declarada sem pasta | 2 |
| N3X-B003 | Manifesto `n3x-plugin.yaml` inválido | 2 |
| N3X-B004 | Faixa `sdk` incompatível com a da imagem (mostra a exigida e a disponível) | 3 |
| N3X-B005 | ID duplicado ou nome de serviço gRPC duplicado | 3 |
| N3X-B006 | `depends_on` faltando na imagem ou em ciclo | 3 |
| N3X-B007 | Namespace `n3x` fora de `github.com/giustech/` | 3 |
| N3X-B008 | Tentativa de remover o plugin core, ou core em versão diferente da imagem | 3 |
| N3X-B009 | Análise estática reprovada (lista as regras violadas, com arquivo e linha) | 4 |
| N3X-B010 | Falha de compilação | 5 |
| N3X-B011 | Símbolo ausente no Envoy do alvo | 6 |
| N3X-B012 | Falha no teste de carga do `.so` | 6 |
| N3X-B013 | Falha ao gravar o manifesto | 7 |
| N3X-B014 | Config do plugin sem schema válido (gerador de schema falhou) | 5 |
| N3X-B015 | Checksum divergente: conteúdo baixado ou ZIP não confere com o `go.sum`, o `sum.golang.org`, o hash gravado pelo `add` ou o `--sha256` | 2 |
| N3X-B016 | Plugin ou dependência exige Go mais novo que o da imagem | 3 |

Exit code 1 é reservado a erro de uso (comando ou flag inválidos).

---

### Considerações de escalabilidade e disponibilidade
Abordagem geral
- O builder não tem tráfego: escalar é ter builds rápidos e previsíveis; disponibilidade é não depender de nada instável em runtime
- Falha do builder nunca afeta produção: a imagem já publicada continua valendo

Técnicas aplicadas
- Cache de módulos e de build do Go via `--mount=type=cache`; módulos de `core/`, `host/`, SDK e plugin core já baixados na imagem
- arm64 compilado em amd64 com `zig cc`, sem QEMU; o teste de carga em arm64 fica na CI nativa do builder
- `GOPROXY` e `GOPRIVATE` configuráveis por `ARG`, para proxy corporativo
- Go proxy ou GitHub fora do ar: o build falha com `N3X-B001`, sem efeito em runtime
- Build reproduzível (ADR-007): `-trimpath`, `-ldflags=-buildid=`, código gerado em ordem fixa e `built_at` fora do hash; a CI compila o mesmo conjunto duas vezes e compara os hashes
- Matriz da CI alvo × arquitetura em runners nativos; cada alvo novo ou patch do Istio adiciona uma linha

Meta de disponibilidade
- Toda tag publicada continua baixável no GHCR; tags são imutáveis e nunca apagadas
- Patch novo do Envoy 1.40.x ou do Istio 1.31.x vira imagem `candidate` em até 7 dias (hipótese)
- Builds: menos de 60 s com cache e menos de 3 min sem cache, até 5 plugins, amd64 nativo (hipótese)

---

### Segurança
Autenticação
- O builder não recebe credenciais: plugin privado entra por ZIP de módulo Go (`add --archive`, com `--sha256` opcional) e dependências privadas por fonte local copiada pelo operador; o ZIP e a fonte são código confidencial e o operador protege contexto, camadas e cache do estágio de build

Autorização
- Namespace `n3x` exclusivo de módulos `github.com/giustech/` (`N3X-B007`)
- Plugin core obrigatório e na versão da imagem (`N3X-B008`)
- Análise estática com as regras do HLD do `n3x-sdk`: `go` cru, `os.Exit`, `log.Fatal`, `init()` com efeito colateral, `unsafe` e `reflect` sobre tipos do SDK, cgo, import do SDK Go do Envoy, de `secret/` e de pacotes internos do sidecar ou do controller, `context.Background()` e `context.TODO()`, chamadas bloqueantes sem `ctx` e `.Reveal()` em log; o catálogo completo das 18 regras do `N3X-B009` está na FDD BLD-06. Pacotes de outros módulos que importam o SDK (escopo `extended`) passam pelas mesmas regras. Exceção por linha `//n3x:allow <regra> <motivo>` só para `loop_no_ctx` e `chan_no_ctx`, com motivo obrigatório e aviso `analysis_exception` no manifesto. API `// Deprecated:` gera só aviso

Proteção de dados
- `go.sum` conferido no `sum.golang.org`, exceto módulos em `GOPRIVATE`
- Imagens base fixadas por digest; build reproduzível permite conferir que o `.so` em produção veio daquele Dockerfile
- Integridade do artefato por SHA-256 das seções carregáveis, conferido pelo runtime (protege contra arquivo corrompido ou trocado por engano, não contra alteração intencional)
- `govulncheck` em todo build, só com aviso (ADR-008); os avisos vão para o campo `warnings` do manifesto
- Sem PII: o builder não lê `n3x.yaml`, segredos nem dados de ninguém
- Fora de escopo na v1: assinatura e SBOM

Gestão de segredos
- O builder não recebe nenhum segredo; a chave AES do `ENC[...]` também nunca passa por ele

---

### Observabilidade
Logs
- Texto legível por padrão, uma linha por etapa com duração, por exemplo `[5/9] gerando registro (3 plugins) ... ok 0.4s`
- `--log-format json` para a CI do operador: um evento por etapa com `step`, `status`, `duration_ms` e `code`
- Resumo final: plugins e versões, alvo, arquitetura, hash e avisos do `govulncheck`
- Credenciais em URL e caminhos absolutos do cache de módulos nunca aparecem em log

Métricas
- No build: duração de cada etapa (no log)
- Na CI do `n3x-builder`: taxa de sucesso da matriz alvo × arquitetura, tempo de build com e sem cache contra as metas, resultado da checagem de reprodutibilidade e resultado do teste de carga por patch

Tracing
- Sem tracing na v1 (ADR-009 de plataforma)

Dashboards e alertas
- Alerta na CI: falha da matriz, hashes diferentes na checagem de reprodutibilidade, falha no teste de carga em patch do Envoy 1.40.x ou do Istio 1.31.x
- Alerta quando sai patch novo do Envoy ou do Istio e não existe imagem `candidate` em 7 dias

---

### Riscos arquiteturais e mitigação
#### Patch do Istio quebra o carregamento do `.so`
- **Probabilidade:** alta
- **Impacto:** o N3X não carrega, ou derruba o proxy, num cluster com Istio
- **Mitigação:**
  - CI por patch do Istio com comparação de símbolos e teste de carga na imagem exata
  - Patch novo entra como `candidate` e só vira `supported` após passar
- **Plano de contingência:** manter fixado o último patch que passou e documentar o uso do sidecar sem Istio naquele serviço

#### SDK Go do Envoy, sem tag, muda e quebra o adaptador
- **Probabilidade:** alta
- **Impacto:** alvo novo ou patch não compila ou muda de comportamento
- **Mitigação:**
  - Pseudo-versão fixada no catálogo e confirmada com `go list -m`
  - CI contra o próximo candidato antes de atualizar o catálogo
- **Plano de contingência:** segurar o alvo na pseudo-versão anterior

#### `.so` arm64 compilado de forma cruzada passa no build e cai no runtime
- **Probabilidade:** média
- **Impacto:** sidecar arm64 não sobe, sem aviso no build do operador
- **Mitigação:**
  - Teste de carga em arm64 nativo na CI do builder para cada imagem
  - glibc alvo fixada no `zig cc`
- **Plano de contingência:** recomendar o build em runner arm64 nativo; o `n3x run` falha na subida, antes de receber tráfego

#### Análise estática deixa passar um contorno do isolamento
- **Probabilidade:** média
- **Impacto:** plugin lê memória de outro ou derruba o Envoy
- **Mitigação:**
  - Corpus de plugins "maliciosos" em `testdata/`, que cresce a cada contorno descoberto
- **Plano de contingência:** versão nova do builder com a regra e recompilação do plugin

#### Imagens com semver próprio combinadas errado pelo operador
- **Probabilidade:** média
- **Impacto:** sidecar ou controller não sobe
- **Mitigação:**
  - Regra de mesma major e minor igual ou maior na subida, com mensagem que diz qual versão usar
  - Tabela de pares testados no README
- **Plano de contingência:** o runtime recusa antes de receber tráfego; o operador ajusta a tag

#### Build não reproduzível por causa do cgo e do `zig cc`
- **Probabilidade:** média
- **Impacto:** hashes diferentes para a mesma entrada, e a prova de origem perde valor
- **Mitigação:**
  - Checagem dupla de reprodutibilidade na CI, em amd64 e arm64
- **Plano de contingência:** documentar a limitação para arm64 até a correção

#### Sem lockfile, um módulo ou tag é removido ou fica fora do ar
- **Probabilidade:** baixa
- **Impacto:** build falha com `N3X-B001`
- **Mitigação:**
  - Versões exatas no `add`
  - Cache do Go proxy e `GOPROXY` corporativo opcional
- **Plano de contingência:** o operador aponta o build para o proxy corporativo

---

### ADRs e próximos passos
ADRs associados
- [ADR-001 – Semver próprio por imagem e regra de compatibilidade no runtime](adr/ADR-001-semver-por-imagem.md)
- [ADR-002 – Hash SHA-256 das seções carregáveis, sem assinatura na v1](adr/ADR-002-hash-das-secoes-carregaveis.md)
- [ADR-003 – Manifesto JSON na seção ELF `.n3x_manifest`](adr/ADR-003-manifesto-em-secao-elf.md)
- [ADR-004 – Comparação de símbolos e teste de carga na arquitetura nativa](adr/ADR-004-simbolos-e-teste-de-carga.md)
- [ADR-005 – Lista de plugins só por `RUN n3x-plugin add`](adr/ADR-005-lista-por-run-add.md)
- [ADR-006 – Códigos de erro estáveis `N3X-Bxxx`](adr/ADR-006-codigos-de-erro.md)
- [ADR-007 – Build reproduzível](adr/ADR-007-build-reproduzivel.md)
- [ADR-008 – `govulncheck` só avisa](adr/ADR-008-govulncheck-so-avisa.md)
- [ADR-009 – Imagens publicadas no GHCR](adr/ADR-009-imagens-no-ghcr.md)

Decisões pendentes
- Versão exata da glibc das imagens do Envoy 1.40 e do Istio 1.31, que limita o toolchain C
- Pseudo-versão do SDK Go do Envoy e patch exato do Istio a declarar no catálogo
- Assinatura e SBOM em versão futura
- Se o bootstrap mínimo do teste de carga funciona igual em todos os alvos

Próximos passos
- Criar `targets/envoy-1.40.yaml` com a pseudo-versão confirmada
- TDD do pipeline `n3x-plugin build`, começando pelos cenários `N3X-B001` a `N3X-B016`
- Montar o corpus "malicioso" da análise estática
- Escrever as imagens de build e a CI de matriz com teste de reprodutibilidade
- Teste de carga do `.so` no istio-proxy 1.31
