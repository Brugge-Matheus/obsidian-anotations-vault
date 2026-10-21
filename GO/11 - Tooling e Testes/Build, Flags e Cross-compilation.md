---
tags:
  - go
  - go/tooling
aliases:
  - go build
  - Cross-compilation
  - Compilação cruzada
  - ldflags
  - Build tags
  - CGO_ENABLED
  - go generate
  - Binário estático
ordem: 48
modulo: "11 - Tooling e Testes"
---
# Build, Flags e Cross-compilation

> **Nota 48 da trilha de Go.** Fechamos o módulo de ferramentas com o caminho do código até o **artefato que vai para produção**. Em [[Como Go Funciona]] vimos o pipeline do compilador e o que existe dentro de um binário; aqui o foco é **controlar** esse processo: compilar para **outras plataformas** com duas variáveis de ambiente, **injetar a versão** no binário, ler as informações de build que o Go grava sozinho (inclusive o commit do Git), reduzir tamanho, gerar builds **reprodutíveis**, compilação condicional com **build tags** e sufixos de arquivo, o papel do **cgo** (binário estático vs dinâmico), níveis de microarquitetura (`GOAMD64`), `go generate`, o diretivo `tool` do `go.mod` e um Dockerfile de produção. Todos os números e saídas são reais.
>
> **Pré-requisitos:** [[Como Go Funciona]], [[Sistema de Módulos (go mod)]], [[Criando Pacotes]].

---

## 🧭 De onde viemos

[[Como Go Funciona]] mostrou as etapas do compilador (parsing, tipos, SSA, código de máquina, linker), as seções de um binário ELF e que o runtime fala com o kernel **sem libc** no Linux, prometendo "os detalhes em Build, Flags e Cross-compilation". [[Ambiente e Primeiro Programa]] apresentou `go build`/`go run`/`go install`, [[Sistema de Módulos (go mod)]] mostrou o `go.mod` e a seleção de versões, e [[Criando Pacotes]] o `//go:embed`. Em [[Sintaxe, Blocos e Escopo]] vimos as diretivas `//go:`. Do lado do SO, [[Arquivos de Cabeçalho e o Modelo de Execução]] apresentou compilação, *linking* e segmentos, e [[System Calls]] a fronteira entre programa e kernel.

> 💡 **Artefato de build:** o que sai do processo de compilação e é distribuído: em Go, normalmente **um único executável** que contém seu código, as dependências e o runtime.

---

## 🧰 1. Os comandos

| Comando | Faz |
| --- | --- |
| `go build` | Compila o pacote do diretório; se for `main`, gera o executável |
| `go build -o bin/app ./cmd/app` | Nome e pacote explícitos |
| `go build ./...` | Compila tudo (verifica que compila; não gera binários de bibliotecas) |
| `go run .` | Compila num diretório temporário e executa |
| `go install ./cmd/app` | Compila e instala em `$(go env GOBIN)` ou `$GOPATH/bin` |
| `go install exemplo.com/ferramenta@v1.4.0` | Instala uma ferramenta de outro módulo, numa versão exata |
| `go vet ./...` | Análise estática (ver [[Convenções da linguagem]]) |
| `go clean -cache` | Esvazia o cache de build |

### 1.1 O cache de build

Go guarda cada pacote compilado num cache indexado pelo **hash de tudo que influencia o resultado** (código, dependências, flags, versão do compilador, variáveis como `GOOS`). Saída real, compilando o mesmo programa (que importa `net/http`) com o cache vazio e logo depois:

```
build com cache vazio: 17226 ms
build de novo (cache): 394 ms
$ go env GOCACHE
/root/.cache/go-build
```

Quase toda a demora da primeira vez é compilar a biblioteca padrão (`net/http`, `crypto/tls`...). Em CI, **guardar o diretório do cache** entre execuções é a otimização de build mais efetiva que existe.

```bash
go build -x .     # mostra cada comando executado (compile, link...)
go build -v .     # lista os pacotes recompilados
go build -json .  # Go 1.24+: saída e erros de build em JSON (para ferramentas)
```

---

## 🌐 2. Compilação cruzada

Como o compilador gera código nativo e o runtime não depende da libc (no Linux), trocar de plataforma é trocar **duas variáveis**. Saída real, compilando o **mesmo** programa numa máquina Linux/amd64:

```bash
for t in linux/amd64 linux/arm64 windows/amd64 darwin/arm64 js/wasm; do
	CGO_ENABLED=0 GOOS=${t%/*} GOARCH=${t#*/} go build -o bin/app-${t%/*}-${t#*/} .
done
```

```
bin/app-linux-amd64      5.1 MB  ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked
bin/app-linux-arm64      4.8 MB  ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked
bin/app-windows-amd64    5.2 MB  PE32+ executable (console) x86-64, for MS Windows
bin/app-darwin-arm64     4.9 MB  Mach-O 64-bit arm64 executable
bin/app-js-wasm          4.7 MB  WebAssembly (wasm) binary module version 0x1 (MVP)
```

Cinco formatos de executável diferentes (ELF, PE, Mach-O, WebAssembly) e três conjuntos de instruções (x86-64, ARM64, WASM), sem instalar nada. `go tool dist list` mostra todas as **47** combinações suportadas pelo Go 1.27.

| Alvo | Uso típico |
| --- | --- |
| `linux/amd64` | Servidores, containers |
| `linux/arm64` | AWS Graviton, Raspberry Pi 4/5, Macs rodando containers |
| `darwin/arm64` | Macs com Apple Silicon |
| `windows/amd64` | Desktop Windows |
| `js/wasm`, `wasip1/wasm` | Navegador; runtimes WASI (servidor, edge) |

> 💡 O que muda por dentro: o **backend** de geração de código (instruções e registradores de cada ISA, ver [[Processadores]]) e a **camada de syscalls** do runtime (`epoll` no Linux, `kqueue` no macOS, IOCP no Windows).

### 2.1 Níveis de microarquitetura: `GOAMD64`

Nem todo x86-64 tem as mesmas instruções. Por padrão (`GOAMD64=v1`), o compilador gera código que roda até em CPUs de 2003, e, para instruções mais novas, **verifica em tempo de execução** se a CPU as tem. Saída real do `objdump` de uma função que conta bits com `bits.OnesCount64`:

```
== GOAMD64=v1
MOVZX runtime.x86HasPOPCNT(SB), CX     ← "esta CPU tem POPCNT?" (verificado a cada chamada)
POPCNTQ AX, AX                          ← se tiver
CALL math/bits.OnesCount64(SB)          ← se não tiver: versão em software
== GOAMD64=v3
POPCNTQ AX, AX                          ← uma instrução, sem verificação
```

| Nível | Exige (resumo) | CPUs |
| --- | --- | --- |
| `v1` (padrão) | x86-64 básico | Qualquer uma |
| `v2` | + POPCNT, SSE4.2 | ~2009+ |
| `v3` | + AVX2, BMI, FMA | Intel Haswell (2013)+, AMD Zen |
| `v4` | + AVX-512 | Servidores recentes |

Um binário `v3` **não roda** numa CPU sem essas instruções. Use quando você controla o hardware de produção (o equivalente em ARM é `GOARM64`).

---

## 🏷️ 3. Informações de versão no binário

### 3.1 O que o Go grava sozinho

Desde o Go 1.18, o `go build` grava no binário o módulo, as dependências, as flags e o estado do **Git**; desde o **1.24**, a versão do módulo principal vem da **tag** do Git. Saída real (`go build` num repositório com a tag `v1.2.0`):

```go
if info, ok := debug.ReadBuildInfo(); ok {
	fmt.Println("módulo principal:", info.Main.Path, info.Main.Version)
	for _, s := range info.Settings {
		if s.Key == "vcs.revision" || s.Key == "vcs.modified" || s.Key == "vcs.time" {
			fmt.Printf("  %s=%s\n", s.Key, s.Value)
		}
	}
}
```

```
módulo principal: exemplo.com/app v1.2.0
  vcs.revision=14b27fb2723545d328df5dba8ff5b27012881d0a
  vcs.time=2026-10-21T04:11:59Z
  vcs.modified=false
```

(Com `go run`, a versão aparece como `(devel)` e sem dados do Git; com alterações não commitadas, `vcs.modified=true` e a versão ganha o sufixo `+dirty`.) O mesmo é lido **de fora**, sem executar o binário:

```
$ go version -m bin/app
	path	exemplo.com/app
	mod	exemplo.com/app	v1.2.0
	build	-ldflags="-X main.Versao=1.2.0 -X main.Commit=14b27fb"
	build	CGO_ENABLED=1
	build	GOAMD64=v1
	build	vcs=git
	build	vcs.revision=14b27fb2723545d328df5dba8ff5b27012881d0a
	build	vcs.modified=false
```

É o que ferramentas como o `govulncheck` usam para verificar se um binário em produção contém dependências vulneráveis (ver [[Sistema de Módulos (go mod)]]).

### 3.2 Injetando valores: `-ldflags -X`

O linker pode substituir o valor de uma variável `string` de pacote:

```go
var (
	Versao = "dev" // sobrescrito com -ldflags "-X main.Versao=..."
	Commit = "desconhecido"
)
```

```bash
go build -ldflags "-X main.Versao=1.2.0 -X main.Commit=$(git rev-parse --short HEAD)" -o bin/app .
./bin/app
# app 1.2.0 (commit 14b27fb) linux/amd64, go1.27.0
```

O caminho é o **caminho de importação completo** do pacote (`-X exemplo.com/app/internal/versao.Versao=...`). Só funciona com variáveis `string` não inicializadas por função. Com as informações automáticas da seção 3.1, muita gente nem precisa mais disso.

---

## 📦 4. Tamanho, caminhos e builds reprodutíveis

### 4.1 `-s -w`

Saída real:

```
normal  5394164 bytes      (go build)
sw      3621127 bytes      (go build -ldflags="-s -w")   ← 33% menor
```

| Flag | Remove | Perde |
| --- | --- | --- |
| `-s` | Tabela de símbolos (`.symtab`) | Ferramentas de análise que dependem dela |
| `-w` | Informação de depuração DWARF | Depurar com Delve/gdb |

Os **stack traces** de panics e os **profiles** continuam funcionando, porque usam a tabela própria do Go (`.gopclntab`), que não é removida (ver [[Profiling e Diagnóstico]]).

### 4.2 `-trimpath`

Sem ela, o binário contém o caminho absoluto dos arquivos na máquina de build (útil em stack traces, mas vaza informação e impede reprodutibilidade). Saída real, contando ocorrências do diretório de build nas strings do binário:

```
$ strings bin/normal | grep -c "/tmp/gt/ex/n48/app"     →  1
$ strings bin/trim   | grep -c "/tmp/gt/ex/n48/app"     →  0      (go build -trimpath)
```

### 4.3 Builds reprodutíveis

Com `-trimpath`, o mesmo código, a mesma versão do Go e as mesmas flags geram **exatamente os mesmos bytes**. Saída real: dois builds, com o cache apagado entre eles:

```
$ go build -trimpath -o bin/r1 . && go clean -cache && go build -trimpath -o bin/r2 .
$ sha256sum bin/r1 bin/r2
aad161775ed5826afa9b...  bin/r1
aad161775ed5826afa9b...  bin/r2
```

Reprodutibilidade permite que qualquer pessoa **verifique** que um binário distribuído veio mesmo daquele código-fonte (defesa contra ataques à cadeia de suprimentos). A própria toolchain de Go é reprodutível desde o 1.21.

---

## 🧩 5. Compilação condicional

### 5.1 Sufixos de arquivo

Arquivos terminados em `_GOOS.go`, `_GOARCH.go` ou `_GOOS_GOARCH.go` só são compilados naquela plataforma. Saída real, listando os arquivos que entram no build de cada sistema (`go list -f '{{.GoFiles}}'`):

```
app/
├── main.go
├── plat_linux.go       ← só no Linux
├── plat_windows.go     ← só no Windows
└── plat_outros.go      ← //go:build !linux && !windows
```

```
linux:   [main.go plat_linux.go]
windows: [main.go plat_windows.go]
darwin:  [main.go plat_outros.go]
```

Cada arquivo define a mesma função `plataforma()`; o compilador vê só uma por plataforma. É assim que a biblioteca padrão separa, por exemplo, o netpoller de cada SO (`netpoll_epoll.go`, `netpoll_kqueue.go`, `netpoll_windows.go`).

### 5.2 A linha `//go:build`

Para condições arbitrárias, a primeira linha do arquivo (antes do `package`, seguida de uma linha em branco):

```go
//go:build linux && (amd64 || arm64)

//go:build !windows

//go:build integration          // tag personalizada: go test -tags integration ./...

//go:build go1.27               // só compila com Go 1.27 ou mais novo
```

```bash
go build -tags "integration,debug" ./...
```

Usos comuns: testes de integração que precisam de banco (`-tags integration`), recursos opcionais, implementações alternativas (`-tags purego` para desligar assembly). É o equivalente aos `#ifdef` do pré-processador C ([[Arquivos de Cabeçalho e o Modelo de Execução]]), só que por **arquivo inteiro**, não por trecho.

> ⚠️ A sintaxe antiga `// +build` foi substituída por `//go:build` no Go 1.17; o `go fix` converte, e o `gofmt` mantém as duas sincronizadas se ambas existirem.

---

## 🔗 6. cgo: estático ou dinâmico

**cgo** permite chamar código C a partir de Go (veremos em [[unsafe e cgo]]). Ele também muda **como o binário é linkado**. Saída real, o mesmo programa (que importa `net/http`):

```
$ go build -o bin/normal .            # CGO_ENABLED=1 (padrão quando há compilador C)
$ file bin/normal
ELF 64-bit LSB executable, x86-64, ..., dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2
$ ldd bin/normal
	linux-vdso.so.1
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6
	/lib64/ld-linux-x86-64.so.2

$ CGO_ENABLED=0 go build -o bin/static .
$ file bin/static
ELF 64-bit LSB executable, x86-64, ..., statically linked
$ ldd bin/static
	not a dynamic executable
```

Por que o `net` puxou a libc? Com cgo ligado, o pacote `net` pode usar o resolvedor de DNS da libc (`getaddrinfo`) e o `os/user` pode usar o NSS do sistema (ver [[Sockets e o Pacote net]]). Com `CGO_ENABLED=0`, ambos usam implementações em Go puro e o binário fica totalmente estático.

| | `CGO_ENABLED=1` | `CGO_ENABLED=0` |
| --- | --- | --- |
| Padrão | Quando há um compilador C no `PATH` (desde o Go 1.20; senão vira 0) | Em compilação cruzada, ou sem compilador C |
| Linkagem | Dinâmica com a libc (se algum pacote usar cgo) | Estática |
| Roda em `FROM scratch` / distroless static | ❌ (falta a libc e o carregador) | ✅ |
| Compilação cruzada | Exige um compilador C **para o alvo** | Trivial |
| Necessário para | SQLite via `mattn/go-sqlite3`, bibliotecas C, plugins | Quase todo serviço Go |

> ✅ Para servidores em containers, `CGO_ENABLED=0` é a escolha padrão: binário estático, imagem mínima, sem a "glibc errada" (*glibc hell*).

---

## ⚙️ 7. Outras flags e variáveis

| Flag / variável | Efeito |
| --- | --- |
| `-race` | Detector de corridas (ver [[Fundamentos de Concorrência]]) |
| `-cover` | Binário instrumentado para cobertura (ver [[Testes com go test]]) |
| `-pgo=auto` (padrão) | Usa `default.pgo` do pacote `main` (ver [[Profiling e Diagnóstico]]) |
| `-gcflags='-m'` | Decisões de inlining e escape analysis (ver [[Memória e Escape Analysis]]) |
| `-gcflags='all=-N -l'` | Sem otimizações e sem inlining: para depurar com Delve |
| `-buildmode=c-shared` / `c-archive` | Biblioteca chamável por C (`.so`, `.a`) |
| `-buildmode=pie` | Executável independente de posição (ASLR); padrão em algumas plataformas |
| `GOFLAGS="-trimpath -mod=readonly"` | Flags padrão para todos os comandos `go` |
| `GOEXPERIMENT=...` | Liga/desliga recursos experimentais do compilador e do runtime |
| `GOTOOLCHAIN=go1.27.0` | Versão da toolchain (ver [[Ambiente e Primeiro Programa]]) |

### 7.1 `GODEBUG` fixado no build

Comportamentos que mudaram entre versões de Go têm uma chave `GODEBUG` para voltar ao antigo. O padrão de cada chave depende da linha `go` do `go.mod`, e pode ser fixado no código:

```go
//go:debug tracebacklabels=0       // num arquivo do pacote main, antes do package
package main
```

```
// ou no go.mod (Go 1.23+)
godebug (
	multipathtcp=0
)
```

Vimos várias dessas chaves ao longo da trilha: `asynctimerchan` (removida no 1.27), `multipathtcp`, `tracebacklabels`, `containermaxprocs`.

---

## 🏭 8. Gerando código: `go generate`

`go generate` executa comandos escritos em comentários `//go:generate`. Ele **não** roda no `go build`: é um passo manual (ou de CI) que gera arquivos `.go` que **são** commitados. Saída real com um gerador escrito no próprio módulo, que cria o método `String()` de uma enumeração (o que a ferramenta `stringer` faz):

```go
//go:generate go run ./internal/gerador -tipo Cor -valores Vermelho,Verde,Azul -saida cor_string.go

type Cor int

const (
	Vermelho Cor = iota
	Verde
	Azul
)
```

```
$ go generate -x ./...
go run ./internal/gerador -tipo Cor -valores Vermelho,Verde,Azul -saida cor_string.go
gerado: cor_string.go

$ head -3 cor_string.go
// Code generated by gerador; DO NOT EDIT.

package main
```

> ⚠️ A primeira linha precisa seguir **exatamente** o padrão `// Code generated ... DO NOT EDIT.` (em inglês, com o ponto final). É assim que o `gofmt`, linters, o GitHub e as revisões de código reconhecem arquivos gerados e os tratam de forma diferente.

Usos comuns: `stringer` (enums), `protoc-gen-go` (Protocol Buffers/gRPC), `sqlc` (consultas SQL tipadas), mocks.

### 8.1 Ferramentas do projeto: o diretivo `tool` (Go 1.24+)

Geradores e linters também são dependências, e precisam de versão fixa. Desde o Go 1.24, o `go.mod` registra ferramentas:

```bash
go get -tool golang.org/x/tools/cmd/stringer@latest   # a versão escolhida fica no go.mod
```

```
// go.mod
tool golang.org/x/tools/cmd/stringer
```

```go
//go:generate go tool stringer -type=Cor
```

O `go tool stringer` compila (com cache) e roda **exatamente** a versão do `go.mod`, em todas as máquinas. Substitui o antigo truque do arquivo `tools.go` com imports em branco.

---

## 🐳 9. Do build ao container

Juntando tudo num Dockerfile *multi-stage* (ver [[Dockerfile]] e [[Imagens]]):

```dockerfile
# ---------- estágio 1: build ----------
FROM golang:1.27 AS build
WORKDIR /src

COPY go.mod go.sum ./
RUN go mod download                      # camada de dependências, reaproveitada se go.mod não mudar

COPY . .
ARG VERSAO=dev
RUN CGO_ENABLED=0 GOOS=linux go build \
      -trimpath \
      -ldflags="-s -w -X main.Versao=${VERSAO}" \
      -o /out/servidor ./cmd/servidor

# ---------- estágio 2: execução ----------
FROM gcr.io/distroless/static-debian12:nonroot   # ou: FROM scratch
COPY --from=build /out/servidor /servidor
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/servidor"]
```

| Decisão | Por quê |
| --- | --- |
| Dois estágios | A imagem final não carrega o compilador (~1 GB): só o binário (~5–15 MB) |
| `COPY go.mod go.sum` antes do código | O Docker reaproveita a camada de dependências entre builds |
| `CGO_ENABLED=0` | Binário estático: roda sem libc |
| `distroless/static` | Inclui certificados TLS raiz, `/etc/passwd` com usuário não-root e dados de fuso horário; `scratch` não inclui nada |
| `USER nonroot` | Não rodar como root dentro do container (ver [[Segurança]]) |
| `ENTRYPOINT` em forma de lista | O binário vira o PID 1 e recebe o `SIGTERM` direto (ver [[Processo raiz (PID 1)]] e [[context.Context]]) |

> 💡 Com `FROM scratch`, chamadas HTTPS falham com `x509: certificate signed by unknown authority` se você não copiar `/etc/ssl/certs/ca-certificates.crt`. Alternativa em Go puro: importar `_ "golang.org/x/crypto/x509roots/fallback"`, que embute os certificados raiz no binário. Para fusos horários, `import _ "time/tzdata"` embute a base de dados.

---

## 🔬 10. Por dentro: do `go build` ao processo

```
go build
  │  1. resolve o grafo de pacotes (go.mod, build tags, GOOS/GOARCH, sufixos de arquivo)
  │  2. para cada pacote, em paralelo e em ordem topológica:
  │       hash das entradas → está no GOCACHE? → reaproveita
  │       senão: compile (gc) → arquivo objeto do pacote (.a), guardado no cache
  │       pacotes com cgo: gcc/clang compila o C; o cgo gera o código de ponte
  │  3. link (cmd/link): junta os objetos + runtime, elimina código morto,
  │     aplica -X e -s/-w, grava buildinfo, gopclntab, build ID
  ▼
binário (ELF / PE / Mach-O / wasm)
  │
  ▼  execve (ver Syscalls para Gerenciamento de Processos)
kernel: mapeia os segmentos (text, rodata, data, bss) na memória virtual
  │   estático: salta direto para _rt0_amd64_linux
  │   dinâmico: primeiro o carregador ld-linux.so resolve a libc
  ▼
runtime: inicializa heap, Ms, Ps, a goroutine main → init() dos pacotes → main.main
```

O **build ID** (gravado no binário e, desde o Go 1.24, também como nota `NT_GNU_BUILD_ID` no ELF) é o hash usado pelo cache e por ferramentas de depuração para casar um binário com seus símbolos.

---

## ⚠️ 11. Armadilhas

| Armadilha | Sintoma | Correção |
| --- | --- | --- |
| Binário com cgo numa imagem `scratch`/Alpine | `no such file or directory` ao executar (falta o carregador/libc) | `CGO_ENABLED=0` |
| Compilação cruzada com cgo | Erros do compilador C | `CGO_ENABLED=0` ou toolchain C do alvo |
| `GOAMD64=v3` em hardware antigo | `illegal instruction` (SIGILL) | Só com hardware controlado |
| `-X` com caminho de pacote errado | Valor não muda, sem erro | Caminho de importação completo |
| Sem `-trimpath` | Caminhos da máquina de build no binário; builds não reprodutíveis | `-trimpath` |
| Esperar que `go build` rode `go generate` | Código gerado desatualizado | Rodar `go generate` no fluxo e commitar o resultado |
| Cabeçalho de código gerado fora do padrão | Ferramentas tratam como código manual | `// Code generated ... DO NOT EDIT.` |
| `scratch` sem certificados | Erros `x509` em HTTPS | `distroless/static` ou copiar os certificados |
| Ferramentas instaladas "na máquina de cada um" | Versões diferentes geram código diferente | Diretivo `tool` no `go.mod` |
| CI sem cache de build | Builds lentos | Persistir `GOCACHE` e o cache de módulos |

---

## 📋 Resumo

| Conceito | Ponto-chave |
| --- | --- |
| Cache | Por hash das entradas; 17 s → 0,4 s; persista no CI |
| Cross-compilation | `GOOS` + `GOARCH`; 47 alvos; ELF/PE/Mach-O/wasm |
| `GOAMD64` | Nível de microarquitetura; `v3` dispensa verificações em tempo de execução |
| Build info | Módulo, dependências, flags e Git gravados; `go version -m`, `debug.ReadBuildInfo`; versão pela tag (1.24+) |
| `-ldflags -X` | Injeta strings em variáveis de pacote |
| `-s -w` | −33% de tamanho; stack traces continuam |
| `-trimpath` | Sem caminhos locais; builds reprodutíveis (mesmo hash) |
| Build tags | Sufixos `_linux.go`; `//go:build expr`; `-tags` |
| cgo | Dinâmico com libc vs estático; `CGO_ENABLED=0` para containers |
| `GODEBUG` | `//go:debug` e bloco `godebug` no `go.mod` |
| `go generate` | Comandos `//go:generate`; resultado commitado; cabeçalho padrão |
| `tool` (1.24) | Ferramentas versionadas no `go.mod`; `go tool nome` |
| Docker | Multi-stage, `CGO_ENABLED=0`, distroless/scratch, não-root |

---

## 🔗 Conexões

**Dentro de Go**
- O pipeline do compilador e as seções do binário → [[Como Go Funciona]]
- `go build`, `go run`, `GOTOOLCHAIN` → [[Ambiente e Primeiro Programa]]
- `go.mod`, versões, `govulncheck` → [[Sistema de Módulos (go mod)]]
- `//go:embed`, `internal/` → [[Criando Pacotes]]
- Diretivas `//go:` → [[Sintaxe, Blocos e Escopo]]
- `-race`, `-cover` → [[Fundamentos de Concorrência]], [[Testes com go test]]
- PGO e `.gopclntab` → [[Profiling e Diagnóstico]]
- `-gcflags=-m` → [[Memória e Escape Analysis]]
- cgo em profundidade → [[unsafe e cgo]]
- Resolvedor de DNS Go vs libc → [[Sockets e o Pacote net]]
- `SIGTERM` e desligamento → [[context.Context]], [[HTTP (net-http)]]

**Sistemas Operacionais**
- [[Arquivos de Cabeçalho e o Modelo de Execução]] — compilação, linkagem, segmentos, `#ifdef`
- [[A Linguagem C]] — a libc e o modelo de compilação de C
- [[Sistemas Monolíticos]] — bibliotecas compartilhadas vs estáticas
- [[System Calls]] e [[Syscalls para Gerenciamento de Processos]] — `execve` e syscalls sem libc
- [[Processadores]] — conjuntos de instruções (ISA), extensões como POPCNT e AVX2
- [[Espaços de Endereçamento]] e [[Memória Virtual]] — como o kernel mapeia o executável

**Docker**
- [[Dockerfile]] e [[Imagens]] — multi-stage e camadas
- [[Segurança]] — usuário não-root e imagens mínimas
- [[Processo raiz (PID 1)]] — o binário como PID 1

---

⬅️ **Anterior:** [[Profiling e Diagnóstico]] · 🗺️ [[MOC - GO|Mapa de Go]] · **Próxima:** [[Memória e Escape Analysis]] ➡️
