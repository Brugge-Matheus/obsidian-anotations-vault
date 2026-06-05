---
tags:
  - go
  - go/tooling
---
# Build, Flags e Cross-compilation

> O toolchain de Go produz binários nativos autocontidos. Cross-compilation é trivial — um único comando compila para qualquer plataforma suportada. Ldflags permitem injetar metadados em tempo de build, build tags habilitam compilação condicional, e o compilador SSA aplica otimizações modernas.
> 

---

## 1. Como o Compilador Go Funciona

```
Pipeline de compilação:

Fonte .go
    ↓
Parsing — AST (Abstract Syntax Tree)
    ↓
Type Checking — verificação estática de tipos
    ↓
SSA IR (Static Single Assignment) — representação intermediária
    ↓ Otimizações SSA:
      - Inlining de funções pequenas
      - Escape analysis (stack vs heap)
      - Dead code elimination
      - Constant folding
      - Strength reduction (x*8 → x<<3)
    ↓
Machine code — backend específico da arquitetura
  (AMD64, ARM64, WASM, etc.)
    ↓
Linker estático
    ↓
Binário autocontido (inclui runtime Go, GC, scheduler)
```

O binário Go inclui:

- Runtime do Go (GC, goroutine scheduler, stack management)
- Todas as dependências (linkagem estática por padrão com `CGO_ENABLED=0`)
- **Não inclui**: libc separada quando CGo está desabilitado

---

## 2. Comandos Básicos

```bash
# Build — gera executável
go build                          # usa o go.mod do diretório atual
go build -o bin/servidor          # nome customizado
go build ./cmd/servidor/          # pacote específico
go build ./...                    # verifica que todos os pacotes compilam

# Run — compila e executa sem gerar arquivo permanente
go run .
go run ./cmd/servidor/

# Install — compila e instala em $(go env GOPATH)/bin
go install github.com/usuario/ferramenta@latest
go install ./cmd/servidor/

# Clean — remove artifacts de build
go clean
go clean -cache                   # limpa cache de build
go clean -modcache                # limpa cache de módulos (cuidado!)
```

---

## 3. Cross-compilation — Compilar para Outras Plataformas

```bash
# Sintaxe: GOOS=sistema GOARCH=arquitetura go build

# Servidores Linux (mais comum em produção)
GOOS=linux   GOARCH=amd64  go build -o bin/servidor-linux-amd64
GOOS=linux   GOARCH=arm64  go build -o bin/servidor-linux-arm64

# Windows
GOOS=windows GOARCH=amd64  go build -o bin/servidor.exe
GOOS=windows GOARCH=arm64  go build -o bin/servidor-arm64.exe

# macOS
GOOS=darwin  GOARCH=amd64  go build -o bin/servidor-mac-intel
GOOS=darwin  GOARCH=arm64  go build -o bin/servidor-mac-silicon

# ARM para IoT / Raspberry Pi
GOOS=linux   GOARCH=arm    GOARM=7  go build -o bin/servidor-armv7
GOOS=linux   GOARCH=arm    GOARM=6  go build -o bin/servidor-armv6

# WebAssembly (Go 1.11+)
GOOS=js      GOARCH=wasm   go build -o main.wasm .

# WASI — WebAssembly System Interface (Go 1.21+)
GOOS=wasip1  GOARCH=wasm   go build -o main.wasm .
```

```bash
# Listar todos os pares GOOS/GOARCH suportados (são mais de 40)
go tool dist list

# Pares mais comuns em produção:
# linux/amd64, linux/arm64, linux/arm
# darwin/amd64, darwin/arm64
# windows/amd64, windows/386
# freebsd/amd64
# js/wasm, wasip1/wasm
```

---

## 4. Ldflags — Injetar Dados em Tempo de Build

O linker pode injetar valores em variáveis `string` de pacotes:

```go
// version/version.go
package version

var (
	Versao    = "dev"          // sobrescrito pelo build
	CommitSHA = "unknown"
	DataBuild = "unknown"
	GoVersion = "unknown"
)

func Info() string {
	return fmt.Sprintf("%s (commit: %s, build: %s, go: %s)",
		Versao, CommitSHA, DataBuild, GoVersion)
}
```

```bash
# Injetar com -ldflags "-X caminho/do/pacote.Variavel=valor"
go build \
	-ldflags " \
		-X github.com/usuario/projeto/version.Versao=$(git describe --tags --always) \
		-X github.com/usuario/projeto/version.CommitSHA=$(git rev-parse --short HEAD) \
		-X 'github.com/usuario/projeto/version.DataBuild=$(date -u +%Y-%m-%dT%H:%M:%SZ)' \
		-X github.com/usuario/projeto/version.GoVersion=$(go version | awk '{print $3}') \
	" \
	-o bin/servidor ./cmd/servidor/
```

### Reduzir Tamanho do Binário

```bash
# -s: remove tabela de símbolos (symbol table)
# -w: remove informações de debug DWARF
# Resultado: ~30% menor, sem impacto em performance
go build -ldflags "-s -w" -o bin/servidor

# -trimpath: remove caminhos locais do código fonte do binário
# Útil para builds reproduzíveis e para não expor paths da máquina de build
go build -trimpath -ldflags "-s -w" -o bin/servidor

# Comparação de tamanhos:
# Normal:          15.2 MB
# Com -s -w:       10.1 MB
# Com upx --best:   3.8 MB (compressão externa, requer upx instalado)
```

---

## 5. Build Tags — Compilação Condicional

Arquivos inteiros são incluídos ou excluídos com base em constraints:

```go
// Deve ser a PRIMEIRA linha do arquivo (antes do package)
// Sintaxe Go 1.17+: //go:build constraint
// A linha antiga "// +build constraint" ainda é aceita mas obsoleta

// Apenas Linux
//go:build linux
package main

// Linux OU macOS
//go:build linux || darwin
package main

// Linux E AMD64
//go:build linux && amd64
package main

// Negação — tudo exceto Windows
//go:build !windows
package main

// Tag customizada
//go:build integration
package mytest

// Versão mínima do Go
//go:build go1.22
package main

// Combinações
//go:build (linux || darwin) && !386 && go1.21
package main
```

```bash
# Build com tag customizada
go build -tags debug ./...
go test  -tags "integration postgres" ./...
```

### Casos de Uso Práticos

```go
// arquivo: log_prod.go
//go:build !debug

package main

func init() {
	slog.SetLogLoggerLevel(slog.LevelInfo)
}

// ---

// arquivo: log_debug.go
//go:build debug

package main

func init() {
	slog.SetLogLoggerLevel(slog.LevelDebug)
	slog.Info("modo debug ativo")
}
```

```go
// arquivo: db_postgres.go
//go:build postgres || (!sqlite && !mysql)

package db

import _ "github.com/lib/pq"

func conectar(url string) (*sql.DB, error) {
	return sql.Open("postgres", url)
}

// ---

// arquivo: db_sqlite.go
//go:build sqlite

package db

import _ "github.com/mattn/go-sqlite3"

func conectar(url string) (*sql.DB, error) {
	return sql.Open("sqlite3", url)
}
```

---

## 6. Visualizar Otimizações do Compilador

```bash
# Escape analysis — quais variáveis vão para a heap
go build -gcflags="-m" ./...
# ./main.go:8:2: moved to heap: x
# ./main.go:15:3: &Config{} escapes to heap

# Inlining — quais funções foram inlined
go build -gcflags="-m=2" ./...
# ./util.go:5:6: can inline Somar with cost 4 as: func(int, int) int { return a + b }
# ./main.go:12:14: inlining call to Somar

# Desabilitar otimizações (para debugging com Delve)
go build -gcflags="all=-N -l" ./...
# -N: desabilita todas as otimizações
# -l: desabilita inlining

# Ver assembly gerado
GOOS=linux GOARCH=amd64 go tool compile -S arquivo.go
go tool objdump -s "^main\." bin/servidor   # disassembly do binário final
```

---

## 7. CGo — Interoperabilidade com C

```bash
# CGO_ENABLED=0: binário 100% estático, sem libc
# Padrão em cross-compilation
CGO_ENABLED=0 GOOS=linux go build ./...

# CGO_ENABLED=1 (padrão quando há código C):
# - Requer gcc/clang na máquina
# - Binário linkado dinamicamente com libc
# - NÃO funciona para cross-compilation simples

# Pacotes que requerem CGo (exigem CGO_ENABLED=1):
# - github.com/mattn/go-sqlite3
# - alguns drivers de banco de dados
# - alguns pacotes de criptografia
```

---

## 8. `go generate` — Geração de Código

```go
// Comentário especial que go generate executa
//go:generate stringer -type=DiaSemana    // gera String() para enums
//go:generate mockgen -source=repo.go -destination=mock_repo.go -package=mocks
//go:generate protoc --go_out=. --go-grpc_out=. api.proto
//go:generate go run cmd/gerar-migrations/main.go
//go:generate enumer -type=Status -json -sql
```

```bash
go generate ./...        # executa todos os go:generate no módulo
go generate ./internal/... # específico para um diretório
```

---

## 9. Makefile Completo

```makefile
# Variáveis extraídas do git e ambiente
BINARIO    := servidor
VERSAO     ?= $(shell git describe --tags --always --dirty 2>/dev/null || echo "dev")
COMMIT     := $(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
DATA_BUILD := $(shell date -u +%Y-%m-%dT%H:%M:%SZ)
GO_VERSAO  := $(shell go version | awk '{print $$3}')

PKG_VERSAO = github.com/empresa/$(BINARIO)/version

LDFLAGS := -ldflags "\
	-X $(PKG_VERSAO).Versao=$(VERSAO) \
	-X $(PKG_VERSAO).CommitSHA=$(COMMIT) \
	-X '$(PKG_VERSAO).DataBuild=$(DATA_BUILD)' \
	-X $(PKG_VERSAO).GoVersion=$(GO_VERSAO) \
	-s -w"

FLAGS_BUILD := -trimpath $(LDFLAGS)

.PHONY: all build run test lint fmt generate clean docker

all: test build

build:
	go build $(FLAGS_BUILD) -o bin/$(BINARIO) ./cmd/$(BINARIO)/

build-linux:
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
		go build $(FLAGS_BUILD) -o bin/$(BINARIO)-linux-amd64 ./cmd/$(BINARIO)/

build-arm64:
	CGO_ENABLED=0 GOOS=linux GOARCH=arm64 \
		go build $(FLAGS_BUILD) -o bin/$(BINARIO)-linux-arm64 ./cmd/$(BINARIO)/

build-windows:
	CGO_ENABLED=0 GOOS=windows GOARCH=amd64 \
		go build $(FLAGS_BUILD) -o bin/$(BINARIO).exe ./cmd/$(BINARIO)/

build-darwin:
	CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 \
		go build $(FLAGS_BUILD) -o bin/$(BINARIO)-darwin-arm64 ./cmd/$(BINARIO)/

build-all: build-linux build-arm64 build-windows build-darwin

run:
	go run ./cmd/$(BINARIO)/

test:
	go test -race -coverprofile=coverage.out -covermode=atomic ./...

test-integration:
	go test -race -tags integration -timeout 120s ./...

bench:
	go test -bench=. -benchmem -count=3 -run='^$$' ./...

lint:
	go vet ./...
	staticcheck ./...
	golangci-lint run

fmt:
	gofmt -w .
	goimports -w .

generate:
	go generate ./...

tidy:
	go mod tidy
	go mod verify

clean:
	rm -rf bin/ coverage.out

docker:
	docker build \
		--build-arg VERSAO=$(VERSAO) \
		--build-arg COMMIT=$(COMMIT) \
		--platform linux/amd64 \
		-t $(BINARIO):$(VERSAO) \
		-t $(BINARIO):latest \
		.

version:
	@echo "Versão: $(VERSAO)"
	@echo "Commit: $(COMMIT)"
	@echo "Go:     $(GO_VERSAO)"
```

---

## 10. Dockerfile Multi-stage Otimizado

```docker
# ========================
# Stage 1: Build
# ========================
FROM golang:1.24-alpine AS builder

# Instalar ferramentas necessárias
RUN apk add --no-cache git ca-certificates tzdata make

WORKDIR /build

# Baixar dependências primeiro (aproveita cache de layer do Docker)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copiar o código
COPY . .

# Build args para metadados de versão
ARG VERSAO=dev
ARG COMMIT=unknown

# Build com todas as otimizações
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build \
    -trimpath \
    -ldflags "-X main.Versao=${VERSAO} -X main.CommitSHA=${COMMIT} -s -w" \
    -o /servidor ./cmd/servidor/

# ========================
# Stage 2: Imagem final mínima
# ========================
# scratch = imagem completamente vazia (menor possível)
# Para ter shell para debug: use FROM alpine:3.19
FROM scratch

# Certificados TLS (necessário para chamadas HTTPS)
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Timezone data (para time.LoadLocation funcionar)
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# O binário
COPY --from=builder /servidor /servidor

# Não executar como root (boa prática de segurança)
# Em scratch, usuário deve ser definido numericamente
USER 65534:65534   # nobody:nobody

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s \
	CMD ["/servidor", "health"]

ENTRYPOINT ["/servidor"]

# Tamanhos típicos:
# golang:1.24-alpine build stage: ~400MB (temporário, descartado)
# Imagem final com scratch:        ~8-12MB
# Imagem final com alpine:          ~15-20MB
```

```bash
docker build \
	--build-arg VERSAO=$(git describe --tags --always) \
	--build-arg COMMIT=$(git rev-parse --short HEAD) \
	--platform linux/amd64 \
	-t meuapp:$(git describe --tags) \
	-t meuapp:latest \
	.

# Build multi-plataforma com buildx
docker buildx build \
	--platform linux/amd64,linux/arm64 \
	-t meuapp:latest \
	--push \
	.
```

---

## 11. Resumo das Flags

| Flag | Efeito |
| --- | --- |
| `-o nome` | Nome do executável de saída |
| `-v` | Lista pacotes compilados |
| `-race` | Habilita race detector (~2x mais lento) |
| `-trimpath` | Remove caminhos locais do binário |
| `-ldflags "-s -w"` | Remove debug info (~30% menor) |
| `-ldflags "-X p.V=v"` | Injeta variável em tempo de build |
| `-gcflags "-m"` | Mostra escape analysis e inlining |
| `-gcflags "all=-N -l"` | Desabilita otimizações (para debugger) |
| `-tags "debug,pg"` | Build tags customizadas |
| `CGO_ENABLED=0` | Binário 100% estático (sem libc) |
| `GOOS=linux` | Sistema operacional alvo |
| `GOARCH=amd64` | Arquitetura alvo |

---

## 12. Por Dentro: Compilação, ISA e Formatos Binários

### GOOS e GOARCH: o que muda no backend do compilador

```
GOOS=linux GOARCH=arm64 go build

Variáveis de ambiente → mudam o backend SSA do compilador gc:

GOARCH=amd64:
  instruções x86-64: MOV, ADD, IMUL, CALL, RET
  registradores: RAX, RBX, ... R15 (16 GPRs de 64 bits)
  ABI Go: argumentos passados em registradores (Go 1.17+)

GOARCH=arm64:
  instruções AArch64: LDR, STR, ADD, MUL, BL, RET
  registradores: X0-X30 (31 GPRs de 64 bits)
  ABI diferente, instrução diferente, tamanho de instrução diferente

GOARCH=wasm:
  instruções WebAssembly bytecode: local.get, i32.add, call
  stack machine (não register machine)
  execução em browser via JS engine ou WASI runtime

O compilador Go contém um backend separado para cada GOARCH:
  cmd/compile/internal/amd64/
  cmd/compile/internal/arm64/
  cmd/compile/internal/wasm/
  ...
```

Conecta com [[Processadores]] — ISA (Instruction Set Architecture) define o contrato entre software e hardware. Cross-compilation significa gerar código para um ISA diferente do host.

### Go compila para código nativo: sem bytecode

```
Comparação de modelos de execução:

Java:
  Fonte.java → javac → Bytecode.class → JVM (interpreta/JIT)
  JVM precisa estar instalada no host

Python:
  script.py → CPython interpreta AST → bytecode.pyc → CPython VM

Go:
  main.go → gc compiler → main.o (código nativo) → linker → ./main
  Execução direta: kernel carrega ELF, CPU executa instruções nativas

Consequências:
  ✅ Sem dependência de runtime externo (JVM, CPython)
  ✅ Startup instantâneo (~5ms vs ~200ms para JVM)
  ✅ Performance previsível (sem JIT pause, sem warmup)
  ❌ Binário maior (inclui runtime Go — ~2MB overhead)
  ❌ Cross-compilation não funciona com CGo
```

Conecta com [[Processadores]] — código nativo é executado diretamente pela CPU. Formato ELF (Executable and Linkable Format) no Linux é o contêiner de código nativo para o kernel carregar.

### Build tags: compilação condicional como `#ifdef`

```go
// arquivo: syscall_linux.go
//go:build linux

package net

// Usa epoll — exclusivo do Linux
func newPollDesc(fd int) { ... epoll_create1 ... }

// arquivo: syscall_darwin.go
//go:build darwin

package net

// Usa kqueue — exclusivo do macOS/BSD
func newPollDesc(fd int) { ... kqueue ... }
```

```
Equivalente em C:
  #ifdef __linux__
    // código epoll
  #elif defined(__APPLE__)
    // código kqueue
  #endif

Go prefere arquivos separados por plataforma a um único arquivo com #ifdefs.
O compilador simplesmente não inclui arquivos que não passam na constraint.

Build matrix:
  GOOS=linux  → inclui syscall_linux.go,   exclui syscall_darwin.go
  GOOS=darwin → inclui syscall_darwin.go,  exclui syscall_linux.go
```

Conecta com [[Processadores]] — código específico por plataforma. A stdlib Go usa build tags extensivamente para adaptar syscalls por OS.

### CGo e linkagem: estático vs dinâmico

```
CGO_ENABLED=0 (padrão em cross-compilation):
  Binário estático — todas as dependências incluídas:
  
  ./servidor  (ELF)
  ├── runtime Go (GC, scheduler, stack mgmt)
  ├── todos os pacotes Go usados
  └── SEM libc (usa syscalls diretas do runtime Go)
  
  Verificar: ldd ./servidor → "not a dynamic executable"
  Rodar em: qualquer Linux com kernel compatível (sem libc necessária)
  
CGO_ENABLED=1 (padrão quando há código C):
  Binário dinâmico — depende de bibliotecas do host:
  
  ./servidor  (ELF)
  ├── runtime Go
  ├── libpthread.so.0  (threads)
  ├── libc.so.6        (C standard library)
  └── libssl.so.1.1    (se usar SSL via C)
  
  Verificar: ldd ./servidor → lista .so files
  Problema: a versão da libc no host pode ser incompatível (glibc hell)
  
Solução para CGo + portabilidade:
  Usar musl libc (estática) ou compilar em Alpine Linux (musl-based)
```

Conecta com [[System Calls]] — `CGO_ENABLED=0` faz o runtime Go emitir syscalls diretamente (sem passar pela libc). `CGO_ENABLED=1` usa glibc que wrappa syscalls com funções C.

### `-ldflags "-s -w"`: DWARF e tabela de símbolos

```
Binário Go sem flags de strip:
┌─────────────────────────────────────────────────┐
│  .text    (código executável)                   │ ~40%
│  .rodata  (constantes, strings)                 │ ~10%
│  .data    (variáveis globais)                   │ ~5%
│  .symtab  (tabela de símbolos — nomes funções)  │ ~15% ← -s remove
│  .debug_* (DWARF — debug info: linhas, vars)    │ ~30% ← -w remove
└─────────────────────────────────────────────────┘

-s remove .symtab:
  → sem nomes de funções no binário
  → pprof e stack traces ficam com endereços, não nomes
  → mas: Go ainda mantém runtime pclntab (para stack traces em panics)

-w remove DWARF (.debug_info, .debug_line, etc.):
  → debugger (dlv, gdb) não consegue mapear endereço→linha de código
  → sem impacto em performance (DWARF é só metadado, não executado)

Resultado típico:
  Antes: 15.2MB   Depois (-s -w): 10.1MB   Upx: 3.8MB
```

Conecta com [[Processadores]] — DWARF (Debugging With Attributed Record Formats) é o formato padrão de debug info nos binários ELF. Symbol table mapeia endereços de memória para nomes de funções.

### Pipeline completo: fonte → binário

```
main.go
   │
   ▼  gc (compilador Go)
   │  Parsing → AST → Typechecking → SSA IR → Otimizações → Asm
   │
   ▼  assembler
main.o  (formato intermediário — code + relocations)
   │
   ▼  linker (cmd/link)
   │  resolve referências entre pacotes
   │  injeta: runtime Go, GC metadata, pclntab
   │  aplica -ldflags (strip symbols, inject vars)
   │
   ▼
./servidor  (ELF no Linux, Mach-O no macOS, PE no Windows)
   │
   ▼  kernel (exec syscall)
   │  carrega ELF em memória (mmap)
   │  salta para _rt0_amd64_linux (entry point do runtime Go)
   │  runtime inicializa: heap, scheduler, goroutine main
   │
   ▼
main.main() executa
```

Conecta com [[Processadores]] — formato ELF é o padrão no Linux. `exec()` syscall carrega o ELF. O kernel usa `mmap()` para mapear as seções do binário em memória virtual.

---

## 13. Conexão com Sistemas Operacionais

- **[[Processadores]]**: `GOOS`/`GOARCH` mudam o backend do compilador, que emite instruções para o ISA alvo (x86-64 ADD/MUL, ARM64 ADD/MUL, WASM i32.add). Go produz código nativo — não bytecode — executado diretamente pela CPU. Sem JVM, sem interpreter.

- **[[Processadores]]** (formatos binários): `go build` produz ELF (Linux), Mach-O (macOS) ou PE (Windows). O kernel carrega o binário com `exec()` + `mmap()`. O entry point é `_rt0_GOARCH_GOOS` que inicializa o runtime Go antes de chamar `main()`.

- **[[Processadores]]** (compilação condicional): Build tags `//go:build linux && amd64` são o equivalente Go de `#ifdef __linux__`. A stdlib usa build tags para adaptar syscalls por plataforma — `epoll` no Linux, `kqueue` no macOS. O compilador simplesmente não compila arquivos com constraints não satisfeitas.

- **[[System Calls]]**: `CGO_ENABLED=0` produz binário que emite syscalls diretamente (sem libc). `CGO_ENABLED=1` linka com libc (glibc ou musl), que wrappa syscalls. Cross-compilation requer `CGO_ENABLED=0` pois o compilador C do host não sabe gerar código para outro OS/arch.

- **[[Processadores]]** (DWARF e símbolos): `-ldflags "-s -w"` remove a tabela de símbolos (`.symtab`) e as informações de debug DWARF (`.debug_*`) do binário. DWARF mapeia endereços de código para linhas de fonte — usado por debuggers (dlv, gdb). Remoção reduz ~30% do tamanho do binário sem impacto em performance de execução.